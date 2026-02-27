---
name: lnbot
description: >-
  Lightning Network wallet infrastructure for AI agents. Use ln.bot to create
  wallets, send/receive Bitcoin over Lightning, generate invoices (BOLT11),
  manage Lightning addresses (*@ln.bot), check balances, rotate API keys,
  configure webhooks, and set up MCP servers for payments. Trigger when the task
  involves: Bitcoin payments, Lightning Network, sending or receiving sats,
  creating wallets, invoices, BOLT11, Lightning addresses, balance checks, API
  keys for payments, webhooks for payment notifications, MCP server config for
  payments, or any mention of ln.bot or lnbot. Do NOT trigger for on-chain
  Bitcoin (L1), Ethereum, other blockchains, traditional payment processors,
  or non-Lightning crypto.
license: MIT
metadata:
  author: lnbotdev
  version: "1.0"
  homepage: https://ln.bot
  repository: https://github.com/lnbotdev/agent-skill
---

# ln.bot — Bitcoin for AI Agents

Lightning wallet with CLI, MCP, and API — ready in seconds.

## Interfaces

| Interface | When to use | Install |
|-----------|------------|---------|
| TypeScript SDK | Node/Bun/Deno apps | `npm install @lnbot/sdk` |
| Python SDK | Python apps, async support | `pip install lnbot` |
| Go SDK | Go services | `go get github.com/lnbotdev/go-sdk` |
| Rust SDK | Rust services | `cargo add lnbot` |
| CLI | Shell scripts, one-off ops | `brew install lnbotdev/tap/lnbot` |
| MCP | AI agent tool calling | `lnbot mcp config` |
| REST API | Any HTTP client | `curl https://api.ln.bot/v1/...` |

## Quick start — TypeScript

```typescript
import { LnBot } from "@lnbot/sdk";

// 1. Create wallet (no auth needed)
const ln = new LnBot();
const wallet = await ln.wallets.create({ name: "my-agent" });
// wallet.primaryKey, wallet.address, wallet.recoveryPassphrase

// 2. Authenticate
const authed = new LnBot({ apiKey: wallet.primaryKey });

// 3. Receive — create invoice
const invoice = await authed.invoices.create({ amount: 1000, memo: "Task payment" });
// invoice.bolt11, invoice.number

// 4. Wait for payment (SSE stream)
for await (const event of authed.invoices.watch(invoice.number)) {
  if (event.event === "settled") break;
}

// 5. Send — pay someone
await authed.payments.create({ target: "alice@ln.bot", amount: 500 });

// 6. Check balance
const current = await authed.wallets.current();
console.log(current.available); // sats
```

## Quick start — Python

```python
from lnbot import LnBot

# 1. Create wallet (no auth needed)
ln = LnBot()
wallet = ln.wallets.create("my-agent")
# wallet.primary_key, wallet.address, wallet.recovery_passphrase

# 2. Authenticate (or set LNBOT_API_KEY env var)
ln = LnBot(api_key=wallet.primary_key)

# 3. Receive
invoice = ln.invoices.create(amount=1000, memo="Task payment")

# 4. Wait for payment
for event in ln.invoices.watch(invoice.number):
    if event.event == "settled":
        break

# 5. Send
ln.payments.create(target="alice@ln.bot", amount=500)

# 6. Check balance
current = ln.wallets.current()
print(current.available)
```

Async variant:

```python
from lnbot import AsyncLnBot

async with AsyncLnBot(api_key="key_...") as ln:
    invoice = await ln.invoices.create(amount=1000, memo="Task payment")
    async for event in ln.invoices.watch(invoice.number):
        if event.event == "settled":
            break
```

## Quick start — CLI

```bash
# Install
brew install lnbotdev/tap/lnbot

# Create wallet
lnbot wallet create --name my-agent

# Receive
lnbot invoice --amount 1000 --memo "Task payment"

# Send
lnbot pay alice@ln.bot --amount 500

# Balance
lnbot balance

# Transaction history
lnbot transactions
```

## Quick start — MCP

Generate config for your AI agent:

```bash
# Remote (streamable HTTP) — direct API endpoint
lnbot mcp config --remote
```

