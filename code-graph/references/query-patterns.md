# Tree-sitter Query Patterns by Language

Use these patterns with `tree-sitter query <pattern-file> <source-files>` to extract structural information. Save the pattern to a `.scm` file, then run the query.

## Query Syntax Cheat Sheet

```scheme
; Match a node type
(function_definition)

; Capture a node with @name
(function_definition name: (identifier) @func-name)

; Wildcards — (_) matches any named node
(call_expression function: (_) @callee)

; Alternations — match one of several
[(import_statement) (import_from_statement)] @import

; Anchors — . means "adjacent"
(comment) . (function_definition) @documented-fn

; Predicates
(#eq? @name "main")              ; exact match
(#match? @name "^test_")         ; regex match
(#not-eq? @name "constructor")   ; negation
```

Run a query:
```bash
tree-sitter query imports.scm 'src/**/*.py'
```

---

## Rust

### Imports (use statements)
```scheme
; references/queries/rust-imports.scm
(use_declaration
  argument: (_) @import-path)
```

### Function definitions with parameters
```scheme
(function_item
  name: (identifier) @func-name
  parameters: (parameters) @params
  return_type: (_)? @return-type)
```

### Struct/enum definitions
```scheme
[
  (struct_item name: (type_identifier) @struct-name)
  (enum_item name: (type_identifier) @enum-name)
] @type-def
```

### Trait definitions and implementations
```scheme
; Trait definitions
(trait_item
  name: (type_identifier) @trait-name)

; Impl blocks
(impl_item
  trait: (type_identifier)? @trait
  type: (_) @impl-type)
```

### Module declarations
```scheme
[
  (mod_item name: (identifier) @mod-name)
] @module
```

### Macro definitions
```scheme
(macro_definition
  name: (identifier) @macro-name)
```

---

## TypeScript / JavaScript

### Imports
```scheme
; ES imports
(import_statement
  source: (string) @import-source)

; Require calls
(call_expression
  function: (identifier) @_func
  arguments: (arguments (string) @require-source)
  (#eq? @_func "require"))
```

### Named imports (what's being imported)
```scheme
(import_statement
  (import_clause
    (named_imports
      (import_specifier
        name: (identifier) @imported-name)))
  source: (string) @from-module)
```

### Default and namespace imports
```scheme
; Default import
(import_statement
  (import_clause
    (identifier) @default-import)
  source: (string) @from-module)

; Namespace import
(import_statement
  (import_clause
    (namespace_import (identifier) @namespace-import))
  source: (string) @from-module)
```

### Exports
```scheme
; Named exports
(export_statement
  declaration: (_) @exported-decl)

; Default export
(export_statement
  value: (_) @default-export)

; Re-exports
(export_statement
  source: (string) @re-export-source)
```

### Function/arrow function definitions
```scheme
; Function declarations
(function_declaration
  name: (identifier) @func-name)

; Arrow functions assigned to const/let
(lexical_declaration
  (variable_declarator
    name: (identifier) @func-name
    value: (arrow_function)))
```

### Class definitions with methods
```scheme
(class_declaration
  name: (type_identifier) @class-name
  body: (class_body
    (method_definition
      name: (property_identifier) @method-name)))
```

### Interface definitions (TypeScript)
```scheme
(interface_declaration
  name: (type_identifier) @interface-name)
```

### Type aliases (TypeScript)
```scheme
(type_alias_declaration
  name: (type_identifier) @type-name
  value: (_) @type-value)
```

---

## Python

### Imports
```scheme
; import foo
(import_statement
  name: (dotted_name) @import-name)

; from foo import bar
(import_from_statement
  module_name: (dotted_name) @module
  name: (dotted_name) @imported-name)

; from foo import *
(import_from_statement
  module_name: (dotted_name) @module
  (wildcard_import) @star-import)
```

### Function definitions
```scheme
(function_definition
  name: (identifier) @func-name
  parameters: (parameters) @params)
```

### Class definitions with inheritance
```scheme
(class_definition
  name: (identifier) @class-name
  superclasses: (argument_list
    (_) @base-class)?)
```

### Decorated definitions
```scheme
(decorated_definition
  (decorator (identifier) @decorator-name)
  definition: [
    (function_definition name: (identifier) @func-name)
    (class_definition name: (identifier) @class-name)
  ])
```

---

## Go

### Imports
```scheme
; Single import
(import_declaration
  (import_spec
    path: (interpreted_string_literal) @import-path))

; Aliased import
(import_declaration
  (import_spec
    name: (package_identifier) @alias
    path: (interpreted_string_literal) @import-path))
```

### Function definitions
```scheme
(function_declaration
  name: (identifier) @func-name
  parameters: (parameter_list) @params)
```

### Method definitions (with receiver)
```scheme
(method_declaration
  receiver: (parameter_list
    (parameter_declaration
      type: (_) @receiver-type))
  name: (field_identifier) @method-name)
```

### Struct definitions
```scheme
(type_declaration
  (type_spec
    name: (type_identifier) @struct-name
    type: (struct_type)))
```

### Interface definitions
```scheme
(type_declaration
  (type_spec
    name: (type_identifier) @interface-name
    type: (interface_type)))
```

---

## C / C++

### Include directives
```scheme
(preproc_include
  path: [
    (string_literal) @include-path
    (system_lib_string) @system-include
  ])
```

### Function definitions
```scheme
(function_definition
  declarator: (function_declarator
    declarator: (_) @func-name
    parameters: (parameter_list) @params))
```

### Struct/class definitions (C++)
```scheme
; Struct
(struct_specifier
  name: (type_identifier) @struct-name)

; Class (C++)
(class_specifier
  name: (type_identifier) @class-name)
```

---

## Tips for Custom Queries

### Finding the right node types

Use `tree-sitter parse` to see the syntax tree for a sample file, then write queries targeting those node types:

```bash
tree-sitter parse sample.rs 2>/dev/null | head -50
```

### Combining queries

Put multiple patterns in one `.scm` file. Each pattern is matched independently:

```scheme
; Extract both imports and function defs in one pass
(use_declaration argument: (_) @import)
(function_item name: (identifier) @func-def)
```

### Performance

- Prefer targeted glob patterns over `**/*`
- Combine related queries into a single `.scm` file to avoid re-parsing
- Use `tree-sitter tags` when its built-in captures are sufficient — it's faster than custom queries for definitions/references
