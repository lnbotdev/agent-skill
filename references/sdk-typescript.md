# TypeScript SDK Reference

Package: `@lnbot/sdk` | Install: `npm install @lnbot/sdk`
Runtime: Node 18+, Bun, Deno, browsers | ESM + CJS | Zero dependencies | < 10KB

## Constructor

```typescript
import { LnBot } from "@lnbot/sdk";

new LnBot()                                    // no auth (wallet create/restore only)
new LnBot({ apiKey: "key_..." })               // authenticated
new LnBot({
  apiKey: "key_...",                           // optional
  baseUrl: "https://api.ln.bot",               // optional — default shown
  fetch: customFetch                           // optional — for testing/proxies
})
```

## Wallets — `ln.wallets`

```typescript
// Create (no auth required)
const wallet = await ln.wallets.create({ name: "my-agent" });
// → { primaryKey, address, recoveryPassphrase, available }

// Get current wallet
const wallet = await ln.wallets.current();
// → { available: number } (balance in sats)

// Update name
await ln.wallets.update({ name: "new-name" });
```

## Invoices — `ln.invoices`

```typescript
// Create
const invoice = await ln.invoices.create({ amount: 1000, memo: "Payment" });
// → { bolt11: string, number: string }

// List (cursor-based pagination)
const invoices = await ln.invoices.list({ limit: 10, after: "cursor" });

// Get by number
const invoice = await ln.invoices.get("inv_number");

// Watch for settlement (SSE stream — AsyncIterable)
for await (const event of ln.invoices.watch(number, timeout, signal)) {
  if (event.event === "settled") break;
  if (event.event === "expired") throw new Error("expired");
}

// Create invoice for a wallet (no auth required)
const inv = await ln.invoices.createForWallet({ walletId: "wal_xxx", amount: 1000 });

// Create invoice for a Lightning address (no auth required)
const inv = await ln.invoices.createForAddress({ address: "alice@ln.bot", amount: 1000 });
```

`watch()` params: `number: number`, `timeout?: number`, `signal?: AbortSignal`

## Payments — `ln.payments`

```typescript
// Pay Lightning address, LNURL, or BOLT11 invoice
await ln.payments.create({ target: "alice@ln.bot", amount: 500 });

// Pay BOLT11 invoice (amount encoded in invoice)
await ln.payments.create({ target: "lnbc1..." });

// List
const payments = await ln.payments.list({ limit: 10, after: "cursor" });

// Watch for settlement (SSE stream — AsyncIterable)
for await (const event of ln.payments.watch(number, timeout, signal)) {
  if (event.event === "settled") break;
  if (event.event === "failed") throw new Error(event.data.failureReason);
}
```

## Addresses — `ln.addresses`

```typescript
await ln.addresses.create();                                          // random
await ln.addresses.create({ address: "alice" });                      // vanity → alice@ln.bot
const addresses = await ln.addresses.list();
await ln.addresses.delete("alice");
await ln.addresses.transfer("alice", { targetWalletKey: "key_..." });
```

## Transactions — `ln.transactions`

```typescript
const txs = await ln.transactions.list({ limit: 50 });
// Each: { type: string, amount: number, note: string }
```

## Webhooks — `ln.webhooks`

```typescript
const webhook = await ln.webhooks.create({ url: "https://example.com/hook" });
// → { id: string, secret: string } — secret shown once. Max 10 per wallet.

const webhooks = await ln.webhooks.list();
await ln.webhooks.delete("webhook_id");
```

## Keys — `ln.keys`

```typescript
const keys = await ln.keys.list();         // metadata only, not actual keys
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

## Errors

```typescript
import { LnBotError, BadRequestError, NotFoundError, ConflictError } from "@lnbot/sdk";

try {
  await ln.invoices.get("bad");
} catch (e) {
  if (e instanceof NotFoundError) console.log(e.status, e.body);
}
```

| Class | HTTP |
|-------|------|
| `BadRequestError` | 400 |
| `NotFoundError` | 404 |
| `ConflictError` | 409 |
| `LnBotError` | base |
