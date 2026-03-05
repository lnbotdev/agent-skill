# Go SDK Reference

Module: `github.com/lnbotdev/go-sdk` | Install: `go get github.com/lnbotdev/go-sdk`
Import as: `lnbot` | Requires: Go 1.21+ | Zero dependencies (stdlib only)

## Constructor

```go
import lnbot "github.com/lnbotdev/go-sdk"

client := lnbot.New("")                        // no auth (register/public invoices)
client := lnbot.New("uk_...")                  // authenticated
client := lnbot.New("uk_...",
    lnbot.WithBaseURL("https://api.ln.bot"),
    lnbot.WithHTTPClient(customHTTPClient),
)
```

```go
// Helper for optional pointer fields
func Ptr[T any](value T) *T
```

## Account — `client.*`

```go
// Register (no auth)
account, err := client.Register(ctx)
// → .PrimaryKey, .RecoveryPassphrase

// Identity
me, err := client.Me(ctx)
```

## Wallets — `client.Wallets`

```go
// Create
wallet, err := client.Wallets.Create(ctx)
// → .WalletID, .Address

// List
wallets, err := client.Wallets.List(ctx)
```

## Wallet handle — `client.Wallet(id)`

```go
w := client.Wallet("wal_...")

// Get wallet info and balance
info, err := w.Get(ctx)
// → .Available int64 (sats)

// Update
_, err := w.Update(ctx, &lnbot.UpdateWalletParams{
    Name: lnbot.Ptr("new-name"),
})
```

## Invoices — `w.Invoices`

```go
// Create
invoice, err := w.Invoices.Create(ctx, &lnbot.CreateInvoiceParams{
    Amount: 1000,                  // int64, required
    Memo:   lnbot.Ptr("Payment"), // *string, optional
})
// → .Bolt11, .Number

// List (cursor-based)
invoices, err := w.Invoices.List(ctx, &lnbot.ListParams{Limit: lnbot.Ptr(10)})

// Get by number
invoice, err := w.Invoices.Get(ctx, number)

// Watch (channel-based SSE, requires wk_)
events, errs := w.Invoices.Watch(ctx, invoice.Number, nil)
for event := range events {
    if event.Event == "settled" {
        break
    }
}
if err := <-errs; err != nil {
    log.Fatal(err)
}

// Create invoice for a wallet (no auth required)
inv, err := client.Invoices.CreateForWallet(ctx, &lnbot.CreateInvoiceForWalletParams{
    WalletID: "wal_xxx",
    Amount:   1000,
})

// Create invoice for a Lightning address (no auth required)
inv, err := client.Invoices.CreateForAddress(ctx, &lnbot.CreateInvoiceForAddressParams{
    Address: "alice@ln.bot",
    Amount:  1000,
})
```

## Payments — `w.Payments`

```go
// Pay Lightning address, LNURL, or BOLT11 invoice
_, err := w.Payments.Create(ctx, &lnbot.CreatePaymentParams{
    Target: "alice@ln.bot",          // string, required
    Amount: lnbot.Ptr(int64(500)),   // *int64, optional
})

// Resolve (inspect target before paying)
res, err := w.Payments.Resolve(ctx, "alice@ln.bot")

// List
payments, err := w.Payments.List(ctx, &lnbot.ListParams{Limit: lnbot.Ptr(10)})

// Get by number
payment, err := w.Payments.Get(ctx, number)

// Watch (channel-based SSE, requires wk_)
events, errs := w.Payments.Watch(ctx, payment.Number, nil)
for event := range events {
    if event.Event == "settled" {
        break
    }
}
```

## Addresses — `w.Addresses`

```go
addr, err := w.Addresses.Create(ctx, &lnbot.CreateAddressParams{
    Address: lnbot.Ptr("alice"),   // *string, optional vanity
})
addrs, err := w.Addresses.List(ctx)
err = w.Addresses.Delete(ctx, "alice")
err = w.Addresses.Transfer(ctx, "alice", &lnbot.TransferAddressParams{
    TargetWalletKey: "wk_...",
})
```

