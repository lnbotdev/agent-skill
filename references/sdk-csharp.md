# C# SDK Reference

Package: `LnBot` | Install: `dotnet add package LnBot`
Runtime: .NET 8.0+ | Async-first | Zero dependencies (`System.Net.Http` + `System.Text.Json` only)

## Constructor

```csharp
using LnBot;

new LnBotClient()                                      // no auth (register/restore/public invoices)
new LnBotClient("uk_...")                               // authenticated (user key)
new LnBotClient("wk_...")                               // authenticated (wallet key — for SSE)
new LnBotClient("uk_...", new LnBotClientOptions
{
    BaseUrl = "https://api.ln.bot",                     // optional — default shown
    Timeout = TimeSpan.FromSeconds(30),                 // optional
    HttpClient = customHttpClient,                      // optional — bring your own
})
```

Implements `ILnBotClient` (for DI) and `IDisposable`.

## Account — `client.*`

```csharp
// Register (no auth)
var account = await client.RegisterAsync();
// → .PrimaryKey, .RecoveryPassphrase

// Identity
var me = await client.MeAsync();
```

## Wallets — `client.Wallets`

```csharp
// Create
var wallet = await client.Wallets.CreateAsync();
// → .WalletId, .Address

// List
var wallets = await client.Wallets.ListAsync();
```

## Wallet handle — `client.Wallet(id)`

```csharp
var w = client.Wallet("wal_...");

// Get wallet info and balance
var info = await w.GetAsync();
// → .Available (long, sats)

// Update name
await w.UpdateAsync(new UpdateWalletRequest { Name = "new-name" });
```

## Invoices — `w.Invoices`

```csharp
// Create
var invoice = await w.Invoices.CreateAsync(new CreateInvoiceRequest
{
    Amount = 1000,
    Memo = "Payment",           // optional
    Reference = "order-42",     // optional
});
// → .Bolt11, .Number

// List (cursor-based pagination)
var invoices = await w.Invoices.ListAsync(new PaginationParams { Limit = 10 });

// Get by number
var invoice = await w.Invoices.GetAsync(1);

// Get by payment hash
var invoice = await w.Invoices.GetByHashAsync("abc123...");

// Watch for settlement (SSE stream — IAsyncEnumerable, requires wk_)
await foreach (var evt in w.Invoices.WatchAsync(invoice.Number))
{
    if (evt.Event == "settled") break;
    if (evt.Event == "expired") throw new Exception("expired");
}

// Create invoice for a wallet (no auth required)
var inv = await client.Invoices.CreateForWalletAsync(new CreateInvoiceForWalletRequest
{
    WalletId = "wal_xxx", Amount = 1000,
});

// Create invoice for a Lightning address (no auth required)
var inv = await client.Invoices.CreateForAddressAsync(new CreateInvoiceForAddressRequest
{
    Address = "alice@ln.bot", Amount = 1000,
});
```

`WatchAsync()` params: `int number`, `int? timeout = null`, `CancellationToken`

## Payments — `w.Payments`

```csharp
// Pay Lightning address, LNURL, or BOLT11 invoice
await w.Payments.CreateAsync(new CreatePaymentRequest
{
    Target = "alice@ln.bot",
    Amount = 500,
});

// Pay BOLT11 invoice (amount encoded in invoice)
await w.Payments.CreateAsync(new CreatePaymentRequest { Target = "lnbc1..." });

// List
var payments = await w.Payments.ListAsync(new PaginationParams { Limit = 10 });

// Get by number or payment hash
var payment = await w.Payments.GetAsync(1);
var payment = await w.Payments.GetByHashAsync("abc123...");

// Watch for settlement (SSE stream — IAsyncEnumerable, requires wk_)
await foreach (var evt in w.Payments.WatchAsync(payment.Number))
{
    if (evt.Event == "settled") break;
    if (evt.Event == "failed") throw new Exception(evt.Data.FailureReason);
}
```

## Addresses — `w.Addresses`

```csharp
await w.Addresses.CreateAsync();                                                  // random
await w.Addresses.CreateAsync(new CreateAddressRequest { Address = "alice" });    // vanity → alice@ln.bot
var addresses = await w.Addresses.ListAsync();
await w.Addresses.DeleteAsync("alice");
await w.Addresses.TransferAsync("alice", new TransferAddressRequest
{
    TargetWalletKey = "wk_...",
});
```

## Transactions — `w.Transactions`

