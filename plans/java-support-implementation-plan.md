# Java support implementation plan

## Conclusion

Java support is possible in the current architecture.

The explorer is not using a real Java parser today; it uses lightweight line-oriented parsers in `src/explore.zig` for each supported language. That means Java can be added without changing the indexing model, MCP surface, snapshot format, or search engine, but "full support" in this codebase means:

- Java files are detected and labeled correctly
- Java symbols appear in outlines/tree output
- Java imports contribute to the dependency graph
- compact line extraction skips Java comments
- telemetry and UI report Java correctly
- tests cover the supported Java syntax surface

If the goal is full semantic Java support, type resolution, or grammar-complete parsing, that would require a different parser architecture. For this repository's current design, the right implementation is a dedicated Java parser path alongside the existing Zig/Python/TS/Rust/PHP parsers.

## Current state

### What already exists

- `src/explore.zig`
  - `Language` enum
  - `detectLanguage`
  - parser dispatch in `Explorer.indexFile`
  - language-specific parsers such as `parsePythonLine`, `parseTsLine`, `parseRustLine`, `parsePhpLine`
  - `isCommentOrBlank` for compact extraction
- `src/tests.zig`
  - language detection tests
  - parser tests
  - comment filtering tests
- `src/telemetry.zig`
  - hard-coded language name list for telemetry bitmasks
- `src/style.zig`
  - language color mapping for tree output

### Important gaps found during review

1. Java is not present in `src/explore.zig` language detection or parser dispatch.
2. `README.md` and `docs/architecture.md` do not consistently match actual parser coverage.
3. `src/telemetry.zig` already has a language-order problem: the names array does not include `php`, even though `Language` does. Java work should fix that at the same time because telemetry indexing depends on enum order.

## Recommended implementation

Implement Java as a first-class explorer language in the same style as PHP and Rust:

1. extend the language enum and detection
2. add Java-specific parsing state
3. add a `parseJavaLine` parser
4. normalize Java imports/packages into dependency paths
5. teach compact extraction that Java comments should be skipped
6. update telemetry/style/docs
7. add focused tests for supported Java constructs

## File-by-file plan

### 1) `/home/runner/work/codedb-g/codedb-g/src/explore.zig`

#### A. Add Java to `Language`

Update the enum near the existing language declarations:

```zig
pub const Language = enum(u8) {
    zig,
    c,
    cpp,
    python,
    javascript,
    typescript,
    rust,
    go_lang,
    php,
    java,
    markdown,
    json,
    yaml,
    unknown,
};
```

Keep `java` before `markdown`/`json`/`yaml` so it stays grouped with source languages. If enum order changes, telemetry must be updated in lockstep.

#### B. Detect `.java` files

Extend `detectLanguage`:

```zig
if (std.mem.endsWith(u8, path, ".java")) return .java;
```

This is enough for file tree output, snapshots, MCP read-path language labeling, and compact extraction routing.

#### C. Add parser dispatch

In the main parse loop, add a Java branch beside the existing Rust/PHP branches:

```zig
} else if (outline.language == .java) {
    try self.parseJavaLine(trimmed, line_num, &outline, &java_state);
}
```

Add a `JavaParseState` local alongside the existing `PhpParseState`.

#### D. Add `JavaParseState`

Java needs parser state similar to PHP because class/method detection depends on nesting depth and block comments:

```zig
const JavaParseState = struct {
    in_block_comment: bool = false,
    brace_depth: i32 = 0,
    type_stack: std.ArrayListUnmanaged(i32) = .{},

    fn deinit(self: *JavaParseState, allocator: std.mem.Allocator) void {
        self.type_stack.deinit(allocator);
    }
};
```

Use the stack to track whether the parser is currently inside a type body, so methods can be marked as `.method` and top-level declarations can be distinguished from nested scopes.

#### E. Add `parseJavaLine`

Add a dedicated parser function near the other parser implementations. It should support at least:

- `package foo.bar;`
- `import foo.bar.Baz;`
- `import static foo.bar.Baz.qux;`
- `class`
- `interface`
- `enum`
- `record`
- `@interface`
- constructors
- methods
- fields/constants
- nested types

