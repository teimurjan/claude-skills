# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of Claude Code custom skills (`~/.claude/skills/`). Each skill is a self-contained directory with a `SKILL.md` (frontmatter + instructions) and optional reference files or tooling.

## Repository Structure

```
<skill-name>/
  SKILL.md              # Skill definition (frontmatter: name, description, trigger conditions)
  references/           # Deep-dive docs the skill loads on demand
  *.js, package.json    # Tooling (if the skill ships a CLI tool)
```

Current skills:
- **browser-testing** — ABP (Agent Browser Protocol) CLI wrapper in Deno (`browser.js`, 750 LOC). Has npm deps for Readability/Markdown extraction (`@mozilla/readability`, `jsdom`, `turndown`).
- **code-graph** — Tree-sitter-based codebase analysis. Pure markdown, no runtime code.
- **perf-engineering** — CPU/memory optimization guidance. Pure markdown, no runtime code.
- **review-pr** — Fetches and addresses GitHub PR review comments. Pure markdown, no runtime code.

## Skill Anatomy

A `SKILL.md` has YAML frontmatter (`name`, `description`) followed by markdown instructions. The `description` field controls when the skill triggers — it doubles as the matching criteria shown to Claude Code. Reference files under `references/` are loaded lazily by the skill's instructions (not auto-loaded).

## Conventions

- Skills are referenced by directory name (e.g. `browser-testing`, `code-graph`)
- Commit messages use conventional commits format (`feat:`, `chore:`, `fix:`)
- No build step, no tests, no linter — skills are authored markdown + lightweight scripts
