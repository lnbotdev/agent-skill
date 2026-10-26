---
name: ln.bot
description: >-
  Lightning Network wallet infrastructure for AI agents. Use ln.bot to create
  wallets, send/receive Bitcoin over Lightning, generate invoices (BOLT11),
  manage Lightning addresses (*@ln.bot), check balances, rotate API keys,
  configure webhooks, set up MCP servers for payments, and implement L402
  paywalls. Trigger when the task involves: Bitcoin payments, Lightning Network,
  sending or receiving sats, creating wallets, invoices, BOLT11, Lightning
  addresses, balance checks, API keys for payments, webhooks for payment
  notifications, MCP server config for payments, L402 paywalls, macaroons,
  HTTP 402 payment-required flows, or any mention of ln.bot. Do NOT trigger for
  on-chain Bitcoin (L1), Ethereum, other blockchains, traditional payment
  processors, or non-Lightning crypto.
license: MIT
metadata:
  author: lnbotdev
  version: "1.3"
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
| C# SDK | .NET apps | `dotnet add package LnBot` |
| CLI | Shell scripts, one-off ops | `brew install lnbotdev/tap/lnbot` |
| MCP | AI agent tool calling | `lnbot mcp config` |
| REST API | Any HTTP client | `curl https://api.ln.bot/v1/...` |

## Quick start — TypeScript

```typescript
import { LnBot } from "@lnbot/sdk";

// 1. Register (no auth needed)
const ln = new LnBot();
const account = await ln.register();

// 2. Create wallet
const authed = new LnBot({ apiKey: account.primaryKey });
const wallet = await authed.wallets.create();

// 3. Wallet operations
const w = authed.wallet(wallet.walletId);
const invoice = await w.invoices.create({ amount: 1000, memo: "Task payment" });

// 4. Wait for payment (SSE stream)
for await (const event of w.invoices.watch(invoice.number)) {
  if (event.event === "settled") break;
}

// 5. Send — pay someone
await w.payments.create({ target: "alice@ln.bot", amount: 500 });

// 6. Check balance
const info = await w.get();
console.log(info.available); // sats
```

## Quick start — Python

```python
from lnbot import LnBot

# 1. Register (no auth needed)
ln = LnBot()
account = ln.register()

# 2. Create wallet
ln = LnBot(api_key=account.primary_key)
wallet = ln.wallets.create()

# 3. Wallet operations
w = ln.wallet(wallet.wallet_id)
invoice = w.invoices.create(amount=1000, memo="Task payment")

# 4. Wait for payment
for event in w.invoices.watch(invoice.number):
    if event.event == "settled":
        break

# 5. Send
w.payments.create(target="alice@ln.bot", amount=500)

# 6. Check balance
info = w.get()
print(info.available)
```

Async variant:

```python
from lnbot import AsyncLnBot

async with AsyncLnBot(api_key="uk_...") as ln:
    w = ln.wallet("wal_...")
    invoice = await w.invoices.create(amount=1000, memo="Task payment")
    async for event in w.invoices.watch(invoice.number):
        if event.event == "settled":
            break
```

## Quick start — CLI

```bash
# Install
brew install lnbotdev/tap/lnbot

# Create an account and its first wallet
lnbot init

# Receive
lnbot invoice create --amount 1000 --memo "Task payment"

# Send (shows the fees and the total, then asks; --yes skips both)
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

MCP endpoint: `https://api.ln.bot/v1/wallets/{walletId}/mcp` (auth: Bearer token)

**Available tools:**

| Tool | Description |
|------|-------------|
| `create_invoice` | Create Lightning invoice to receive sats. Returns bolt11, number, amount, expiry. |
| `quote_payment` | Price a payment without sending it: network fee estimate, service fee, total. Omit `amount` for a Lightning address or LNURL to get the most the wallet can send there. Read-only. |
| `send_payment` | Send sats to Lightning address, LNURL or BOLT11 invoice. Sends real money. Leave `maxFee` unset. |
| `get_wallet` | Get balance, on-hold amount, available balance, and counts. |
| `list_invoices` | List invoices (newest first), paginated. |
| `list_payments` | List payments (newest first), paginated. |
| `list_transactions` | List transaction history (newest first), paginated. |
| `list_addresses` | List wallet's Lightning addresses. |
| `buy_address` | Buy a vanity address (name@ln.bot) from the balance. |
| `create_l402_challenge` | Create an L402 challenge (invoice + macaroon). |
| `pay_l402_challenge` | Pay an L402 challenge and get the Authorization header. |
| `verify_l402_token` | Verify an L402 token. |

Wallet creation, key rotation, webhooks, backup/restore are not exposed via MCP — use SDK or CLI for those.

