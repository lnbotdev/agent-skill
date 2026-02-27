# lnbot Agent Skill

[Agent Skill](https://agentskills.io) for [ln.bot](https://ln.bot) — Lightning Network wallet infrastructure for AI agents.

Teaches AI coding agents (Claude Code, Codex, Copilot, etc.) how to integrate ln.bot wallets, invoices, payments, Lightning addresses, and webhooks using any of the 4 SDKs, CLI, MCP, or REST API.

## Install

### Claude Code

```bash
claude skill add --url https://github.com/lnbotdev/agent-skill
```

### Codex

```bash
codex skill add https://github.com/lnbotdev/agent-skill
```

### Copilot

```bash
copilot skill add https://github.com/lnbotdev/agent-skill
```

### Manual

```bash
git clone https://github.com/lnbotdev/agent-skill.git
# Copy or symlink into your agent's skills directory
```

## Structure

```
agent-skill/
├── SKILL.md                      # Core instructions
├── README.md                     # This file
├── LICENSE                       # MIT
└── references/
    ├── sdk-typescript.md         # TypeScript SDK — @lnbot/sdk
    ├── sdk-python.md             # Python SDK — lnbot (sync + async)
    ├── sdk-go.md                 # Go SDK — github.com/lnbotdev/go-sdk
    ├── sdk-rust.md               # Rust SDK — lnbot crate
    ├── cli-reference.md          # CLI commands and flags
    └── patterns.md               # Integration patterns and anti-patterns
```

## Links

- [ln.bot](https://ln.bot) — product homepage
- [TypeScript SDK](https://github.com/lnbotdev/typescript-sdk) — `@lnbot/sdk` on npm
- [Python SDK](https://github.com/lnbotdev/python-sdk) — `lnbot` on PyPI
- [Go SDK](https://github.com/lnbotdev/go-sdk) — `github.com/lnbotdev/go-sdk`
- [Rust SDK](https://github.com/lnbotdev/rust-sdk) — `lnbot` on crates.io
- [CLI](https://github.com/lnbotdev/cli) — `lnbot` command-line tool
- [Agent Skills spec](https://agentskills.io/specification)
