# Rust SDK Reference

Crate: `lnbot` | Install: `cargo add lnbot`
Edition: Rust 2021 | Async: `tokio` + `reqwest`

```toml
[dependencies]
lnbot = "1"
tokio = { version = "1", features = ["full"] }
```

## Constructor

```rust
use lnbot::LnBot;

let client = LnBot::unauthenticated();                            // no auth
let client = LnBot::new("uk_...");                                 // authenticated
let client = LnBot::new("uk_...").with_base_url("https://...");    // custom URL
```

## Account — `client.*`

```rust
// Register (no auth)
let account = client.register().await?;
// → .primary_key, .recovery_passphrase

// Identity
let me = client.me().await?;
```

## Wallets — `client.wallets()`

```rust
// Create
let wallet = client.wallets().create().await?;
// → .wallet_id, .address

// List
let wallets = client.wallets().list().await?;
```

## Wallet handle — `client.wallet(id)`

```rust
let w = client.wallet("wal_...");

// Get wallet info and balance
let info = w.get().await?;
// → .available: i64 (sats)

// Update
use lnbot::UpdateWalletRequest;
w.update(&UpdateWalletRequest::new("new-name")).await?;
```

## Invoices — `w.invoices()`

```rust
use lnbot::CreateInvoiceRequest;

// Create (builder pattern)
let invoice = w.invoices().create(
    &CreateInvoiceRequest::new(1000).memo("Payment")
).await?;
// → .bolt11: String, .number: u64

// Get
let invoice = w.invoices().get(invoice_number).await?;

// Watch (Stream-based SSE, requires wk_)
use futures_util::StreamExt;
use lnbot::InvoiceEventType;

let mut stream = w.invoices().watch(invoice.number, None);
while let Some(event) = stream.next().await {
    let event = event?;
    if event.event == InvoiceEventType::Settled {
        println!("Paid!");
        break;
    }
}
```

```rust
// Create invoice for a wallet (no auth required)
use lnbot::CreateInvoiceForWalletRequest;
let inv = client.invoices().create_for_wallet(
    &CreateInvoiceForWalletRequest::new("wal_xxx", 1000)
).await?;

// Create invoice for a Lightning address (no auth required)
use lnbot::CreateInvoiceForAddressRequest;
let inv = client.invoices().create_for_address(
    &CreateInvoiceForAddressRequest::new("alice@ln.bot", 1000)
).await?;
```

## Payments — `w.payments()`

```rust
use lnbot::CreatePaymentRequest;

// Pay Lightning address, LNURL, or BOLT11 invoice (builder pattern)
w.payments().create(
    &CreatePaymentRequest::new("alice@ln.bot").amount(500)
).await?;

// Pay BOLT11 invoice
w.payments().create(
    &CreatePaymentRequest::new("lnbc1...")
).await?;

// Resolve (inspect target before paying)
let res = w.payments().resolve("alice@ln.bot").await?;

// Watch (Stream-based SSE, requires wk_)
use lnbot::PaymentEventType;

let mut stream = w.payments().watch(payment.number, None);
while let Some(event) = stream.next().await {
    let event = event?;
    if event.event == PaymentEventType::Settled {
        println!("Payment complete!");
        break;
    }
}
```

## Addresses — `w.addresses()`

```rust
w.addresses().create(&CreateAddressRequest::default()).await?;       // random
w.addresses().create(&CreateAddressRequest {
    address: Some("alice".into()),
}).await?;                                                            // vanity
let addrs = w.addresses().list().await?;
w.addresses().delete("alice").await?;
w.addresses().transfer("alice", &TransferAddressRequest {
    target_wallet_key: "wk_...".into(),
}).await?;
```

## Transactions — `w.transactions()`

```rust
let txs = w.transactions().list(Some(50), None).await?;
```

## Webhooks — `w.webhooks()`

```rust
let webhook = w.webhooks().create(&CreateWebhookRequest {
    url: "https://example.com/hook".into(),
}).await?;
// → .id, .secret (shown once). Max 10 per wallet.

let webhooks = w.webhooks().list().await?;
w.webhooks().delete("webhook_id").await?;
```

## Events — `w.events()`

```rust
// Stream all wallet events (SSE, requires wk_)
let mut stream = w.events().stream();
```

## Wallet Key — `w.key()`

```rust
// Create (one key per wallet)
let key = w.key().create().await?;
// → .key ("wk_...") — shown once

// Get metadata
let info = w.key().get().await?;

// Rotate (old key invalidated)
let rotated = w.key().rotate().await?;

// Delete
w.key().delete().await?;
```

## Keys — `client.keys()`

```rust
let new_key = client.keys().rotate(0).await?;  // 0 = primary, 1 = secondary
```

## Backup — `client.backup()`

```rust
let passphrase = client.backup().recovery().await?;  // 12-word BIP-39
```

## Restore — `client.restore()`

```rust
let client = LnBot::unauthenticated();
let wallet = client.restore().recovery("word1 word2 ... word12").await?;
// → .primary_key
```

## L402 — `w.l402()`

```rust
use lnbot::{CreateL402ChallengeRequest, PayL402Request, VerifyL402Request};

// Create challenge (server side)
let challenge = w.l402().create_challenge(&CreateL402ChallengeRequest {
    amount: 100,
    description: Some("API access".into()),
    expiry_seconds: Some(3600),
    caveats: Some(vec!["tier=pro".into()]),
}).await?;
// → .macaroon, .invoice, .payment_hash, .expires_at, .www_authenticate

// Pay challenge (client side)
let result = w.l402().pay(&PayL402Request {
    www_authenticate: challenge.www_authenticate,
    max_fee: Some(10),
    reference: None,
    wait: None,
    timeout: None,
}).await?;
// → .authorization, .payment_hash, .preimage, .amount, .fee, .payment_number, .status

// Verify token (server side, stateless)
let v = w.l402().verify(&VerifyL402Request {
    authorization: result.authorization.unwrap(),
}).await?;
// → .valid, .payment_hash, .caveats, .error
```

## Errors

```rust
use lnbot::LnBotError;

match w.invoices().get(999).await {
    Ok(inv) => println!("{:?}", inv),
    Err(LnBotError::NotFound { body }) => eprintln!("not found: {body}"),
    Err(LnBotError::BadRequest { body }) => eprintln!("bad request: {body}"),
    Err(LnBotError::Conflict { body }) => eprintln!("conflict: {body}"),
    Err(e) => eprintln!("error: {e}"),
}
```

| Variant | HTTP |
|---------|------|
| `LnBotError::BadRequest { body }` | 400 |
| `LnBotError::NotFound { body }` | 404 |
| `LnBotError::Conflict { body }` | 409 |

## Key imports

```rust
use lnbot::{
    LnBot, CreateInvoiceRequest,
    CreateInvoiceForWalletRequest, CreateInvoiceForAddressRequest,
    CreatePaymentRequest, InvoiceEventType, PaymentEventType, LnBotError,
};
use futures_util::StreamExt;
```

## Conventions

- All methods: **snake_case** (`wallets()`, `create`, `watch`)
- All fields: **snake_case** (`primary_key`, `bolt11`)
- Account-level resources: `client.wallets()`, `client.invoices()`, `client.keys()`
- Wallet-scoped resources: `w.invoices()`, `w.payments()`, `w.addresses()`, `w.l402()`
- Optional fields: `Option<T>`
- Request structs: builder pattern (`CreateInvoiceRequest::new(1000).memo("...")`)
- Enums: typed (`InvoiceEventType::Settled`), `#[non_exhaustive]`
- All I/O: async (`.await`)