Paste the generated JSON into your AI agent's MCP config (Claude Desktop, Cursor, etc.).

MCP endpoint: `https://api.ln.bot/mcp` (auth: Bearer token)

**Available tools:**

| Tool | Description |
|------|-------------|
| `create_invoice` | Create Lightning invoice to receive sats. Returns bolt11, number, amount, expiry. |
| `send_payment` | Send sats to Lightning address or BOLT11 invoice. Sends real money. |
| `get_wallet` | Get balance, on-hold amount, available balance, and counts. |
| `list_invoices` | List invoices (newest first), paginated. |
| `list_payments` | List payments (newest first), paginated. |
| `list_transactions` | List transaction history (newest first), paginated. |
| `list_addresses` | List wallet's Lightning addresses. |

Wallet creation, key rotation, webhooks, backup/restore are not exposed via MCP — use SDK or CLI for those.

## Core concepts

**Wallet-as-identity:** Each wallet is an autonomous identity with its own API keys, Lightning addresses, and balance. No user accounts — wallets are the primitive.

**Two API keys:** Every wallet has a primary key (index 0) and secondary key (index 1). Both work for all operations. Rotate independently for zero-downtime key rotation.

**Lightning addresses:** Every wallet gets a random address at creation. Add vanity addresses (`alice@ln.bot`) or plus-addresses (`agent+task42@ln.bot`) that route to the same wallet.

**Environment auto-detection:** Python SDK reads `LNBOT_API_KEY`. CLI reads `~/.config/lnbot/config.json` (override with `LNBOT_CONFIG`).

**Idempotency:** Always pass unique identifiers when creating payments in automated systems to prevent duplicate sends.

**Recovery:** Every wallet has a 12-word BIP-39 passphrase. Store it securely — it's the only way to recover a wallet if keys are lost. Passkey (WebAuthn) backup also supported.

**SSE streaming:** Watch invoice settlement in real-time instead of polling. All SDKs support streaming: `watch()` in TS/Python/Rust, `Watch()` with channels in Go.

**Webhooks:** Register HTTP endpoints to receive POST notifications on invoice settlement. Max 10 per wallet. Each webhook has a secret for signature verification.

## Pricing

| Route | Cost |
|-------|------|
| ln.bot → ln.bot | Free |
| Lightning → ln.bot (receiving) | Free |
| ln.bot → Lightning (sending) | 0.1% + network fee (min 1 sat) |

No monthly fees. No hidden costs.

## Error handling

| HTTP | TypeScript | Python | Go | Rust |
|------|-----------|--------|-----|------|
| 400 | `BadRequestError` | `BadRequestError` | `*BadRequestError` | `LnBotError::BadRequest { body }` |
| 401 | — | `UnauthorizedError` | `*UnauthorizedError` | — |
| 404 | `NotFoundError` | `NotFoundError` | `*NotFoundError` | `LnBotError::NotFound { body }` |
| 409 | `ConflictError` | `ConflictError` | `*ConflictError` | `LnBotError::Conflict { body }` |
| base | `LnBotError` | `LnBotError` | `*APIError` | `LnBotError` |

All errors carry `status` and `body`/`message` fields.

## Pagination

List endpoints use cursor-based pagination:

```typescript
// TypeScript
const page1 = await ln.invoices.list({ limit: 10 });
const page2 = await ln.invoices.list({ limit: 10, after: lastId });
```

```python
# Python
page1 = ln.invoices.list(limit=10)
page2 = ln.invoices.list(limit=10, after=last_id)
```

Applies to: `invoices.list`, `payments.list`, `transactions.list`.

## References

- [TypeScript SDK](references/sdk-typescript.md) — `@lnbot/sdk` complete API
- [Python SDK](references/sdk-python.md) — `lnbot` complete API (sync + async)
- [Go SDK](references/sdk-go.md) — `github.com/lnbotdev/go-sdk` complete API
- [Rust SDK](references/sdk-rust.md) — `lnbot` crate complete API
- [CLI reference](references/cli-reference.md) — all commands, flags, config format
- [Integration patterns](references/patterns.md) — common patterns and anti-patterns
- [OpenAPI spec](https://api.ln.bot/openapi/v1.json) — full request/response schemas, status codes
