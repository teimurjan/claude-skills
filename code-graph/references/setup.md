# Tree-sitter Setup & Configuration

## Installation

```bash
# Via cargo (recommended — always latest)
cargo install --locked tree-sitter-cli

# Via npm
npm install -g tree-sitter-cli

# Via brew
brew install tree-sitter

# Verify
tree-sitter --version
```

## Configuration

Tree-sitter uses a config file at `~/.config/tree-sitter/config.json`. Generate the default:

```bash
tree-sitter init-config
```

The config defines `parser-directories` — paths where tree-sitter looks for grammar repos. Any directory whose name starts with `tree-sitter-` inside these paths is auto-discovered.

Default config example:
```json
{
  "parser-directories": [
    "/Users/you/github",
    "/Users/you/src"
  ]
}
```

## Installing Language Grammars

Tree-sitter needs grammar repos cloned into a parser directory.

### Quick setup for common languages

```bash
# Pick a parser directory (must be in your config's parser-directories)
PARSER_DIR="$HOME/.local/share/tree-sitter/parsers"
mkdir -p "$PARSER_DIR"

# Add to config if not already there
# Edit ~/.config/tree-sitter/config.json and add PARSER_DIR to parser-directories

# Clone grammars you need
cd "$PARSER_DIR"
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-rust
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-python
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-javascript
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-typescript
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-go
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-c
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-cpp
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-java
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-ruby
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-html
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-css
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-json
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-bash
git clone --depth 1 https://github.com/tree-sitter/tree-sitter-swift
```

### TypeScript grammar note

The TypeScript grammar lives in a monorepo with two sub-parsers:
- `tree-sitter-typescript/typescript` — for `.ts` files
- `tree-sitter-typescript/tsx` — for `.tsx` files

Both are needed for full TypeScript/React support.

### Verify installed grammars

```bash
tree-sitter dump-languages
```

This lists all discovered parsers with their paths and file types.

## Troubleshooting

### "No language found for file"

The grammar for this file extension isn't installed or not in a parser directory. Check:
1. `tree-sitter dump-languages` — is the language listed?
2. Is the grammar repo in a directory listed in `parser-directories`?
3. Does the grammar repo name start with `tree-sitter-`?

### "Error compiling parser"

Tree-sitter compiles parsers to native code on first use. You need a C compiler:
- macOS: `xcode-select --install`
- Linux: `apt install build-essential` or equivalent
- The compiled `.so`/`.dylib` is cached for subsequent runs

### "No tags.scm found"

Not all grammars ship with `queries/tags.scm`. You can:
1. Check the grammar repo for the file
2. Write a minimal `tags.scm` — see `references/query-patterns.md` for patterns
3. Fall back to `tree-sitter parse` and manual inspection

### Slow parsing on large projects

- Use glob patterns to target specific directories: `tree-sitter tags 'src/**/*.rs'` instead of `'**/*.rs'`
- Exclude generated code, vendor dirs, node_modules, build artifacts
- Parse in batches by directory if the project is very large
