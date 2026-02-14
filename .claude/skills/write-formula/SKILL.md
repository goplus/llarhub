---
name: write-formula
description: Write an llar build formula (.gox file) for a library package. Use when the user asks to create a formula, add a new package, or write a build recipe.
argument-hint: "[owner/repo] [version]"
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, WebFetch
---

# Write an LLAR Build Formula

Create a build formula for `$ARGUMENTS`.

---

## XGo Language Reference (Differences from Go)

LLAR formulas are written in **XGo** — a superset of Go with syntax sugar. **If you know Go, you know 90% of XGo.** This section covers only the differences.

### Lowercase calling convention

All exported Go functions/methods can be called with **lowercase** first letter. This is the preferred XGo style:

```
strings.index(s, sub)       // calls strings.Index()
strings.trimSpace(s)        // calls strings.TrimSpace()
fmt.println("hello")        // calls fmt.Println()
ctx.outputDir()             // calls ctx.OutputDir__0()
```

**PascalCase also works** (`strings.Index`) but camelCase is idiomatic XGo.

### Command-style calls (no parentheses)

When a function/method is called as a statement, parentheses are optional:

```
echo "hello"                     // fmt.Println("hello")
c.buildType "Release"            // c.BuildType("Release")
c.define "KEY", "VALUE"          // c.Define("KEY", "VALUE")
out.setMetadata "-lz"            // out.SetMetadata("-lz")
deps.require "madler/zlib", "v1" // deps.Require("madler/zlib", "v1")
```

When used in an expression (assignment, condition, etc.), parentheses are still required:

```
err = c.configure()              // expression → needs parens
installDir, err := ctx.outputDir()
```

### Lambda expressions

```
x => x * x                      // single param
(x, y) => x + y                 // multiple params need parens
=> someValue                     // no params
(ctx, proj, out) => { ... }     // block body
```

Equivalent Go: `func(x int) int { return x * x }`

### Auto properties (zero-param methods)

Zero-parameter methods can be called without `()`:

```
output                           // calls this.Output()
lastErr                          // calls this.LastErr()
```

### Error handling operators

```
v := strconv.atoi("10")!         // panic if error
v := strconv.atoi("10")?         // return if error
v := strconv.atoi("10")?:0       // default value if error
```

### `echo` built-in

`echo` is a built-in alias for `fmt.Println`:

```
echo "hello"                     // fmt.Println("hello")
echo "x =", x                   // fmt.Println("x =", x)
```

### String interpolation

```
"age = ${age}"
"path = ${dir}/${name}"
"$${price}"                      // literal $ with $$
```

### String/number methods

```
"hello".toUpper                  // "HELLO"
"Hello World".toLower            // "hello world"
"hello world".capitalize         // "Hello world" (first letter)
"abc".len                        // 3
"12".int!                        // 12 (panic if invalid)
"12".int                         // 12, err (safe)
"3.14".float!                    // 3.14 (float64)
42.string                        // "42"

// Searching
"hello".contains("ell")          // true
"hello".hasPrefix("he")          // true
"hello.txt".hasSuffix(".txt")    // true
"hello".index("ll")              // 2 (-1 if not found)
"hello".lastIndex("l")           // 3

// Splitting & trimming
"a,b,c".split(",")              // ["a", "b", "c"]
"  hello  ".trimSpace            // "hello"
"hello".trimPrefix("he")        // "llo"
"file.txt".trimSuffix(".txt")   // "file"
"##hi##".trim("#")              // "hi"
"  hello  world  ".fields       // ["hello", "world"]

// Replacing
"hello".replaceAll("l", "L")    // "heLLo"
"hello".replace("l", "L", 1)   // "heLlo" (replace first n)
"Ha".repeat(3)                   // "HaHaHa"

// Slice of strings
["a", "b", "c"].join(",")       // "a,b,c"
```

### Slice literals

```
a := [1, 2, 3]                  // []int (inferred)
b := ["hello", "world"]         // []string
c := []                          // []any (empty)
```

Equivalent Go: `[]int{1, 2, 3}`

### Map literals & field access

