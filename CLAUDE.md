# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Code plugin marketplace for ESBMC (Efficient SMT-based Context-Bounded Model Checker). It contains the `esbmc-plugin` which integrates ESBMC formal verification into Claude Code, enabling verification of C, C++, Python, Solidity, CUDA, and Java/Kotlin programs.

## Architecture

This is a **Claude Code plugin** (not a traditional application). There is no build step, no tests, and no dependencies to install. The plugin consists entirely of markdown files and shell scripts that Claude Code interprets at runtime.

**Commands** (`esbmc-plugin/commands/*.md`): Define slash commands with YAML frontmatter (`name`, `description`, `arguments`) followed by step-by-step instructions Claude follows at invocation. Current commands: `/verify <file> [checks]` and `/audit <file>`.

**Skills** (`esbmc-plugin/skills/*/SKILL.md`): Auto-triggered by Claude Code when the user's message matches phrases listed verbatim in the skill's `description` frontmatter. The SKILL.md body is reference material Claude reads to guide the interaction. The `description` field is the trigger — changing it changes what phrases activate the skill.

**Plugin manifest** (`esbmc-plugin/.claude-plugin/plugin.json`): Declares the plugin's name, version, and author. The `name` field here is the plugin identifier used after `@` in install commands.

**Marketplace manifest** (`.claude-plugin/marketplace.json`): Top-level registry mapping plugin names to source directories. The `name` field (`esbmc-marketplace`) is what users pass to `/plugin marketplace add`.

## Development

No build, lint, or test commands. Changes are validated by:
1. Ensuring markdown frontmatter is valid YAML
2. Verifying `plugin.json` and `marketplace.json` are valid JSON
3. Testing commands/skills within Claude Code after installation

To install locally for testing:
```
/plugin marketplace add esbmc/agent-marketplace
/plugin install esbmc-plugin@esbmc-marketplace
```

## Key Conventions

- Command files: `---` delimited YAML frontmatter with `name`, `description`, and `arguments`; body is ordered steps Claude follows
- Skill files: `---` delimited YAML frontmatter with `name`, `description`, and optionally `version`; the `description` value is the exact trigger text — it must enumerate phrases users would say
- Reference docs under `skills/esbmc-verification/references/` cover CLI options, verification strategies, language-specific flags, intrinsics API, and failure diagnosis
- Shell scripts under `skills/esbmc-verification/scripts/` are standalone wrappers for quick-verify and full-audit workflows
- Java/Kotlin verification requires pre-converting `.class` files to `.jimple` via Soot before passing to ESBMC
