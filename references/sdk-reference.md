# SDK Reference

Complete method signatures for all 4 SDKs.

## Constructors

### TypeScript

```typescript
import { LnBot } from "@lnbot/sdk";

new LnBot()                                    // no auth (wallet create/restore only)
new LnBot({ apiKey: "key_..." })               // authenticated
new LnBot({ apiKey: "key_...", baseUrl: "https://api.ln.bot", fetch: customFetch })
```

### Python

```python
from lnbot import LnBot, AsyncLnBot

LnBot()                                        # reads LNBOT_API_KEY env var
LnBot(api_key="key_...")                       # explicit key
LnBot(api_key="key_...", base_url="https://api.ln.bot", timeout=30.0)

async with AsyncLnBot(api_key="key_...") as ln:  # async context manager
    ...
```

### Go

```go
import lnbot "github.com/lnbotdev/go-sdk"

client := lnbot.New("")                        // no auth
client := lnbot.New("key_...")                 // authenticated
client := lnbot.New("key_...", lnbot.WithBaseURL("..."), lnbot.WithHTTPClient(c))
```

### Rust

```rust
use lnbot::LnBot;

let client = LnBot::unauthenticated();        // no auth
let client = LnBot::new("key_...");           // authenticated
let client = LnBot::new("key_...").with_base_url("https://api.ln.bot");
```

---

## Wallets

### create

No authentication required.

```typescript
// TypeScript
const wallet = await ln.wallets.create({ name: "my-agent" });
// Returns: { primaryKey, address, recoveryPassphrase, available }
```

```python
# Python
wallet = ln.wallets.create("my-agent")
# Returns: Wallet with .primary_key, .address, .recovery_passphrase
```

```go
// Go
wallet, err := client.Wallets.Create(ctx, &lnbot.CreateWalletParams{
    Name: lnbot.Ptr("my-agent"),
})
// Returns: *Wallet { PrimaryKey, Address, RecoveryPassphrase, Available }
```

```rust
// Rust
let wallet = client.wallets().create(&CreateWalletRequest {
    name: Some("my-agent".into()),
}).await?;
// Returns: Wallet { primary_key, address, recovery_passphrase, available }
```

### current

```typescript
const wallet = await ln.wallets.current();     // { available: number }
```

```python
wallet = ln.wallets.current()                  # .available (int, sats)
```

```go
wallet, err := client.Wallets.Current(ctx)     // .Available int64
```

```rust
let wallet = client.wallets().current().await?; // .available: i64
```

### update

```typescript
await ln.wallets.update({ name: "new-name" });
```

```python
ln.wallets.update("new-name")
```

---

## Invoices

### create

```typescript
// TypeScript
const invoice = await ln.invoices.create({ amount: 1000, memo: "Payment" });
// Returns: { bolt11: string, number: string }
```

```python
# Python
invoice = ln.invoices.create(amount=1000, memo="Payment", reference="order-42")
# Returns: Invoice with .bolt11, .number
```

```go
// Go
invoice, err := client.Invoices.Create(ctx, &lnbot.CreateInvoiceParams{
    Amount: 1000,
    Memo:   lnbot.Ptr("Payment"),
})
// Returns: *Invoice { Bolt11, Number }
```

```rust
// Rust
use lnbot::CreateInvoiceRequest;

let invoice = client.invoices().create(
    &CreateInvoiceRequest::new(1000).memo("Payment")
).await?;
// Returns: Invoice { bolt11, number }
```

### list

```typescript
const invoices = await ln.invoices.list({ limit: 10, after: "cursor" });
```

```python
invoices = ln.invoices.list(limit=10, after="cursor")
```

### get

```typescript
const invoice = await ln.invoices.get("inv_number");
```

```python
invoice = ln.invoices.get("inv_number")
```

```rust
let invoice = client.invoices().get(invoice_id).await?;
```

### watch (SSE streaming)

```typescript
// TypeScript — AsyncIterable
for await (const event of ln.invoices.watch(number, timeout, signal)) {
    if (event.event === "settled") break;
}
```

```python
# Python — sync iterator
for event in ln.invoices.watch(invoice.number, timeout=60):
    if event.event == "settled":
        break

# Python — async iterator
async for event in ln.invoices.watch(invoice.number):
    if event.event == "settled":
        break
```

```go
// Go — dual channels
events, errs := client.Invoices.Watch(ctx, invoice.Number, nil)
for {
    select {
    case event := <-events:
        if event.Event == "settled" {
            return
        }
    case err := <-errs:
        log.Fatal(err)
    }
}
```

```rust
// Rust — Stream
use futures_util::StreamExt;
use lnbot::InvoiceEventType;

let mut stream = client.invoices().watch(invoice.number, None);
while let Some(event) = stream.next().await {
    let event = event?;
    if event.event == InvoiceEventType::Settled {
        println!("Paid!");
        break;
    }
}
```

---

## Payments

### create

```typescript
// TypeScript — target is Lightning address or BOLT11 invoice
await ln.payments.create({ target: "alice@ln.bot", amount: 500 });
await ln.payments.create({ target: "lnbc1..." }); // amount from invoice
```

```python
# Python
ln.payments.create(target="alice@ln.bot", amount=500)
```

```go
// Go
_, err := client.Payments.Create(ctx, &lnbot.CreatePaymentParams{
    Target: "alice@ln.bot",
    Amount: lnbot.Ptr(int64(500)),
})
```

```rust
// Rust
use lnbot::CreatePaymentRequest;

client.payments().create(
    &CreatePaymentRequest::new("alice@ln.bot").amount(500)
).await?;
```

### list

```typescript
const payments = await ln.payments.list({ limit: 10, after: "cursor" });
```

```python
payments = ln.payments.list(limit=10, after="cursor")
```

