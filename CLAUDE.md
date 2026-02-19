# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Code plugin marketplace for ESBMC (Efficient SMT-based Context-Bounded Model Checker). It contains the `esbmc-plugin` which integrates ESBMC formal verification into Claude Code, enabling verification of C, C++, Python, Solidity, CUDA, and Java/Kotlin programs.

## Repository Structure

- `.claude-plugin/marketplace.json` — Marketplace manifest that registers available plugins
- `esbmc-plugin/` — The main plugin package
  - `.claude-plugin/plugin.json` — Plugin manifest (name, version, metadata)
  - `commands/` — Slash commands (`/verify`, `/audit`) defined as markdown files with frontmatter
  - `skills/esbmc-verification/` — Auto-triggering skill with SKILL.md, reference docs, examples, and scripts

## Architecture

This is a **Claude Code plugin** (not a traditional application). There is no build step, no tests, and no dependencies to install. The plugin consists entirely of markdown files and shell scripts that Claude Code interprets at runtime.

**Commands** (`commands/*.md`): Define slash commands with YAML frontmatter (name, description, arguments) followed by step-by-step instructions Claude follows when the command is invoked.

**Skills** (`skills/*/SKILL.md`): Auto-triggered based on keyword matching in the skill's `description` frontmatter field. The SKILL.md body contains reference material and workflows Claude uses to assist with verification tasks.

**Marketplace manifest** (`.claude-plugin/marketplace.json`): Top-level registry that maps plugin names to their source directories. This is what users reference during `plugin marketplace add`.

## Development

There are no build, lint, or test commands. Changes are validated by:
1. Ensuring markdown frontmatter is valid YAML
2. Verifying `plugin.json` and `marketplace.json` are valid JSON
3. Testing commands/skills within Claude Code after installation

To install locally for testing:
```
/plugin marketplace add esbmc/agent-marketplace
/plugin install esbmc-plugin@esbmc-marketplace
```

## Key Conventions

- Command files use `---` delimited YAML frontmatter with `name`, `description`, and `arguments` fields
- Skill files use `---` delimited YAML frontmatter with `name`, `description`, and optionally `version`
- The skill `description` field doubles as the trigger — it must list relevant phrases users might say
- Reference docs under `skills/esbmc-verification/references/` are loaded by the skill as needed
- Shell scripts under `skills/esbmc-verification/scripts/` are standalone wrappers around ESBMC
