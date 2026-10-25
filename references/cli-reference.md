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
lnbot invoice create --amount 1000            # Create invoice for ₿1,000 (1,000 sats)
lnbot invoice create --amount 1000 --memo "Payment for task"
lnbot invoice create --amount 1000 --no-wait  # Don't wait for payment
lnbot invoice list                            # List invoices (alias: ls)
lnbot invoice list --limit 50                 # Max results (default: 20)
lnbot invoice list --after 42                 # Pagination cursor
```

`invoice create` waits for settlement via SSE by default. Use `--no-wait` to return immediately.

### pay

```bash
lnbot pay alice@ln.bot --amount 500           # Pay Lightning address
lnbot pay lnbc1...                            # Pay BOLT11 invoice
lnbot pay lnurl1... --amount 500              # Pay LNURL
lnbot pay alice@example.com --all             # Send everything the wallet can, after fees
lnbot pay alice@ln.bot --amount 500 --no-wait # Don't wait for settlement
lnbot pay alice@ln.bot --amount 500 --yes     # Skip the fee quote and the prompt
```

Before asking, `pay` prints the amount, the network fee estimate, the service fee and the total. Waits for settlement via SSE by default. Use `--no-wait` to return immediately. Leave `--max-fee` out: the network fee defaults to a cap of 1% of the amount (min 20 sats). Pass `--max-fee <sats>` only when `pay` reports the fee is above that cap. Requires CLI 1.1.0.

While it waits, `pay` (CLI 1.2.0) prints the payment's stage and a line each time it changes, such as `Retrying: Not sent yet; ... (until 14:43)`; `payment list` shows the stage of payments in progress (`Pending · retrying`). With `--json`, read `stage`, `statusDetail` and `retryUntil` from the payment instead.

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
lnbot status                                  # Service status, then wallet status and API health
lnbot status --json                           # Adds "service" and "summary24h" to the wallet's keys
```

`lnbot status` needs no key or wallet for the service part: the overall state, each part (sending, receiving, node, webhooks) with live numbers, any operators' notice, and the last 24 hours. It exits 0 whatever the service's state; only failing to reach the API (or a 429: 10 requests a minute per IP) is an error. Requires CLI 1.2.0.

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
