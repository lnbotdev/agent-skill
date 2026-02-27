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

### init

```bash
lnbot init                                    # Create local config file
```

### wallet

```bash
lnbot wallet create                           # Create wallet (name auto-generated)
lnbot wallet create --name my-agent           # Create wallet with name
lnbot wallet list                             # List wallets (alias: ls)
lnbot wallet use <name|id>                    # Switch active wallet
lnbot wallet rename <new-name>               # Rename active wallet
lnbot wallet delete [name]                    # Remove wallet from config
lnbot wallet delete [name] --force            # Skip confirmation
```

### balance

```bash
lnbot balance                                 # Show available and on-hold amounts
```

### invoice

```bash
lnbot invoice create --amount 1000            # Create invoice for 1000 sats
lnbot invoice create --amount 1000 --memo "Payment for task"
lnbot invoice list                            # List invoices (alias: ls)
lnbot invoice list --limit 50                 # Max results (default: 20)
lnbot invoice list --after 42                 # Pagination cursor
```

### pay

```bash
lnbot pay alice@ln.bot --amount 500           # Pay Lightning address
lnbot pay lnbc1...                            # Pay BOLT11 invoice
lnbot pay alice@ln.bot --amount 500 --max-fee 10  # Set max routing fee (sats)
```

### payment

```bash
lnbot payment list                            # List outgoing payments (alias: ls)
lnbot payment list --limit 50                 # Max results (default: 20)
lnbot payment list --after 42                 # Pagination cursor
```

### transactions

```bash
lnbot transactions                            # List all transactions (aliases: txns, tx)
lnbot transactions --limit 50                 # Max results (default: 20)
lnbot transactions --after 42                 # Pagination cursor
```

### address

```bash
lnbot address list                            # List Lightning addresses (alias: ls)
lnbot address buy <name>                      # Buy vanity address → name@ln.bot
lnbot address transfer <address> --to <wallet-name>        # Transfer to local wallet
lnbot address transfer <address> --target-key <api-key>    # Transfer to external wallet
lnbot address delete <address>                # Delete a Lightning address
```

### whoami / status

```bash
lnbot whoami                                  # Show current wallet info
lnbot status                                  # Wallet status and API health
```

### key

```bash
lnbot key show                                # Show API keys from local config
lnbot key rotate 0                            # Rotate primary key (slot 0)
lnbot key rotate 1                            # Rotate secondary key (slot 1)
```

### backup

```bash
lnbot backup recovery                         # Generate 12-word recovery passphrase
lnbot backup passkey                          # Register a passkey (browser)
```

### restore

```bash
lnbot restore recovery --passphrase "word1 word2 ... word12"
lnbot restore passkey                         # Restore via passkey (browser)
```

### webhook

```bash
lnbot webhook create --url https://example.com/hook   # Register endpoint
lnbot webhook list                            # List webhooks (alias: ls)
lnbot webhook delete <id>                     # Delete webhook by ID
```

### mcp

```bash
lnbot mcp config --remote                     # Generate remote MCP config (streamable HTTP)
lnbot mcp serve                               # Start local MCP server (coming soon)
```

Paste `mcp config` output into your AI agent's MCP settings (Claude Desktop, Cursor, Windsurf, etc.).

### version / update

```bash
lnbot version                                 # Print version
lnbot update                                  # Check for updates
```

### completion

```bash
source <(lnbot completion bash)
source <(lnbot completion zsh)
lnbot completion fish | source
lnbot completion powershell                   # PowerShell completions
```

## JSON output

All commands support `--json` for machine-readable output:

```bash
lnbot balance --json
lnbot invoice create --amount 1000 --json
lnbot wallet list --json
```

## Multi-wallet

Use `-w` to target a specific wallet without switching:

```bash
lnbot balance -w agent-1
lnbot pay alice@ln.bot --amount 100 -w agent-2
```
