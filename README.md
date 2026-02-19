# ESBMC Agent Marketplace

A [Claude Code](https://docs.anthropic.com/claude-code) plugin marketplace for [ESBMC](https://github.com/esbmc/esbmc) (Efficient SMT-based Context-Bounded Model Checker) tools.

## Available Plugins

### [esbmc-plugin](esbmc-plugin/)

Formal verification integration for Claude Code. Verify C, C++, Python, Solidity, and Java/Kotlin programs for bugs, memory safety, undefined behavior, and more using ESBMC bounded model checker.

**Features:**
- `/verify` command for quick verification of source files
- `/audit` command for comprehensive security audits with multiple verification passes
- Verification skill that triggers automatically when discussing code verification topics
- Reference documentation, examples, and utility scripts

See the [plugin README](esbmc-plugin/README.md) for full documentation or the [Tutorial](esbmc-plugin/TUTORIAL.md) for a step-by-step guide.

## Installation

From within Claude Code:

```
/plugin marketplace add esbmc/agent-marketplace
/plugin install esbmc-plugin@esbmc-marketplace
```

## Prerequisites

- [ESBMC](https://github.com/esbmc/esbmc/releases) installed and available in your PATH
- [Claude Code](https://docs.anthropic.com/claude-code) CLI installed

## License

MIT License - See [LICENSE](LICENSE) for details.