```
m := {"host": "localhost", "port": 8080}  // map[string]any
echo m.host                               // sugar for m["host"]
m.debug = true                            // sugar for m["debug"] = true
```

### `<-` append operator

```
a := [1, 2, 3]
a <- 4                           // a = append(a, 4)
a <- 5, 6, 7                    // append multiple
a <- other...                    // append another slice
```

### For-in loops

```
for v in arr { ... }             // equivalent to: for _, v := range arr
for i, v in arr { ... }         // equivalent to: for i, v := range arr
for key, val in m { ... }       // iterate map
```

### Range expressions

```
for i in :5 { ... }             // 0, 1, 2, 3, 4
for i in 1:5 { ... }            // 1, 2, 3, 4
for i in 0:10:2 { ... }         // 0, 2, 4, 6, 8
```

### List/Map comprehension

```
[x*x for x in arr]
[x*x for x in arr if x > 3]
[i for i in 1:11]               // [1,2,...,10]
{v: k for k, v in m}            // swap keys and values
```

### Select from collection / existence check

Select picks the **first matching value** from a collection:

```
type Student struct {
    Name  string
    Score int
}
students := [Student{"Alice", 90}, Student{"Bob", 85}]

score := {s.Score for s in students if s.Name == "Alice"}
// score = 90
```

Existence check returns **bool** (true if any match):

```
hasAlice := {for s in students if s.Name == "Alice"}
// hasAlice = true
```

### Overloaded functions

Multiple implementations dispatched by argument types (note: **no commas** between entries):

```
func add = (
    func(a, b int) int { return a + b }
    func(a, b string) string { return a + b }
)

add 1, 2         // → 3         (int version)
add "hi", " go"  // → "hi go"  (string version)
```

Can also reference named functions:

```
func mul = (
    mulInt
    mulFloat
)
```

### Overloaded methods

```
func (foo).mul = (
    (foo).mulInt
    (foo).mulFoo
)
// a.mul(100) → mulInt, a.mul(b) → mulFoo
```

**Go naming convention**: Overloaded functions/methods are compiled to Go with `__N` suffix distinguishing each variant: `outputDir()` → `OutputDir__0()`, `outputDir(dep)` → `OutputDir__1(dep)`.

### Overloaded operators

Define `+`, `-`, `*`, etc. on custom types:

```
type Vec struct { X, Y float64 }

func (a Vec) + (b Vec) Vec {
    return Vec{a.X + b.X, a.Y + b.Y}
}
func -(a Vec) Vec {                   // unary minus
    return Vec{-a.X, -a.Y}
}

a := Vec{1, 2}
b := Vec{3, 4}
c := a + b       // Vec{4, 6}
d := -a           // Vec{-1, -2}
```

Alternative: define via method list (supports mixed types):

```
func (foo).* = (
    (foo).mulInt     // foo * int
    (foo).mulFoo     // foo * foo
    intMulFoo        // int * foo
)
```

### Variadic unpacking

```
more := [4, 5, 6]
echo more...                     // unpack slice as variadic args
```

### Comments

```
// Go-style line comment
# Python-style line comment (also supported)
/* block comment */
```

### Import

Same as Go. **stdlib is NOT auto-imported** — must be explicit:

```
import "strings"
import "fmt"
```

### Optional parameters

```
func connect(host string, port int?, secure bool?) {
    if port == 0 { port = 80 }
    echo host, port, secure
}

connect "example.com", 443, true  // all args
connect "example.com"             // port=0→80, secure=false
```

`T?` means optional — defaults to zero value of that type.

### Keyword arguments

Works with maps or structs as the last parameter:

```
type Config struct {
    Timeout    int
    MaxRetries int
}

func run(cfg *Config?) {
    echo cfg
}

run timeout = 60, maxRetries = 5  // lowercase field names work
run Timeout = 10                  // uppercase also works
run                               // nil (all optional)
```

### Struct type deduction

When a function expects a struct pointer, you can omit the type name:

```
type Config struct {
    Dir   string
    Level int
}

func foo(conf *Config) { ... }

foo {Dir: "/foo/bar", Level: 1}
// equivalent to: foo(&Config{Dir: "/foo/bar", Level: 1})
```