## Transactions — `w.Transactions`

```go
txs, err := w.Transactions.List(ctx, &lnbot.ListParams{
    Limit: lnbot.Ptr(50),
})
```

## Webhooks — `w.Webhooks`

```go
webhook, err := w.Webhooks.Create(ctx, &lnbot.CreateWebhookParams{
    URL: "https://example.com/hook",
})
// → .ID, .Secret (shown once). Max 10 per wallet.

webhooks, err := w.Webhooks.List(ctx)
err = w.Webhooks.Delete(ctx, "webhook_id")
```

## Events — `w.Events`

```go
// Stream all wallet events (SSE, requires wk_)
events, errs := w.Events.Stream(ctx)
```

## Wallet Key — `w.Key`

```go
// Create (one key per wallet)
key, err := w.Key.Create(ctx)
// → .Key ("wk_...") — shown once

// Get metadata
info, err := w.Key.Get(ctx)

// Rotate (old key invalidated)
rotated, err := w.Key.Rotate(ctx)

// Delete
err = w.Key.Delete(ctx)
```

## Keys — `client.Keys`

```go
newKey, err := client.Keys.Rotate(ctx, 0)  // 0 = primary, 1 = secondary
```

## Backup — `client.Backup`

```go
passphrase, err := client.Backup.Recovery(ctx)
session, err := client.Backup.PasskeyBegin(ctx)
err = client.Backup.PasskeyComplete(ctx, &lnbot.PasskeyCompleteParams{...})
```

## Restore — `client.Restore`

```go
client := lnbot.New("") // no auth
wallet, err := client.Restore.Recovery(ctx, &lnbot.RecoveryParams{
    Passphrase: "word1 word2 ... word12",
})
// → .PrimaryKey, .SecondaryKey
```

## L402 — `w.L402`

```go
// Create challenge (server side)
challenge, err := w.L402.CreateChallenge(ctx, &lnbot.CreateL402ChallengeParams{
    Amount:        100,
    Description:   lnbot.Ptr("API access"),
    ExpirySeconds: lnbot.Ptr(3600),
    Caveats:       []string{"tier=pro"},
})
// → .Macaroon, .Invoice, .PaymentHash, .ExpiresAt, .WwwAuthenticate

// Pay challenge (client side)
result, err := w.L402.Pay(ctx, &lnbot.PayL402Params{
    WwwAuthenticate: challenge.WwwAuthenticate,
    MaxFee:          lnbot.Ptr(int64(10)),
})
// → .Authorization, .PaymentHash, .Preimage, .Amount, .Fee, .PaymentNumber, .Status

// Verify token (server side, stateless)
v, err := w.L402.Verify(ctx, &lnbot.VerifyL402Params{
    Authorization: *result.Authorization,
})
// → .Valid, .PaymentHash, .Caveats, .Error
```

## Errors

```go
import "errors"

var notFound *lnbot.NotFoundError
if errors.As(err, &notFound) {
    fmt.Println(notFound.StatusCode, notFound.Message)
}
```

| Type | HTTP |
|------|------|
| `*BadRequestError` | 400 |
| `*UnauthorizedError` | 401 |
| `*ForbiddenError` | 403 |
| `*NotFoundError` | 404 |
| `*ConflictError` | 409 |
| `*APIError` | base |

## Conventions

- All methods: **PascalCase** (`Create`, `Get`, `Watch`)
- All fields: **PascalCase** (`PrimaryKey`, `StatusCode`)
- Account-level resources: `client.Wallets`, `client.Invoices`, `client.Keys`
- Wallet-scoped resources: `w.Invoices`, `w.Payments`, `w.Addresses`, `w.L402`
- Optional fields: pointers (`*string`, `*int64`) + `lnbot.Ptr()` helper
- All methods take `context.Context` as first parameter
