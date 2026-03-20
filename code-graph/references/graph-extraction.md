# Building Code Graphs from Tree-sitter Output

Techniques for constructing dependency graphs, module maps, and structural overviews from tree-sitter data.

## Strategy: Tags First, Queries Second

Always start with `tree-sitter tags` — it's the fastest way to get definitions and references across a codebase. Use custom queries only when you need information tags doesn't provide (imports, exports, inheritance, type annotations).

## Module Map

A module map shows the project's organizational structure — what each file/directory is responsible for.

### How to build it

1. Run `tree-sitter tags` across all source files
2. Group definitions by file path
3. For each file, list: functions, classes/structs, interfaces/traits, constants
4. Roll up per-file summaries into per-directory summaries
5. Identify patterns: does the project use barrel files? Feature folders? Layer-based organization?

### Command

```bash
# Get all definitions, filter to just defs (not refs)
tree-sitter tags 'src/**/*.rs' 2>/dev/null | grep 'definition'

# For TypeScript projects
tree-sitter tags 'src/**/*.ts' 'src/**/*.tsx' 2>/dev/null | grep 'definition'
```

### What to look for

- **Entry points**: `main`, `index`, `app`, `server` files
- **Public API surface**: exported functions/classes in `lib.rs`, `index.ts`, `__init__.py`
- **Core abstractions**: traits, interfaces, abstract classes — these define the architecture
- **Configuration**: files with many constants or config structures

## Dependency Graph

Shows which modules depend on which — the flow of imports.

### How to build it

1. Extract imports using language-specific queries from `references/query-patterns.md`
2. Resolve import paths to actual files (language-specific rules below)
3. Build an adjacency list: `file A → imports from → file B`
4. Identify: entry points (nothing imports them), leaf modules (they import nothing), hubs (many imports/importers)

### Import resolution by language

**Rust:**
- `use crate::foo::bar` → `src/foo/bar.rs` or `src/foo/bar/mod.rs`
- `use super::baz` → sibling module
- `mod foo;` → `foo.rs` or `foo/mod.rs` in same directory
- External crates: `use serde::Serialize` → dependency in `Cargo.toml`

**TypeScript/JavaScript:**
- `import from './foo'` → `./foo.ts`, `./foo/index.ts`, `./foo.tsx`
- `import from 'react'` → `node_modules/react`
- Path aliases: check `tsconfig.json` `paths` field
- Barrel files: `index.ts` re-exporting from subdirectories

**Python:**
- `from .foo import bar` → `foo.py` or `foo/__init__.py` in same package
- `import foo.bar` → `foo/bar.py` or `foo/bar/__init__.py`
- Absolute imports: relative to project root or `sys.path`

**Go:**
- `import "github.com/org/repo/pkg/foo"` → `pkg/foo/` directory
- `import "./internal/bar"` → relative (rare in Go)
- Standard library: `import "fmt"` → no local file

### Identifying architectural layers

Look at the dependency direction:
- **Top-down**: entry points → business logic → data layer → utilities
- **Circular dependencies**: file A imports B and B imports A — often a design smell
- **Shared kernel**: modules imported by many others — these are your core abstractions

## Call Graph (Lightweight)

Tree-sitter tags gives you `reference.call` entries. These show function/method calls but not which definition they resolve to (that requires semantic analysis). Still useful for:

### What you can determine

1. **Which functions are called most** — count `reference.call` entries per name
2. **Which files make the most calls** — high fan-out files are orchestrators
3. **Unused definitions** — definitions with no matching `reference.call` anywhere (caveat: dynamic calls, reflection, and cross-language boundaries are invisible)

### Command

```bash
# All function calls in the project
tree-sitter tags 'src/**/*.rs' 2>/dev/null | grep 'reference.call'

# All definitions
tree-sitter tags 'src/**/*.rs' 2>/dev/null | grep 'definition.function'
```

## Class/Trait Hierarchy

For OOP or trait-based codebases, understanding the type hierarchy is critical.

### How to extract it

Use the inheritance/implementation queries from `references/query-patterns.md`:

- **Rust**: extract `impl Trait for Type` blocks → build `Type → implements → Trait` edges
- **TypeScript**: extract `class Foo extends Bar implements Baz` → build inheritance + implementation edges
- **Python**: extract `class Foo(Bar, Baz)` → build inheritance edges
- **Go**: extract interface definitions, then find structs with matching method sets

## Presentation Format

When presenting a code graph to the user, use a structured format:

### For module maps
```
src/
  core/         — Core engine: parsing, diffing, output
    engine.rs   — Main diff algorithm (BlazeDiff, diff_images)
    parser.rs   — Image format parsing (PNG, JPEG, WebP)
    output.rs   — Result formatting and serialization
  simd/         — SIMD-accelerated pixel operations
    mod.rs      — Feature detection and dispatch
    x86.rs      — SSE/AVX2 implementations
    arm.rs      — NEON implementations
  cli/          — Command-line interface
    main.rs     — Entry point, arg parsing
    config.rs   — Configuration and defaults
```

### For dependency graphs
```
main.rs → cli/config.rs → core/engine.rs → simd/mod.rs
                        → core/parser.rs
                        → core/output.rs
```

Or use a list format:
```
core/engine.rs
  imports: core/parser.rs, simd/mod.rs, core/output.rs
  exports: BlazeDiff, diff_images, DiffResult
  imported by: cli/main.rs, tests/integration.rs
```

### For type hierarchies
```
Trait: ImageDiffer
  ├── impl: PixelDiffer
  ├── impl: PerceptualDiffer
  └── impl: StructuralDiffer

Trait: OutputFormat
  ├── impl: PngOutput
  └── impl: JsonOutput
```

## Automating the Process

For a full codebase analysis, run these in sequence:

```bash
# 1. Check available parsers
tree-sitter dump-languages 2>/dev/null

# 2. Get all definitions (adjust glob for your project)
tree-sitter tags 'src/**/*.rs' 2>/dev/null | grep 'definition'

# 3. Get all references
tree-sitter tags 'src/**/*.rs' 2>/dev/null | grep 'reference'

# 4. Extract imports (save the query pattern to a temp file first)
echo '(use_declaration argument: (_) @import)' > /tmp/ts-imports.scm
tree-sitter query /tmp/ts-imports.scm 'src/**/*.rs' 2>/dev/null
```

Then synthesize the three outputs into a coherent graph. Prioritize:
1. Entry points and public API
2. Core abstractions (traits/interfaces)
3. Module boundaries and dependency direction
4. Hot spots (most-referenced definitions)
