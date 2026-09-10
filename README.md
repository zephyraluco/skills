# Skills Repository

A collection of agent skills that can be installed with `bunx skills add`.

## Installation

To install a skill from this repository, run:

```bash
bunx skills add your-username/skills --skill skill-name
```

## Available Skills

- `coding-standards` - Baseline cross-project coding conventions for naming, readability, immutability, and code-quality review. Use detailed frontend or backend skills for framework-specific patterns.
- `cpp-coding-standards` - C++ coding standards based on the C++ Core Guidelines (isocpp.github.io). Use when writing, reviewing, or refactoring C++ code to enforce modern, safe, and idiomatic practices.
- `git-commit` - Execute git commit with conventional commit message analysis, intelligent staging, and message generation. Use when user asks to commit changes, create a git commit, or mentions "/commit". Supports: (1) Auto-detecting type and scope from changes, (2) Generating conventional commit messages from diff, (3) Interactive commit with optional type/scope/description overrides, (4) Intelligent file staging for logical grouping
- `python-patterns` - Pythonic idioms, PEP 8 standards, type hints, and best practices for building robust, efficient, and maintainable Python applications. Use when writing or reviewing Python code and idiomatic structure, typing, or PEP 8 is in question.
- `rust-patterns` - Idiomatic Rust patterns, ownership, error handling, traits, concurrency, and best practices for building safe, performant applications.

## Creating a Skill

1. Create a new directory under `skills/`
2. Add a `SKILL.md` file with YAML frontmatter and markdown content
3. Optionally add a `references/` directory with reference files

## SKILL.md Format

```markdown
---
name: skill-name
description: A short description of what this skill does.
---

# Skill Name

Instructions for the AI agent go here.
```

## License

MIT