## Core concepts

**Wallet-as-identity:** Each wallet is an autonomous identity with its own Lightning addresses and balance. All wallet-level operations are scoped via `ln.wallet("wal_...")` — invoices, payments, L402, webhooks, etc. Account-level operations (register, create wallet, list wallets, rotate keys) stay on the client directly.

**Two API keys:** Every wallet has a primary key (index 0) and secondary key (index 1). Both work for all operations. Rotate independently for zero-downtime key rotation.

**Lightning addresses:** Every wallet gets a random address at creation. Add vanity addresses (`alice@ln.bot`) or plus-addresses (`agent+task42@ln.bot`) that route to the same wallet.

**Environment auto-detection:** Python SDK reads `LNBOT_API_KEY`. CLI reads `~/.config/lnbot/config.json` (override with `LNBOT_CONFIG`).

**Idempotency:** Always pass unique identifiers when creating payments in automated systems to prevent duplicate sends.

**Recovery:** Every wallet has a 12-word BIP-39 passphrase. Store it securely — it's the only way to recover a wallet if keys are lost. Passkey (WebAuthn) backup also supported.

**SSE streaming:** Watch invoice and payment status in real-time instead of polling. All SDKs support streaming on the wallet scope: `w.invoices.watch()` in TS/Python/Rust, `w.Invoices.Watch()` with channels in Go. Invoices emit `settled`/`expired`; payments emit `settled`/`failed`. An `expired` invoice is not always final: one paid just before it expired can still turn `settled`, so re-check it (`w.invoices.get()`) before treating it as unpaid. A payment that can't be sent fails within minutes, with its funds released: about half a minute if the Lightning service refuses it, or after 10 minutes of automatic retries (`stage: "retrying"`) for a temporary problem; one already on the Lightning network stays `processing` until the network settles or fails it, usually within seconds. Show `failureReason` as is: one ending in "Please try again." can be sent again later.

**Payment stages:** a payment carries `stage`, where it is more precisely than `status` (`queued`, `retrying`, `sending`, `in_flight`, `confirming`, `on_hold`, `settled`, `failed`), and `statusDetail`, a sentence to show the user as is. A `retrying` payment also carries `retryUntil`: if it still can't be sent by then, it fails within about a minute and the funds are released. A payment can go back from `processing` to `pending` while it is retried, so treat both as in progress. More stages may be added: for one you don't know, go by `status`. The payment's event stream carries the same fields (snake_case on the wire); webhooks and the wallet event stream don't. Needs SDK 1.2.0 or CLI 1.2.0.

**Service status:** `GET https://api.ln.bot/v1/status` (public, no key, 10 requests a minute per IP, 429 over it) says whether ln.bot has a problem right now; check it before blaming your own code for failed sends. `GET /v1/status/history?range=24h|7d|30d|90d` gives the same over time. SDKs: `status()` and `statusHistory()` on the client, no key needed. CLI: `lnbot status`. A person can look at https://ln.bot/status.

**Webhooks:** Register HTTP endpoints to receive POST notifications on invoice settlement. Max 10 per wallet. Each webhook has a secret for signature verification.

**L402 (HTTP 402 paywalls):** Monetize APIs with Lightning-native authentication. Create challenges (invoice + macaroon), verify tokens statelessly, or pay challenges to get Authorization headers. All SDKs expose L402 on the wallet scope: `w.l402.createChallenge()`, `w.l402.verify()`, and `w.l402.pay()`.

## L402 paywalls

L402 lets you gate API access behind Lightning micropayments. The flow:

1. **Server** creates a challenge → returns `WWW-Authenticate` header with macaroon + invoice
2. **Client** pays the invoice → gets preimage (proof of payment)
3. **Client** sends `Authorization: L402 <macaroon>:<preimage>` on subsequent requests
4. **Server** verifies the token statelessly (signature + preimage + caveats)

Verification is stateless, so a token stays valid until it expires. To charge per request, use the `@lnbot/l402` (Express) or `LnBot.L402.AspNetCore` paywall, version 1.1 or later: each token works once, on the route and at the price it was issued for. Calling the SDK directly, record each `paymentHash` you accept and refuse repeats.

```typescript
import { LnBot } from "@lnbot/sdk";

// Server side — create a challenge
const ln = new LnBot({ apiKey: "uk_..." });
const w = ln.wallet("wal_...");

const challenge = await w.l402.createChallenge({
  amount: 100,
  description: "API access",
  expirySeconds: 3600,
  caveats: ["tier=pro"],
});
// challenge.wwwAuthenticate → send as WWW-Authenticate header in 402 response

// Client side — pay the challenge
const client = new LnBot({ apiKey: "uk_..." });
const cw = client.wallet("wal_...");
const result = await cw.l402.pay({
  wwwAuthenticate: challenge.wwwAuthenticate,
});
// result.authorization → send as Authorization header

// Server side — verify the token
const verification = await w.l402.verify({
  authorization: result.authorization!,
});
// verification.valid, verification.caveats
```

