# C# SDK Reference

Package: `LnBot` | Install: `dotnet add package LnBot`
Runtime: .NET 8.0+ | Async-first | Zero dependencies (`System.Net.Http` + `System.Text.Json` only)

## Constructor

```csharp
using LnBot;

new LnBotClient()                                      // no auth (wallet create/restore only)
new LnBotClient("key_...")                              // authenticated
new LnBotClient("key_...", new LnBotClientOptions
{
    BaseUrl = "https://api.ln.bot",                     // optional — default shown
    Timeout = TimeSpan.FromSeconds(30),                 // optional
    HttpClient = customHttpClient,                      // optional — bring your own
})
```

Implements `ILnBotClient` (for DI) and `IDisposable`.

## Wallets — `client.Wallets`

```csharp
// Create (no auth required)
var wallet = await client.Wallets.CreateAsync(new CreateWalletRequest { Name = "my-agent" });
// → .PrimaryKey, .Address, .RecoveryPassphrase, .Available

// Get current wallet
var wallet = await client.Wallets.CurrentAsync();
// → .Available (long, sats)

// Update name
await client.Wallets.UpdateAsync(new UpdateWalletRequest { Name = "new-name" });
```

## Invoices — `client.Invoices`

```csharp
// Create
var invoice = await client.Invoices.CreateAsync(new CreateInvoiceRequest
{
    Amount = 1000,
    Memo = "Payment",           // optional
    Reference = "order-42",     // optional
});
// → .Bolt11, .Number

// List (cursor-based pagination)
var invoices = await client.Invoices.ListAsync(new PaginationParams { Limit = 10 });

// Get by number
var invoice = await client.Invoices.GetAsync(1);

// Get by payment hash
var invoice = await client.Invoices.GetByHashAsync("abc123...");

// Watch for settlement (SSE stream — IAsyncEnumerable)
await foreach (var evt in client.Invoices.WatchAsync(invoice.Number))
{
    if (evt.Event == "settled") break;
    if (evt.Event == "expired") throw new Exception("expired");
}

// Watch by payment hash
await foreach (var evt in client.Invoices.WatchByHashAsync("abc123..."))
{
    if (evt.Event == "settled") break;
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

## Payments — `client.Payments`

```csharp
// Pay Lightning address, LNURL, or BOLT11 invoice
await client.Payments.CreateAsync(new CreatePaymentRequest
{
    Target = "alice@ln.bot",
    Amount = 500,
});

// Pay BOLT11 invoice (amount encoded in invoice)
await client.Payments.CreateAsync(new CreatePaymentRequest { Target = "lnbc1..." });

// List
var payments = await client.Payments.ListAsync(new PaginationParams { Limit = 10 });

// Get by number or payment hash
var payment = await client.Payments.GetAsync(1);
var payment = await client.Payments.GetByHashAsync("abc123...");

// Watch for settlement (SSE stream — IAsyncEnumerable)
await foreach (var evt in client.Payments.WatchAsync(payment.Number))
{
    if (evt.Event == "settled") break;
    if (evt.Event == "failed") throw new Exception(evt.Data.FailureReason);
}
```

## Addresses — `client.Addresses`

```csharp
await client.Addresses.CreateAsync();                                                  // random
await client.Addresses.CreateAsync(new CreateAddressRequest { Address = "alice" });    // vanity → alice@ln.bot
var addresses = await client.Addresses.ListAsync();
await client.Addresses.DeleteAsync("alice");
await client.Addresses.TransferAsync("alice", new TransferAddressRequest
{
    TargetWalletKey = "key_...",
});
```

## Transactions — `client.Transactions`

```csharp
var txs = await client.Transactions.ListAsync(new PaginationParams { Limit = 50 });
// Each: .Type (Credit/Debit), .Amount, .Note
```

## Webhooks — `client.Webhooks`

```csharp
var webhook = await client.Webhooks.CreateAsync(new CreateWebhookRequest
{
    Url = "https://example.com/hook",
});
// → .Id, .Secret (shown once). Max 10 per wallet.

var webhooks = await client.Webhooks.ListAsync();
await client.Webhooks.DeleteAsync("webhook_id");
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

## L402 — `client.L402`

```csharp
// Create challenge (server side)
var challenge = await client.L402.CreateChallengeAsync(new CreateL402ChallengeRequest
{
    Amount = 100,                          // sats
    Description = "API access",            // optional — embedded in invoice
    ExpirySeconds = 3600,                  // optional — adds expiry caveat
    Caveats = new List<string> { "tier=pro" },  // optional (max 10)
});
// → .Macaroon, .Invoice, .PaymentHash, .ExpiresAt, .WwwAuthenticate

// Pay challenge (client side)
var result = await client.L402.PayAsync(new PayL402Request
{
    WwwAuthenticate = challenge.WwwAuthenticate,
    MaxFee = 10,                           // optional
    Reference = "order-42",                // optional
    Wait = true,                           // optional — wait for settlement (default true)
    Timeout = 60,                          // optional — max seconds to wait
});
// → .Authorization, .PaymentHash, .Preimage, .Amount, .Fee, .PaymentNumber, .Status

// Verify token (server side, stateless)
var v = await client.L402.VerifyAsync(new VerifyL402Request
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
    await client.Invoices.GetAsync(999);
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

- All methods: **PascalCase** with `Async` suffix (`CreateAsync`, `CurrentAsync`, `WatchAsync`)
- All properties: **PascalCase** (`PrimaryKey`, `StatusCode`)
- Resources: properties on the client (`client.Wallets`, `client.Invoices`)
- Optional fields: nullable types (`string?`, `long?`, `int?`)
- All methods accept `CancellationToken` as last parameter
- SSE streams: `IAsyncEnumerable<T>` via `await foreach`
- Dependency injection: implements `ILnBotClient` interface