Also works in return statements:

```
func newConfig() *Config {
    return {Dir: "/tmp", Level: 2}
}
```

### Custom iterators

Types can implement `Gop_Enum` to support `for in`:

```
func (p *MyList) Gop_Enum(proc func(key int, val string)) {
    // call proc for each element
}

list := &MyList{}
for i, v in list { echo i, v }
```

### Rational numbers

```
a := 1r << 200   // bigint (suffix r)
b := 4/5r        // bigrat = 4/5
c := b + 1/3r    // rational arithmetic
```

### Bool to number

```
echo int(true)       // 1  (not supported in Go)
echo float64(false)  // 0
```

### C strings (for llgo/C interop)

```
import "c"
c.printf c"Hello, %s\n", c"world"
```

### Domain text literals

Inline DSL content with compile-time parsing:

```
config := json`{"host": "localhost", "port": 8080}`!
pattern := regexp`^[a-z]+\[[0-9]+\]$`!
```

The `!` suffix panics on parse error. `domainTag` maps to `package.New(string)`.

### Tuple types

Like structs but with positional construction:

```
type Point (x, y int)

pt := Point(2, 3)
echo pt.x, pt.y    // 2 3
pt = (100, 200)     // positional assignment
pt2 := Point(y = 5, x = 3)  // keyword construction
```

### Unit literals

Duration suffixes for `time` package:

```
import "time"
time.sleep 1s       // time.Sleep(1 * time.Second)
```

### `type()` built-in

Returns `reflect.Type` of a value:

```
echo type(42)       // int
echo type("hello")  // string
echo type([1, 2])   // []int
```

### Everything else is Go

Variables, types, structs, `if/else`, `for`, `switch`, `make()`, `append()`, goroutines, channels — all standard Go syntax works. XGo also adds `int128`, `uint128`, `bigint`, `bigrat` types.

---

## Classfile Mechanism

The classfile mechanism is how XGo creates DSLs. Each `.gox` file maps to a Go struct, and top-level statements become method calls on `this`.

### How it works

A file `zlib_llar.gox`:

```
id "madler/zlib"
fromVer "1.0.0"

onBuild (ctx, proj, out) => {
    echo "building"
}
```

Is equivalent to this Go code:

```go
type zlib struct {
    formula.ModuleF   // embeds ModuleF (which embeds gsh.App)
}

func (this *zlib) MainEntry() {
    this.Id("madler/zlib")
    this.FromVer("1.0.0")
    this.OnBuild(func(ctx *formula.Context, proj *formula.Project, out *formula.BuildResult) {
        fmt.Println("building")
    })
}
```