Recommended behavior:

- `class X`, `record X` => `.class_def`
- `interface X`, `@interface X` => `.interface_def`
- `enum X` => `.enum_def`
- methods/constructors inside a type => `.method`
- top-level `static final` constants => `.constant`
- imports => `.import`
- package declaration should update import normalization context, not create a symbol

Suggested skeleton:

```zig
fn parseJavaLine(
    self: *Explorer,
    raw_line: []const u8,
    line_num: u32,
    outline: *FileOutline,
    state: *JavaParseState,
) !void {
    const a = self.allocator;
    var line = raw_line;

    if (line.len == 0) return;

    if (state.in_block_comment) {
        if (std.mem.indexOf(u8, line, "*/")) |end| {
            state.in_block_comment = false;
            line = std.mem.trim(u8, line[end + 2 ..], " \t");
            if (line.len == 0) return;
        } else return;
    }

    if (std.mem.startsWith(u8, line, "/*")) {
        if (std.mem.indexOf(u8, line, "*/") == null) state.in_block_comment = true;
        return;
    }
    if (std.mem.startsWith(u8, line, "//")) return;

    if (std.mem.startsWith(u8, line, "package ")) {
        // store package name in parser state if needed for path normalization
    } else if (std.mem.startsWith(u8, line, "import ")) {
        try self.parseJavaImport(a, line, line_num, outline);
        return;
    } else if (self.javaMatchType(line)) |match| {
        // append class/interface/enum symbol and push type depth
    } else if (self.javaMatchMethod(line, state)) |name| {
        // append method symbol
    } else if (self.javaMatchConstant(line, state)) |name| {
        // append constant symbol
    }

    // update brace depth while ignoring braces inside strings
}
```

#### F. Add Java helpers

Add small helpers rather than embedding all logic in `parseJavaLine`:

- `javaMatchType`
- `javaMatchMethod`
- `javaMatchConstant`
- `parseJavaImport`
- `normalizeJavaImportPath`
- optional: `stripJavaAnnotations`
- optional: `stripJavaGenerics`

Recommended import normalization:

```zig
fn normalizeJavaImportPath(allocator: std.mem.Allocator, import_path: []const u8) ![]const u8 {
    var buf: std.ArrayList(u8) = .{};
    defer buf.deinit(allocator);

    for (import_path) |ch| {
        try buf.append(allocator, if (ch == '.') '/' else ch);
    }

    if (!std.mem.endsWith(u8, buf.items, ".java")) {
        try buf.appendSlice(allocator, ".java");
    }

    return buf.toOwnedSlice(allocator);
}
```

That keeps Java dependency edges consistent with the existing Python/PHP normalization style.

#### G. Extend `isCommentOrBlank`

Add `.java` to the C/JS-style comment branch:

```zig
.javascript, .typescript, .c, .cpp, .java =>
    std.mem.startsWith(u8, trimmed, "//") or
    std.mem.startsWith(u8, trimmed, "/*") or
    std.mem.startsWith(u8, trimmed, "*"),
```

This is required so compact line extraction works for Java in MCP responses.

### 2) `/home/runner/work/codedb-g/codedb-g/src/tests.zig`

Add targeted tests instead of one large fixture.

#### A. Extend language detection coverage

Update `test "detectLanguage: all supported extensions"`:

```zig
try testing.expect(explore.detectLanguage("Main.java") == .java);
```

#### B. Extend compact comment coverage

Add Java comment tests near the existing Rust/Go/C++ tests:

```zig
test "isCommentOrBlank: java comments" {
    try testing.expect(isCommentOrBlank("  // java line comment", .java));
    try testing.expect(isCommentOrBlank("  /* java block comment */", .java));
    try testing.expect(isCommentOrBlank("  * java continuation", .java));
    try testing.expect(!isCommentOrBlank("  public class Main {", .java));
}
```

#### C. Add parser tests

At minimum add:

1. `test "explorer: java parser detects class, import, and method"`
2. `test "explorer: java parser detects interface and enum"`
3. `test "explorer: java parser normalizes imports into dep_graph paths"`
4. `test "explorer: java parser handles record and annotation interface"`
5. `test "explorer: java parser ignores comments and annotations"`

Useful fixture:

```zig
try explorer.indexFile("src/com/acme/Main.java",
    \\package com.acme;
    \\
    \\import java.util.List;
    \\import com.acme.service.UserService;
    \\
    \\public class Main {
    \\    private static final int PORT = 8080;
    \\
    \\    public Main() {}
    \\
    \\    public void run() {}
    \\}
);
```

Expected assertions:

- outline language is `.java`
- symbols contain import/class/method/constant entries
- `com.acme.service.UserService` becomes `com/acme/service/UserService.java`

#### D. Add regression tests for Java-specific syntax

Add small issue-style tests for:

- annotations before a class or method
- generic signatures
- nested classes
- constructor detection
- `record User(String id) {}`
- `sealed interface`
- `public static void main(String[] args)`
- `import static foo.Bar.baz;`

These do not need full semantic parsing; they only need to verify that outline extraction stays stable.

### 3) `/home/runner/work/codedb-g/codedb-g/src/telemetry.zig`

Update `writeLanguages` so the hard-coded names match `Language` enum order exactly.

Recommended result:

```zig
const names = [_][]const u8{
    "zig",
    "c",
    "cpp",
    "python",
    "javascript",
    "typescript",
    "rust",
    "go_lang",
    "php",
    "java",
    "markdown",
    "json",
    "yaml",
    "unknown",
};
```

This is required even if Java is added cleanly, because the current list is already out of sync with the enum.

### 4) `/home/runner/work/codedb-g/codedb-g/src/style.zig`

Add a Java color mapping so tree output is not shown as the default dim style:

```zig
if (std.mem.eql(u8, lang, "java")) return self.orange;
```

Any existing source-language color is fine; the important part is that Java is treated as a first-class code language in CLI output.

### 5) `/home/runner/work/codedb-g/codedb-g/README.md`

Update the language support claims so they match implementation reality.

Recommended changes:

- add Java to the support list once parser/tests land
- do not claim Java until tests pass
- consider tightening the current wording, because the README currently suggests support breadth that is wider than the parser implementation shown in `src/explore.zig`

### 6) `/home/runner/work/codedb-g/codedb-g/docs/architecture.md`

Update the Explorer description to include Java in the parser list after implementation.

## Parsing scope to target in v1

To keep the change bounded, v1 Java support should cover:

- package declarations
- normal imports
- static imports
- classes
- interfaces
- annotation interfaces
- enums
- records
- constructors
- methods
- `static final` constants
- nested types
- single-line and block comments

It should explicitly not try to implement:

- type resolution
- overload resolution
- inheritance graphs
- annotation semantics
- full generic parsing
- compiler-accurate parsing for malformed Java

## Validation checklist

After implementation:

1. run `zig build test`
2. verify Java files appear in `codedb tree` with `language = "java"`
3. verify Java comment lines are removed by compact extraction
4. verify Java imports create dependency edges
5. verify telemetry language output includes both `php` and `java` in correct positions

## Suggested implementation order

1. `src/explore.zig`: enum + detection + comment filtering
2. `src/tests.zig`: add failing Java detection/comment/parser tests
3. `src/explore.zig`: add `JavaParseState`, helpers, and `parseJavaLine`
4. `src/telemetry.zig`: fix enum/name alignment and add Java
5. `src/style.zig`: add Java color
6. `README.md` and `docs/architecture.md`: update docs after tests pass

## Risk notes

- The biggest risk is false positives from line-based parsing when annotations, generics, or multi-line signatures are involved.
- The safest path is to follow the PHP parser pattern: use explicit parser state and focused helpers instead of large regex-like logic.
- Because telemetry uses enum indices, changing `Language` without updating `src/telemetry.zig` will silently corrupt reported language names.
- The current README already appears ahead of the implementation in a few places, so docs should only be updated after the parser and tests are in place.