### get

```python
payment = ln.payments.get("pay_number")
```

---

## Addresses

### create

```typescript
// Random address
await ln.addresses.create();
// Vanity address
await ln.addresses.create({ address: "alice" }); // → alice@ln.bot
```

```python
ln.addresses.create()              # random
ln.addresses.create("alice")       # vanity → alice@ln.bot
```

### list

```typescript
const addresses = await ln.addresses.list();
```

```python
addresses = ln.addresses.list()
```

### delete

```typescript
await ln.addresses.delete("alice");
```

```python
ln.addresses.delete("alice")
```

### transfer

```typescript
await ln.addresses.transfer("alice", { targetWalletKey: "key_..." });
```

```python
ln.addresses.transfer("alice", target_wallet_key="key_...")
```

---

## Transactions

### list

```typescript
const txs = await ln.transactions.list({ limit: 50 });
// Each: { type, amount, note }
```

```python
txs = ln.transactions.list(limit=50, after="cursor")
```

---

## Webhooks

### create

```typescript
const webhook = await ln.webhooks.create({ url: "https://example.com/hook" });
// Returns: { id: string, secret: string } — secret shown once
```

```python
webhook = ln.webhooks.create("https://example.com/hook")
# Max 10 webhooks per wallet
```

### list

```typescript
const webhooks = await ln.webhooks.list();
```

```python
webhooks = ln.webhooks.list()
```

### delete

```typescript
await ln.webhooks.delete("webhook_id");
```

```python
ln.webhooks.delete("webhook_id")
```

---

## Keys

### list

```typescript
const keys = await ln.keys.list(); // metadata only, not the actual keys
```

```python
keys = ln.keys.list()
```

### rotate

```typescript
const newKey = await ln.keys.rotate(0); // 0 = primary, 1 = secondary
// Returns: { key: string } — shown once
```

```python
new_key = ln.keys.rotate(0)  # slot 0 = primary, 1 = secondary
# Returns: str — the new key (shown once)
```

---

## Backup

### recovery

```typescript
const { passphrase } = await ln.backup.recovery(); // 12-word BIP-39
```

```python
passphrase = ln.backup.recovery()  # 12-word BIP-39 string
```

### passkeyBegin / passkeyComplete

```typescript
const { options, sessionId } = await ln.backup.passkeyBegin();
await ln.backup.passkeyComplete({ sessionId, attestation: credential });
```

```python
session = ln.backup.passkey_begin()
ln.backup.passkey_complete(session_id=session.session_id, attestation=cred)
```

---

## Restore

### recovery

```typescript
const ln = new LnBot(); // no auth
const { primaryKey, secondaryKey } = await ln.restore.recovery({
    passphrase: "word1 word2 ... word12"
});
```

```python
ln = LnBot()  # no auth
wallet = ln.restore.recovery("word1 word2 ... word12")
# wallet.primary_key, wallet.secondary_key
```

---

## Error classes

### TypeScript

```typescript
import { LnBotError, BadRequestError, NotFoundError, ConflictError } from "@lnbot/sdk";

try {
    await ln.invoices.get("bad");
} catch (e) {
    if (e instanceof NotFoundError) {
        console.log(e.status, e.body);
    }
}
```

### Python

```python
from lnbot import LnBotError, BadRequestError, UnauthorizedError, NotFoundError, ConflictError

try:
    ln.invoices.get("bad")
except NotFoundError as e:
    print(e.status, e.body)
```

### Go

```go
import "errors"

var notFound *lnbot.NotFoundError
if errors.As(err, &notFound) {
    fmt.Println(notFound.StatusCode, notFound.Message)
}
```

### Rust

```rust
use lnbot::LnBotError;

match client.invoices().get(999).await {
    Ok(inv) => println!("{:?}", inv),
    Err(LnBotError::NotFound { body }) => eprintln!("not found: {body}"),
    Err(LnBotError::BadRequest { body }) => eprintln!("bad request: {body}"),
    Err(LnBotError::Conflict { body }) => eprintln!("conflict: {body}"),
    Err(e) => eprintln!("error: {e}"),
}
```

---

## REST API endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/wallets` | Create wallet |
| GET | `/v1/wallets/current` | Get current wallet |
| PATCH | `/v1/wallets/current` | Update wallet |
| POST | `/v1/invoices` | Create invoice |
| GET | `/v1/invoices` | List invoices |
| GET | `/v1/invoices/:number` | Get invoice |
| GET | `/v1/invoices/:number/watch` | SSE stream for invoice |
| POST | `/v1/payments` | Create payment |
| GET | `/v1/payments` | List payments |
| GET | `/v1/payments/:number` | Get payment |
| POST | `/v1/addresses` | Create address |
| GET | `/v1/addresses` | List addresses |
| DELETE | `/v1/addresses/:address` | Delete address |
| POST | `/v1/addresses/:address/transfer` | Transfer address |
| GET | `/v1/transactions` | List transactions |
| POST | `/v1/webhooks` | Create webhook |
| GET | `/v1/webhooks` | List webhooks |
| DELETE | `/v1/webhooks/:id` | Delete webhook |
| GET | `/v1/keys` | List keys |
| POST | `/v1/keys/:index/rotate` | Rotate key |
| GET | `/v1/backup/recovery` | Get recovery passphrase |
| POST | `/v1/backup/passkey/begin` | Start passkey backup |
| POST | `/v1/backup/passkey/complete` | Complete passkey backup |
| POST | `/v1/restore/recovery` | Restore from passphrase |
| POST | `/v1/restore/passkey/begin` | Start passkey restore |
| POST | `/v1/restore/passkey/complete` | Complete passkey restore |

All endpoints use `Authorization: Bearer key_...` header except wallet creation and restore.
