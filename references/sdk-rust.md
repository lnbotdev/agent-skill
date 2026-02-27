# Rust SDK Reference

Crate: `lnbot` | Install: `cargo add lnbot`
Edition: Rust 2021 | Async: `tokio` + `reqwest`

```toml
[dependencies]
lnbot = "0.1"
tokio = { version = "1", features = ["full"] }
```

## Constructor

```rust
use lnbot::LnBot;

let client = LnBot::unauthenticated();                            // no auth
let client = LnBot::new("key_...");                               // authenticated
let client = LnBot::new("key_...").with_base_url("https://...");  // custom URL
```

## Wallets — `client.wallets()`

```rust
use lnbot::CreateWalletRequest;

// Create (no auth required)
let wallet = client.wallets().create(&CreateWalletRequest {
    name: Some("my-agent".into()),  // Option<String>
}).await?;
// → .primary_key, .address, .recovery_passphrase, .available

// Get current
let wallet = client.wallets().current().await?;
// → .available: i64 (sats)
```

## Invoices — `client.invoices()`

```rust
use lnbot::CreateInvoiceRequest;

// Create (builder pattern)
let invoice = client.invoices().create(
    &CreateInvoiceRequest::new(1000).memo("Payment")
).await?;
// → .bolt11: String, .number: u64

// Get
let invoice = client.invoices().get(invoice_id).await?;

// Watch (Stream-based SSE)
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

`watch()` params: `number: i32`, `timeout: Option<i32>`

## Payments — `client.payments()`

```rust
use lnbot::CreatePaymentRequest;

// Pay Lightning address, LNURL, or BOLT11 invoice (builder pattern)
client.payments().create(
    &CreatePaymentRequest::new("alice@ln.bot").amount(500)
).await?;

// Pay BOLT11 invoice
client.payments().create(
    &CreatePaymentRequest::new("lnbc1...")
).await?;

// Watch (Stream-based SSE)
use futures_util::StreamExt;
use lnbot::PaymentEventType;

let mut stream = client.payments().watch(payment.number, None);
while let Some(event) = stream.next().await {
    let event = event?;
    if event.event == PaymentEventType::Settled {
        println!("Payment complete!");
        break;
    }
}
```

## Addresses — `client.addresses()`

```rust
client.addresses().create(&CreateAddressRequest::default()).await?;       // random
client.addresses().create(&CreateAddressRequest {
    address: Some("alice".into()),
}).await?;                                                                 // vanity
let addrs = client.addresses().list().await?;
client.addresses().delete("alice").await?;
client.addresses().transfer("alice", &TransferAddressRequest {
    target_wallet_key: "key_...".into(),
}).await?;
```

## Transactions — `client.transactions()`

```rust
let txs = client.transactions().list(Some(50), None).await?;
```

## Webhooks — `client.webhooks()`

```rust
let webhook = client.webhooks().create(&CreateWebhookRequest {
    url: "https://example.com/hook".into(),
}).await?;
// → .id, .secret (shown once). Max 10 per wallet.

let webhooks = client.webhooks().list().await?;
client.webhooks().delete("webhook_id").await?;
```

## Keys — `client.keys()`

```rust
let keys = client.keys().list().await?;
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

## Errors

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

| Variant | HTTP |
|---------|------|
| `LnBotError::BadRequest { body }` | 400 |
| `LnBotError::NotFound { body }` | 404 |
| `LnBotError::Conflict { body }` | 409 |

## Key imports

```rust
use lnbot::{
    LnBot, CreateWalletRequest, CreateInvoiceRequest,
    CreateInvoiceForWalletRequest, CreateInvoiceForAddressRequest,
    CreatePaymentRequest, InvoiceEventType, PaymentEventType, LnBotError,
};
use futures_util::StreamExt;
```

## Conventions

- All methods: **snake_case** (`wallets()`, `create`, `watch`)
- All fields: **snake_case** (`primary_key`, `bolt11`)
- Resources: method calls (`client.wallets()`, `client.invoices()`)
- Optional fields: `Option<T>`
- Request structs: builder pattern (`CreateInvoiceRequest::new(1000).memo("...")`)
- Enums: typed (`InvoiceEventType::Settled`), `#[non_exhaustive]`
- All I/O: async (`.await`)
