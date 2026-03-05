# TypeScript SDK Reference

Package: `@lnbot/sdk` | Install: `npm install @lnbot/sdk`
Runtime: Node 18+, Bun, Deno, browsers | ESM + CJS | Zero dependencies | < 10KB

## Constructor

```typescript
import { LnBot } from "@lnbot/sdk";

new LnBot()                                    // no auth (register/restore/public invoices)
new LnBot({ apiKey: "uk_..." })                // authenticated (user key)
new LnBot({ apiKey: "wk_..." })                // authenticated (wallet key — for SSE)
new LnBot({
  apiKey: "uk_...",                            // optional
  baseUrl: "https://api.ln.bot",               // optional — default shown
  fetch: customFetch                           // optional — for testing/proxies
})
```

## Account — `ln.*`

```typescript
// Register (no auth)
const account = await ln.register();
// → { primaryKey, recoveryPassphrase }

// Identity
const me = await ln.me();
```

## Wallets — `ln.wallets`

```typescript
// Create
const wallet = await ln.wallets.create();
// → { walletId, address }

// List
const wallets = await ln.wallets.list();
```

## Wallet handle — `ln.wallet(id)`

```typescript
const w = ln.wallet("wal_...");

// Get wallet info and balance
const info = await w.get();
// → { available: number } (balance in sats)

// Update name
await w.update({ name: "new-name" });
```

## Invoices — `w.invoices`

```typescript
// Create
const invoice = await w.invoices.create({ amount: 1000, memo: "Payment" });
// → { bolt11: string, number: number }

// List (cursor-based pagination)
const invoices = await w.invoices.list({ limit: 10, after: "cursor" });

// Get by number or payment hash
const invoice = await w.invoices.get("numberOrHash");

// Watch for settlement (SSE stream — AsyncIterable, requires wk_)
for await (const event of w.invoices.watch(number, timeout, signal)) {
  if (event.event === "settled") break;
  if (event.event === "expired") throw new Error("expired");
}

// Create invoice for a wallet (no auth required)
const inv = await ln.invoices.createForWallet({ walletId: "wal_xxx", amount: 1000 });

// Create invoice for a Lightning address (no auth required)
const inv = await ln.invoices.createForAddress({ address: "alice@ln.bot", amount: 1000 });
```

## Payments — `w.payments`

```typescript
// Pay Lightning address, LNURL, or BOLT11 invoice
await w.payments.create({ target: "alice@ln.bot", amount: 500 });

// Pay BOLT11 invoice (amount encoded in invoice)
await w.payments.create({ target: "lnbc1..." });

// Resolve (inspect target without sending)
const res = await w.payments.resolve({ target: "alice@ln.bot" });

// List
const payments = await w.payments.list({ limit: 10, after: "cursor" });

// Get by number or hash
const payment = await w.payments.get("numberOrHash");

// Watch for settlement (SSE stream — AsyncIterable, requires wk_)
for await (const event of w.payments.watch(number, timeout, signal)) {
  if (event.event === "settled") break;
  if (event.event === "failed") throw new Error(event.data.failureReason);
}
```

## Addresses — `w.addresses`

```typescript
await w.addresses.create();                                          // random
await w.addresses.create({ address: "alice" });                      // vanity → alice@ln.bot
const addresses = await w.addresses.list();
await w.addresses.delete("alice@ln.bot");
await w.addresses.transfer("alice@ln.bot", { targetWalletKey: "wk_..." });
```

## Transactions — `w.transactions`

```typescript
const txs = await w.transactions.list({ limit: 50 });
// Each: { type: string, amount: number, balanceAfter: number }
```

## Webhooks — `w.webhooks`

```typescript
const webhook = await w.webhooks.create({ url: "https://example.com/hook" });
// → { id: string, secret: string } — secret shown once. Max 10 per wallet.

const webhooks = await w.webhooks.list();
await w.webhooks.delete("webhook_id");
```

## Events — `w.events`

```typescript
// Stream all wallet events (SSE, requires wk_)
const controller = new AbortController();
for await (const event of w.events.stream(controller.signal)) {
  console.log(event.event, event.data);
}
```

## Wallet Key — `w.key`

```typescript
// Create (one key per wallet)
const key = await w.key.create();
// → { key: "wk_..." } — shown once

// Get metadata
const info = await w.key.get();

// Rotate (old key invalidated immediately)
const rotated = await w.key.rotate();

// Delete
await w.key.delete();
```

## Keys — `ln.keys`

```typescript
const newKey = await ln.keys.rotate(0);    // 0 = primary, 1 = secondary
// → { key: string } — shown once
```

## Backup — `ln.backup`

```typescript
const { passphrase } = await ln.backup.recovery();   // 12-word BIP-39

const { options, sessionId } = await ln.backup.passkeyBegin();
await ln.backup.passkeyComplete({ sessionId, attestation: credential });
```

## Restore — `ln.restore`

```typescript
const ln = new LnBot(); // no auth needed
const { primaryKey, secondaryKey } = await ln.restore.recovery({
  passphrase: "word1 word2 ... word12"
});
```

## L402 — `w.l402`

```typescript
// Create challenge (server side)
const challenge = await w.l402.createChallenge({
  amount: 100,                    // sats
  description: "API access",     // optional — embedded in invoice
  expirySeconds: 3600,           // optional — adds expiry caveat
  caveats: ["tier=pro"],         // optional — custom caveats (max 10)
});
// → { macaroon, invoice, paymentHash, expiresAt, wwwAuthenticate }

// Pay challenge (client side)
const result = await w.l402.pay({
  wwwAuthenticate: "L402 macaroon=\"...\", invoice=\"lnbc...\"",
  maxFee: 10,                    // optional
  reference: "order-42",         // optional
  wait: true,                    // optional — wait for settlement (default true)
  timeout: 60,                   // optional — max seconds to wait
});
// → { authorization, paymentHash, preimage, amount, fee, paymentNumber, status }

// Verify token (server side, stateless)
const v = await w.l402.verify({ authorization: "L402 <macaroon>:<preimage>" });
// → { valid: boolean, paymentHash, caveats, error }
```

## Errors

```typescript
import { LnBotError, BadRequestError, UnauthorizedError, ForbiddenError, NotFoundError, ConflictError } from "@lnbot/sdk";

try {
  await w.invoices.get("bad");
} catch (e) {
  if (e instanceof NotFoundError) console.log(e.status, e.body);
}
```

| Class | HTTP |
|-------|------|
| `BadRequestError` | 400 |
| `UnauthorizedError` | 401 |
| `ForbiddenError` | 403 |
| `NotFoundError` | 404 |
| `ConflictError` | 409 |
| `LnBotError` | base |