```csharp
var txs = await w.Transactions.ListAsync(new PaginationParams { Limit = 50 });
// Each: .Type (Credit/Debit), .Amount, .Note
```

## Webhooks — `w.Webhooks`

```csharp
var webhook = await w.Webhooks.CreateAsync(new CreateWebhookRequest
{
    Url = "https://example.com/hook",
});
// → .Id, .Secret (shown once). Max 10 per wallet.

var webhooks = await w.Webhooks.ListAsync();
await w.Webhooks.DeleteAsync("webhook_id");
```

## Events — `w.Events`

```csharp
// Stream all wallet events (SSE, requires wk_)
await foreach (var evt in w.Events.StreamAsync(cancellationToken))
{
    Console.WriteLine($"{evt.Event}: {evt.Data}");
}
```

## Wallet Key — `w.Key`

```csharp
// Create (one key per wallet)
var key = await w.Key.CreateAsync();
// → .Key ("wk_...") — shown once

// Get metadata
var info = await w.Key.GetAsync();

// Rotate (old key invalidated)
var rotated = await w.Key.RotateAsync();

// Delete
await w.Key.DeleteAsync();
```

## Keys — `client.Keys`

```csharp
var newKey = await client.Keys.RotateAsync(0);  // 0 = primary, 1 = secondary
// → .Key (shown once)
```

## Backup — `client.Backup`

```csharp
var backup = await client.Backup.RecoveryAsync();  // .Passphrase — 12-word BIP-39

var begin = await client.Backup.PasskeyBeginAsync();
await client.Backup.PasskeyCompleteAsync(new BackupPasskeyCompleteRequest
{
    SessionId = begin.SessionId,
    Attestation = credential,
});
```

## Restore — `client.Restore`

```csharp
using var client = new LnBotClient(); // no auth needed
var restored = await client.Restore.RecoveryAsync(new RecoveryRestoreRequest
{
    Passphrase = "word1 word2 ... word12",
});
// → .PrimaryKey, .SecondaryKey
```

## L402 — `w.L402`

```csharp
// Create challenge (server side)
var challenge = await w.L402.CreateChallengeAsync(new CreateL402ChallengeRequest
{
    Amount = 100,                          // sats
    Description = "API access",            // optional — embedded in invoice
    ExpirySeconds = 3600,                  // optional — adds expiry caveat
    Caveats = new List<string> { "tier=pro" },  // optional (max 10)
});
// → .Macaroon, .Invoice, .PaymentHash, .ExpiresAt, .WwwAuthenticate

// Pay challenge (client side)
var result = await w.L402.PayAsync(new PayL402Request
{
    WwwAuthenticate = challenge.WwwAuthenticate,
    MaxFee = 10,                           // optional
    Reference = "order-42",                // optional
    Wait = true,                           // optional — wait for settlement (default true)
    Timeout = 60,                          // optional — max seconds to wait
});
// → .Authorization, .PaymentHash, .Preimage, .Amount, .Fee, .PaymentNumber, .Status

// Verify token (server side, stateless)
var v = await w.L402.VerifyAsync(new VerifyL402Request
{
    Authorization = "L402 <macaroon>:<preimage>",
});
// → .Valid, .PaymentHash, .Caveats, .Error
```

## Errors

```csharp
using LnBot.Exceptions;

try
{
    await w.Invoices.GetAsync(999);
}
catch (NotFoundException ex)
{
    Console.WriteLine($"{ex.StatusCode}: {ex.Message}");
}
```

| Class | HTTP |
|-------|------|
| `BadRequestException` | 400 |
| `UnauthorizedException` | 401 |
| `ForbiddenException` | 403 |
| `NotFoundException` | 404 |
| `ConflictException` | 409 |
| `LnBotException` | base |

## Conventions

- All methods: **PascalCase** with `Async` suffix (`CreateAsync`, `GetAsync`, `WatchAsync`)
- All properties: **PascalCase** (`PrimaryKey`, `StatusCode`)
- Account-level resources: `client.Wallets`, `client.Invoices`, `client.Keys`
- Wallet-scoped resources: `w.Invoices`, `w.Payments`, `w.Addresses`, `w.L402`
- Optional fields: nullable types (`string?`, `long?`, `int?`)
- All methods accept `CancellationToken` as last parameter
- SSE streams: `IAsyncEnumerable<T>` via `await foreach`
- Dependency injection: implements `ILnBotClient` interface
