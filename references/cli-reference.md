# CLI Reference

## Installation

```bash
# macOS/Linux one-liner
curl -fsSL https://ln.bot/install.sh | bash

# Homebrew
brew install lnbotdev/tap/lnbot

# Go install
go install github.com/lnbotdev/cli@latest

# Binary download
# https://github.com/lnbotdev/cli/releases
# Platforms: linux, darwin, windows | Architectures: amd64, arm64
```

## Configuration

Config file: `~/.config/lnbot/config.json`

Override path: `LNBOT_CONFIG` env var.

## Global flags

| Flag | Short | Description |
|------|-------|-------------|
| `--wallet <name>` | `-w` | Target a specific wallet |
| `--json` | | Machine-readable JSON output |
| `--yes` | `-y` | Skip confirmation prompts |
| `--help` | | Show help |

## Commands

### Getting started

```bash
lnbot init                         # Create local config file
```

### Wallet management

```bash
lnbot wallet create --name <name>  # Create new wallet
lnbot wallet list                  # List wallets
lnbot wallet switch                # Switch active wallet
lnbot wallet rename                # Rename wallet
lnbot wallet delete                # Delete wallet
```

### Money operations

```bash
lnbot balance                      # Show wallet balance
lnbot invoice --amount <sats> --memo <text>  # Create invoice
lnbot pay <recipient> --amount <sats>        # Pay address or BOLT11
lnbot payment                      # List outgoing payments
lnbot transactions                 # List all transactions
```

`<recipient>` is a Lightning address (`alice@ln.bot`) or BOLT11 invoice (`lnbc1...`).

### Identity

```bash
lnbot address                      # Manage Lightning addresses (buy, list, transfer, delete)
lnbot whoami                       # Show current wallet info
lnbot status                       # Wallet status and API health
```

### Security

```bash
lnbot key                          # Show or rotate API keys
lnbot backup                       # Generate recovery passphrase or register passkey
lnbot restore                      # Restore wallet from passphrase or passkey
```

### Integrations

```bash
lnbot webhook                      # Register, list, delete webhook endpoints
lnbot mcp config                   # Generate MCP config (local/stdio)
lnbot mcp config --remote          # Generate MCP config (remote/SSE)
```

### Shell completions

```bash
source <(lnbot completion bash)
source <(lnbot completion zsh)
lnbot completion fish | source
```

## JSON output

All commands support `--json` for machine-readable output:

```bash
lnbot balance --json
# {"available": 42000}

lnbot invoice --amount 1000 --json
# {"bolt11": "lnbc1...", "number": "..."}
```

## Multi-wallet

Use `-w` to target a specific wallet without switching:

```bash
lnbot balance -w agent-1
lnbot pay alice@ln.bot --amount 100 -w agent-2
```

## MCP server

The `lnbot mcp config` command generates JSON configuration for AI agent MCP clients.

**Local (stdio) mode** — runs `npx @lnbot/mcp` as a subprocess:

```bash
lnbot mcp config
```

**Remote (SSE) mode** — connects directly to the ln.bot SSE endpoint:

```bash
lnbot mcp config --remote
```

Paste the output into your AI agent's MCP configuration file (Claude Desktop, Cursor, Windsurf, etc.).