```python
from lnbot import LnBot

# Server side
ln = LnBot(api_key="uk_...")
w = ln.wallet("wal_...")
challenge = w.l402.create_challenge(amount=100, description="API access", expiry_seconds=3600)
# challenge.www_authenticate → send in 402 response

# Client side
client = LnBot(api_key="uk_...")
cw = client.wallet("wal_...")
result = cw.l402.pay(www_authenticate=challenge.www_authenticate)
# result.authorization → send as Authorization header

# Server side — verify
verification = w.l402.verify(authorization=result.authorization)
# verification.valid
```

## Pricing and fees

| Route | Cost |
|-------|------|
| ln.bot → ln.bot | Free |
| Lightning → ln.bot (receiving) | Free |
| ln.bot → Lightning (sending) | 0.25% service fee (min 1 sat) + network fee (routing) |

No monthly fees. No hidden costs.

**Leave `maxFee` unset.** It caps the network fee (the routing; the service fee is separate) and defaults to 1% of the amount (min 20 sats). A hard-coded value like `10` rejects most payments above about 4,000 sats. While a payment is in flight only the fee estimate plus a small buffer is held; on success the wallet is debited `amount + actualFee + serviceFee`.

**Price before paying.** `resolve` with an amount returns the fees without sending anything (MCP: `quote_payment`):

```typescript
const quote = await w.payments.resolve({ target: "alice@example.com", amount: 5000 });
// quote.internal (true = ln.bot wallet, no fees), quote.networkFeeEstimate,
// quote.serviceFee, quote.total, quote.feeWarning (set = fee unusually high)
```

**Send everything.** Resolve a Lightning address or LNURL without an amount, then pay `maxSendable` with no `maxFee`:

```typescript
const { maxSendable } = await w.payments.resolve({ target: "alice@example.com" });
if (maxSendable) await w.payments.create({ target: "alice@example.com", amount: maxSendable });
```

Pricing a Lightning address or LNURL fetches an invoice from the recipient, so it is slower; call it once per payment, not in a loop. Requires SDK 1.1.0: Python `resolve(target=..., amount=...)`, Go `ResolveWithAmount`, Rust `resolve_with_amount`, C# `ResolveAsync(target, amount)`.

## Error handling

| HTTP | TypeScript | Python | Go | Rust | C# |
|------|-----------|--------|-----|------|----|
| 400 | `BadRequestError` | `BadRequestError` | `*BadRequestError` | `LnBotError::BadRequest { body }` | `BadRequestException` |
| 401 | — | `UnauthorizedError` | `*UnauthorizedError` | — | `UnauthorizedException` |
| 404 | `NotFoundError` | `NotFoundError` | `*NotFoundError` | `LnBotError::NotFound { body }` | `NotFoundException` |
| 409 | `ConflictError` | `ConflictError` | `*ConflictError` | `LnBotError::Conflict { body }` | `ConflictException` |
| base | `LnBotError` | `LnBotError` | `*APIError` | `LnBotError` | `LnBotException` |

All errors carry `status` and `body`/`message` fields.

## Pagination

List endpoints use cursor-based pagination:

```typescript
// TypeScript
const w = ln.wallet("wal_...");
const page1 = await w.invoices.list({ limit: 10 });
const page2 = await w.invoices.list({ limit: 10, after: lastId });
```

```python
# Python
w = ln.wallet("wal_...")
page1 = w.invoices.list(limit=10)
page2 = w.invoices.list(limit=10, after=last_id)
```

Applies to: `invoices.list`, `payments.list`, `transactions.list`.

## References

- [TypeScript SDK](references/sdk-typescript.md) — `@lnbot/sdk` complete API
- [Python SDK](references/sdk-python.md) — `lnbot` complete API (sync + async)
- [Go SDK](references/sdk-go.md) — `github.com/lnbotdev/go-sdk` complete API
- [Rust SDK](references/sdk-rust.md) — `lnbot` crate complete API
- [C# SDK](references/sdk-csharp.md) — `LnBot` NuGet package complete API
- [CLI reference](references/cli-reference.md) — all commands, flags, config format
- [Integration patterns](references/patterns.md) — common patterns and anti-patterns
- [OpenAPI spec](https://api.ln.bot/openapi/v1.json) — full request/response schemas, status codes