Key points:
- The filename prefix (before `_`) becomes the struct name
- Top-level calls like `id "x"` become `this.Id("x")` — methods on the embedded struct
- `ModuleF` embeds [`gsh.App`](https://github.com/qiniu/x/blob/main/gsh/classfile.go), the project class ([classfile](https://github.com/goplus/xgo/blob/main/doc/classfile.md)) behind `.gsh` files — enables shell command execution in XGo via `exec`, `capout`, `output`, `lastErr`, `exitCode`, and direct command syntax (e.g. `mkdir "testgsh"` calls `XGo_Exec("mkdir", "testgsh")`)
- Overloaded methods in Go use `__N` suffix (see Overloaded methods above): `outputDir()` → `OutputDir__0()`, `outputDir(dep)` → `OutputDir__1(dep)`

---

## Two Classfiles

### `_llar.gox` — Build Formula

Struct: [`formula.ModuleF`](https://github.com/goplus/llar/blob/main/formula/classfile.go) (embeds [`gsh.App`](https://github.com/qiniu/x/blob/main/gsh/classfile.go) for shell execution)

Auto-imported packages: `cmake` (`github.com/goplus/llar/x/cmake`), `autotools` (`github.com/goplus/llar/x/autotools`)

You can also `import "strings"` etc. at the top of the file to use Go stdlib.

### `_cmp.gox` — Version Comparator (optional)

Struct: `cmp.CmpApp`

Auto-imported packages: `semver` (`golang.org/x/mod/semver`), `gnu` (`github.com/goplus/llar/x/gnu`)

Default (when no `_cmp.gox` exists): `gnu.Compare` (Debian dpkg-style version comparison).

Provide a `_cmp.gox` when the library uses non-standard version tags (e.g. semver `v1.2.3`, or custom like `VER-2-13-3`):

```
compareVer (a, b) => {
    return semver.Compare(a.Version, b.Version)
}
```

---

## Complete API Reference

### ModuleF (top-level DSL)

| DSL | Go Signature | Purpose |
|-----|-------------|---------|
| `id path` | `Id(path string)` | Module path, format `"owner/repo"` |
| `fromVer ver` | `FromVer(ver string)` | Minimum version this formula serves |
| `matrix m` | `Matrix(m Matrix)` | Build matrix variations |
| `onRequire (proj, deps) => {}` | `OnRequire(func(*Project, *ModuleDeps))` | Declare dependencies |
| `onBuild (ctx, proj, out) => {}` | `OnBuild(func(*Context, *Project, *BuildResult))` | Build steps |

### Project

| DSL | Go Signature | Purpose |
|-----|-------------|---------|
| `proj.readFile(path)` | `ReadFile(path string) ([]byte, error)` | Read file from source. In `onRequire`: reads from real source repo at tagged version. In `onBuild`: reads from formula store. |
| `proj.Deps` | `Deps []module.Version` | Direct dependencies (resolved) |

### Context

| DSL | Go Signature | Purpose |
|-----|-------------|---------|
| `ctx.outputDir()` | `OutputDir__0() (string, error)` | Current module's install directory |
| `ctx.outputDir(dep)` | `OutputDir__1(mod module.Version) (string, error)` | Dependency's install directory |
| `ctx.currentMatrix()` | `CurrentMatrix() string` | Active matrix string (e.g. `"amd64-linux"`) |
| `ctx.buildResult(dep)` | `BuildResult(mod module.Version) (BuildResult, bool)` | Get a dependency's build result |
| `ctx.SourceDir` | `SourceDir string` | Source code directory path |

### BuildResult

| DSL | Go Signature | Purpose |
|-----|-------------|---------|
| `out.addErr err` | `AddErr(err error)` | Record a build error |
| `out.setMetadata s` | `SetMetadata(metadata string)` | Set output metadata (e.g. pkg-config flags) |
| `out.metadata()` | `Metadata() string` | Read metadata |
| `out.errs()` | `Errs() []error` | Read errors |

### ModuleDeps

| DSL | Go Signature | Purpose |
|-----|-------------|---------|
| `deps.require path, ver` | `Require(path, ver string)` | Declare a dependency |
| `deps.deps()` | `Deps() []module.Version` | Read declared deps |

### gsh.App (shell, inherited by ModuleF)

| DSL | Go Signature | Purpose |
|-----|-------------|---------|
| `exec name, args...` | `Exec__2(name string, args ...string)` | Run a command |
| `exec cmdline` | `Exec__1(cmdline string)` | Run shell command line (supports `$VAR`) |
| `capout => { ... }` | `Capout(func()) (string, error)` | Capture stdout of block |
| `output` | `Output() string` | Get last captured output string |
| `lastErr` | `LastErr() error` | Get last command error |
| `exitCode()` | `ExitCode() int` | Get last exit code |

### cmake

| DSL | Go Signature | Purpose |
|-----|-------------|---------|
| `cmake.new(src, build, install)` | `New(src, build, install string) *CMake` | Create cmake builder |
| `c.buildType name` | `BuildType(name string)` | Set CMAKE_BUILD_TYPE |
| `c.define key, val` | `Define(key, val string)` | `-DKEY:STRING=val` |
| `c.defineBool key, val` | `DefineBool(key string, val bool)` | `-DKEY:BOOL=ON/OFF` |
| `c.use root` | `Use(root string)` | Inject dependency paths |
| `c.configure(args...)` | `Configure(args ...string) error` | Run cmake configure |
| `c.build(args...)` | `Build(args ...string) error` | Run cmake --build |
| `c.install(args...)` | `Install(args ...string) error` | Run cmake --install |

### autotools

| DSL | Go Signature | Purpose |
|-----|-------------|---------|
| `autotools.new(src, build, install)` | `New(src, build, install string) *AutoTools` | Create autotools builder |
| `a.use root` | `Use(root string)` | Inject dependency paths |
| `a.configure(args...)` | `Configure(args ...string) error` | Run ./configure |
| `a.build(args...)` | `Build(args ...string) error` | Run make |
| `a.install(args...)` | `Install(args ...string) error` | Run make install |

Both `.use()` set: `CMAKE_PREFIX_PATH`, `PKG_CONFIG_PATH`, `CPPFLAGS -I.../include`, `LDFLAGS -L.../lib`.

---

## XGo Pitfalls in Formula Context

1. **Map literal at statement level** — bare `{"k":"v"}` at statement level is ambiguous with lambdas. In assignments it works fine (`m := {"k":"v"}`), but when unsure, use `make`:
   ```
   m := make(map[string]string)
   m["k"] = "v"
   ```
2. **Missing import** — stdlib is NOT auto-imported. Add `import "strings"` at file top
3. **Functions become methods** — `func xxx() {}` in `.gox` is translated to `func (this *StructName) xxx()` — a method on the classfile struct, not a standalone function
4. **Old cmake compat** — add `c.define "CMAKE_POLICY_VERSION_MINIMUM", "3.5"` for projects with old cmake_minimum_required
5. **cmake CPPFLAGS** — cmake custom commands may ignore CPPFLAGS; switch to autotools if dep include paths conflict
6. **pkg-config needs `.use`** — call `.use installDir` before `exec "pkg-config"` to set PKG_CONFIG_PATH

---

## File Layout

```
{owner}/{repo}/
  versions.json                  # module metadata + static deps fallback
  {name}_cmp.gox                 # optional: custom version comparator
  {from_version}/{name}_llar.gox # the build formula
```

`{name}` is the library name (e.g. `zlib`, `freetype`, `libpng`).

### versions.json

```json
{
  "path": "owner/repo",
  "deps": {}
}
```

- `deps` is a **fallback** for when `onRequire` is absent or fails
- For static deps: `"deps": {"1.0.0": [{"path": "dep/path", "version": "v1.0.0"}]}`

---

## Workflow

### Step 1: Investigate the target library

1. Identify the build system: cmake (`CMakeLists.txt`), autotools (`configure`), meson (`meson.build`), etc.
2. Find dependencies and how they are declared
3. Check for lock/wrap files with pinned versions (e.g. meson `subprojects/*.wrap`)
4. Note the git tag format (e.g. `v1.3.1`, `VER-2-13-3`)

### Step 2: Write versions.json + optional _cmp.gox + _llar.gox

### Step 3: Test with `llar make`

---

## Dynamic Dependency Extraction (onRequire)

`proj.readFile(path)` in `onRequire` reads from the **real source repo** at the tagged version. Use this to discover dependencies from the package's own build tool.

**Pattern: discover then lookup** — do NOT hardcode which deps to read. Discover them from the build config first, then look up versions from lock/wrap files.

---

## Build Tool Selection

| | cmake | autotools |
|---|---|---|
| When | Has `CMakeLists.txt` | Has `configure` script |
| Caveat | Custom commands may **ignore** CPPFLAGS, causing system lib header conflicts | Correctly uses CPPFLAGS/LDFLAGS |

If cmake dep injection causes include path conflicts, switch to autotools.

Tip: for old cmake projects, add `c.define "CMAKE_POLICY_VERSION_MINIMUM", "3.5"`.

---

## Metadata & pkg-config

```
c.use installDir    // set PKG_CONFIG_PATH first!
capout => {
    exec "pkg-config", "--libs", "libname"
}
out.setMetadata output
```

---

## Complete Examples

### Simple: zlib (cmake, no deps)

```
id "madler/zlib"
fromVer "1.0.0"

onBuild (ctx, proj, out) => {
    installDir, err := ctx.outputDir()
    if err != nil { out.addErr err; return }

    c := cmake.new(ctx.SourceDir, ctx.SourceDir+"/_build", installDir)
    c.buildType "Release"
    c.define "CMAKE_POLICY_VERSION_MINIMUM", "3.5"

    err = c.configure()
    if err != nil { out.addErr err; return }
    err = c.build()
    if err != nil { out.addErr err; return }
    err = c.install()
    if err != nil { out.addErr err; return }

    out.setMetadata "-lz"
}
```

### Medium: libpng (autotools, static dep on zlib)

```
id "pnggroup/libpng"
fromVer "1.0.0"

onRequire (proj, deps) => {
    deps.require "madler/zlib", "v1.2.11"
}

onBuild (ctx, proj, out) => {
    installDir, err := ctx.outputDir()
    if err != nil { out.addErr err; return }

    a := autotools.new(ctx.SourceDir, ctx.SourceDir+"/_build", installDir)
    for _, dep := range proj.Deps {
        depDir, err := ctx.outputDir(dep)
        if err != nil { out.addErr err; return }
        a.use depDir
    }

    err = a.configure()
    if err != nil { out.addErr err; return }
    err = a.build()
    if err != nil { out.addErr err; return }
    err = a.install()
    if err != nil { out.addErr err; return }

    out.setMetadata "-lpng"
}
```

### Advanced: freetype (dynamic deps from meson wraps, pkg-config, custom version comparator)

**freetype_cmp.gox** (next to versions.json):
```
compareVer (a, b) => {
    return semver.Compare(a.Version, b.Version)
}
```

**1.0.0/freetype_llar.gox**:
```
import "strings"

id "freetype/freetype"
fromVer "1.0.0"

onRequire (proj, deps) => {
    depMap := make(map[string]string)
    depMap["zlib"] = "madler/zlib"
    depMap["libpng"] = "pnggroup/libpng"

    data, err := proj.readFile("meson.build")
    if err != nil { return }
    content := string(data)

    needle := "dependency('"
    for {
        i := strings.index(content, needle)
        if i < 0 { break }
        content = content[i+len(needle):]
        j := strings.index(content, "'")
        if j < 0 { break }
        name := content[:j]
        content = content[j+1:]

        modPath, ok := depMap[name]
        if !ok { continue }

        wrapData, wrapErr := proj.readFile("subprojects/" + name + ".wrap")
        if wrapErr != nil { continue }
        wrapContent := string(wrapData)

        dirKey := "directory = "
        di := strings.index(wrapContent, dirKey)
        if di < 0 { continue }
        rest := wrapContent[di+len(dirKey):]
        nl := strings.index(rest, "\n")
        if nl >= 0 { rest = rest[:nl] }
        rest = strings.trimSpace(rest)
        dash := strings.lastIndex(rest, "-")
        if dash < 0 { continue }
        ver := rest[dash+1:]
        deps.require modPath, "v"+ver
    }
}

onBuild (ctx, proj, out) => {
    installDir, err := ctx.outputDir()
    if err != nil { out.addErr err; return }

    c := cmake.new(ctx.SourceDir, ctx.SourceDir+"/_build", installDir)
    c.buildType "Release"
    c.define "CMAKE_POLICY_VERSION_MINIMUM", "3.5"
    c.defineBool "FT_DISABLE_HARFBUZZ", true
    c.defineBool "FT_DISABLE_BROTLI", true
    c.defineBool "FT_DISABLE_BZIP2", true
    c.defineBool "FT_REQUIRE_ZLIB", true
    c.defineBool "FT_REQUIRE_PNG", true

    for _, dep := range proj.Deps {
        depDir, err := ctx.outputDir(dep)
        if err != nil { out.addErr err; return }
        c.use depDir
    }

    err = c.configure()
    if err != nil { out.addErr err; return }
    err = c.build()
    if err != nil { out.addErr err; return }
    err = c.install()
    if err != nil { out.addErr err; return }

    c.use installDir
    capout => {
        exec "pkg-config", "--libs", "freetype2"
    }
    out.setMetadata output
}
```
