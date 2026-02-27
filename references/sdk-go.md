# Go SDK Reference

Module: `github.com/lnbotdev/go-sdk` | Install: `go get github.com/lnbotdev/go-sdk`
Import as: `lnbot` | Requires: Go 1.21+ | Zero dependencies (stdlib only)

## Constructor

```go
import lnbot "github.com/lnbotdev/go-sdk"

client := lnbot.New("")                        // no auth (wallet creation)
client := lnbot.New("key_...")                 // authenticated
client := lnbot.New("key_...",
    lnbot.WithBaseURL("https://api.ln.bot"),
    lnbot.WithHTTPClient(customHTTPClient),
)
```

```go
// Helper for optional pointer fields
func Ptr[T any](value T) *T
```

## Wallets — `client.Wallets`

```go
// Create (no auth required)
wallet, err := client.Wallets.Create(ctx, &lnbot.CreateWalletParams{
    Name: lnbot.Ptr("my-agent"),  // *string, optional
})
// → .PrimaryKey, .Address, .RecoveryPassphrase, .Available

// Get current
wallet, err := client.Wallets.Current(ctx)
// → .Available int64 (sats)
```

## Invoices — `client.Invoices`

```go
// Create
invoice, err := client.Invoices.Create(ctx, &lnbot.CreateInvoiceParams{
    Amount: 1000,                  // int64, required
    Memo:   lnbot.Ptr("Payment"), // *string, optional
})
// → .Bolt11, .Number

// Watch (channel-based SSE)
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

## Payments — `client.Payments`

```go
_, err := client.Payments.Create(ctx, &lnbot.CreatePaymentParams{
    Target: "alice@ln.bot",          // string, required
    Amount: lnbot.Ptr(int64(500)),   // *int64, optional
})
```

## Addresses — `client.Addresses`

```go
addr, err := client.Addresses.Create(ctx, &lnbot.CreateAddressParams{
    Address: lnbot.Ptr("alice"),   // *string, optional vanity
})
addrs, err := client.Addresses.List(ctx)
err = client.Addresses.Delete(ctx, "alice")
err = client.Addresses.Transfer(ctx, "alice", &lnbot.TransferAddressParams{
    TargetWalletKey: "key_...",
})
```

## Transactions — `client.Transactions`

```go
txs, err := client.Transactions.List(ctx, &lnbot.ListParams{
    Limit: lnbot.Ptr(50),
})
```

## Webhooks — `client.Webhooks`

```go
webhook, err := client.Webhooks.Create(ctx, &lnbot.CreateWebhookParams{
    URL: "https://example.com/hook",
})
// → .ID, .Secret (shown once). Max 10 per wallet.

webhooks, err := client.Webhooks.List(ctx)
err = client.Webhooks.Delete(ctx, "webhook_id")
```

## Keys — `client.Keys`

```go
keys, err := client.Keys.List(ctx)
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

- All methods: **PascalCase** (`Create`, `Current`, `Watch`)
- All fields: **PascalCase** (`PrimaryKey`, `StatusCode`)
- Resources: struct fields (`client.Wallets`, `client.Invoices`)
- Optional fields: pointers (`*string`, `*int64`) + `lnbot.Ptr()` helper
- All methods take `context.Context` as first parameter
