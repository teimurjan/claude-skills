---
name: code-graph
description: >
  Use tree-sitter to build a structural code graph of a project — extract definitions, references, imports, and module dependencies to quickly understand unfamiliar codebases. Trigger when: user asks to "understand this codebase", "map the architecture", "show me the structure", "what does this project do", "how is this organized", "code graph", "dependency graph", "call graph", or any request to get a high-level overview of a project's code. Also trigger when exploring a new/unfamiliar repo for the first time, or when the user asks about relationships between modules, files, or components. Also trigger when the user asks to refactor a module, file, or component — understanding the module's public surface, internal structure, and all dependents is a prerequisite to safe refactoring.
---

# Code Graph: Project Understanding via Tree-sitter

Use tree-sitter's CLI to parse source code into syntax trees and extract structural information — definitions, references, imports, exports, and module boundaries. This gives you a fast, accurate map of any codebase without running the code.

## Prerequisites

Tree-sitter CLI must be installed. If not available, install it:

```bash
# Check if installed
which tree-sitter

# Install via cargo (preferred)
cargo install --locked tree-sitter-cli

# Or via npm
npm install -g tree-sitter-cli

# Or via brew
brew install tree-sitter
```

Tree-sitter also needs language grammars. Run `tree-sitter dump-languages` to see what's available. If a language is missing, see `references/setup.md`.

## How to Use This Skill

### Step 1: Identify the project's languages

```bash
tree-sitter dump-languages
```

Cross-reference with the file extensions in the project. If grammars are missing, consult `references/setup.md`.

### Step 2: Extract the structural map

Use `tree-sitter tags` on the project's source directories to get all definitions and references:

```bash
tree-sitter tags 'src/**/*.rs'       # Rust
tree-sitter tags 'src/**/*.ts'       # TypeScript
tree-sitter tags 'lib/**/*.py'       # Python
tree-sitter tags '**/*.go'           # Go
```

This outputs: symbol name, kind (definition/reference), type (function, class, method, module, interface), location, and the first line of code.

### Step 3: Extract imports and dependencies

Use `tree-sitter query` with language-specific patterns from `references/query-patterns.md` to extract import statements, module declarations, and export statements.

### Step 4: Build the mental model

With definitions, references, and imports in hand:

1. **Module map** — group definitions by file/directory to understand module boundaries
2. **Dependency graph** — trace imports to see which modules depend on which
3. **Entry points** — identify main functions, exported APIs, route handlers
4. **Core abstractions** — find traits/interfaces/base classes that define the architecture
5. **Hot spots** — files with the most definitions or references are usually central

### Step 5: Deep-dive specific areas

Use `tree-sitter parse` for full syntax trees of specific files when you need more detail:

```bash
tree-sitter parse src/core/engine.rs        # S-expression tree
tree-sitter parse --cst src/core/engine.rs   # Pretty-printed CST
```

Use `tree-sitter query` with custom patterns from `references/query-patterns.md` for targeted extraction.

## Reference Files

- `references/setup.md` — Installing tree-sitter, configuring grammars, troubleshooting parser issues
- `references/query-patterns.md` — Language-specific query patterns for extracting imports, exports, class hierarchies, function signatures, and more
- `references/graph-extraction.md` — Techniques for building dependency graphs, call graphs, and module maps from tree-sitter output

## Refactoring a Module

When asked to refactor a module, file, or component, use the code graph to assess blast radius **before** making changes.

### Step 1: Map the module's public surface

Run `tree-sitter tags` on the target module to identify every exported definition:

```bash
tree-sitter tags 'src/core/engine.rs' 2>/dev/null | grep 'definition'
```

These are the symbols that other modules may depend on. Renaming, removing, or changing the signature of any of these is a breaking change.

### Step 2: Find all dependents

Search the codebase for imports of this module and references to its exports:

```bash
# Find who imports this module (adjust query per language — see references/query-patterns.md)
tree-sitter tags 'src/**/*.rs' 2>/dev/null | grep 'reference'
```

Cross-reference with a grep for the module path to catch re-exports and indirect usage.

### Step 3: Map internal structure

Run `tree-sitter tags` on the module's files to understand internal organization — private functions, helper types, internal state. These are safe to restructure freely.

### Step 4: Assess and proceed

With the public surface, dependents, and internal structure mapped:

- **Safe changes**: renaming private functions, reorganizing internal logic, splitting internal helpers — no dependents affected
- **Requires updates**: renaming public symbols, changing function signatures, moving exports — update all dependents
- **Risky changes**: removing public exports, changing return types, altering trait/interface contracts — verify all call sites

Always present the blast radius to the user before proceeding with breaking changes.

## Decision Flowchart

1. **New codebase, need overview?** → Run `tree-sitter tags` across all source files, then read `references/graph-extraction.md`
2. **Need to understand a specific module?** → Run `tree-sitter tags` + `tree-sitter parse` on that module's files
3. **Need import/dependency info?** → Load `references/query-patterns.md`, use the import queries for the project's language
4. **Need class hierarchy or interface map?** → Load `references/query-patterns.md`, use the inheritance/implementation queries
5. **Refactoring a module?** → Follow the "Refactoring a Module" section above, then load `references/graph-extraction.md` for dependency graph techniques
6. **Parser not working for a language?** → Load `references/setup.md`

## Output Interpretation

`tree-sitter tags` output format (tab-separated):
```
symbol_name    kind    def/ref    (start_row, start_col) - (end_row, end_col)    `first line`    "docstring"
```

Kinds you'll see:
- `definition.function` / `definition.method` — function/method declarations
- `definition.class` — class/struct/enum declarations
- `definition.interface` — trait/interface declarations
- `definition.module` — module declarations
- `reference.call` — function/method calls
- `reference.class` — class/type references
- `reference.implementation` — interface implementations (e.g., `impl Trait for Type`)

## Keywords
code graph, codebase structure, project architecture, module map, dependency graph, tree-sitter, code navigation, understand codebase, explore project, definitions, references, imports, call graph, class hierarchy, refactor, refactoring, restructure, reorganize, extract module, split module, blast radius, breaking changes, dependents
