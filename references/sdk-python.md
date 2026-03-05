# Python SDK Reference

Package: `lnbot` | Install: `pip install lnbot`
Requires: Python 3.10+ | Env var: `LNBOT_API_KEY`

## Constructor

```python
from lnbot import LnBot, AsyncLnBot

# Sync
LnBot()                                        # reads LNBOT_API_KEY env var
LnBot(api_key="uk_...")                        # explicit key
LnBot(api_key="uk_...", base_url="https://api.ln.bot", timeout=30.0)

# Async — context manager
async with AsyncLnBot(api_key="uk_...") as ln:
    w = ln.wallet("wal_...")
    info = await w.get()
```

## Account — `ln.*`

```python
# Register (no auth)
account = ln.register()
# → .primary_key, .recovery_passphrase

# Identity
me = ln.me()
```

## Wallets — `ln.wallets`

```python
# Create
wallet = ln.wallets.create()
# → .wallet_id, .address

# List
wallets = ln.wallets.list()
```

## Wallet handle — `ln.wallet(id)`

```python
w = ln.wallet("wal_...")

# Get wallet info and balance
info = w.get()
# → .available (int, sats)

# Update
w.update(name="new-name")
```

## Invoices — `w.invoices`

```python
# Create
invoice = w.invoices.create(amount=1000, memo="Payment", reference="order-42")
# → .bolt11, .number

# List (cursor-based)
invoices = w.invoices.list(limit=10, after="cursor")

# Get
invoice = w.invoices.get("inv_number")

# Watch (sync)
for event in w.invoices.watch(invoice.number, timeout=60):
    if event.event == "settled":
        break

# Watch (async)
async for event in w.invoices.watch(invoice.number):
    if event.event == "settled":
        break

# Create invoice for a wallet (no auth required)
inv = ln.invoices.create_for_wallet(wallet_id="wal_xxx", amount=1000)

# Create invoice for a Lightning address (no auth required)
inv = ln.invoices.create_for_address(address="alice@ln.bot", amount=1000)
```

## Payments — `w.payments`

```python
# Pay Lightning address, LNURL, or BOLT11 invoice
w.payments.create(target="alice@ln.bot", amount=500)

# Pay BOLT11 invoice
w.payments.create(target="lnbc1...")

# List
payments = w.payments.list(limit=10, after="cursor")

# Get
payment = w.payments.get("pay_number")

# Watch (sync)
for event in w.payments.watch(payment.number, timeout=60):
    if event.event == "settled":
        break
    if event.event == "failed":
        raise Exception(event.data.failure_reason)

# Watch (async)
async for event in w.payments.watch(payment.number):
    if event.event == "settled":
        break
```

## Addresses — `w.addresses`

```python
w.addresses.create()              # random
w.addresses.create("alice")       # vanity → alice@ln.bot
addresses = w.addresses.list()
w.addresses.delete("alice")
w.addresses.transfer("alice", target_wallet_key="wk_...")
```

## Transactions — `w.transactions`

```python
txs = w.transactions.list(limit=50, after="cursor")
```

## Webhooks — `w.webhooks`

```python
webhook = w.webhooks.create("https://example.com/hook")
# → .id, .secret (secret shown once). Max 10 per wallet.

webhooks = w.webhooks.list()
w.webhooks.delete("webhook_id")
```

## Wallet Key — `w.key`

```python
# Create (one key per wallet)
key = w.key.create()
# → .key ("wk_...") — shown once

# Get metadata
info = w.key.get()

# Rotate (old key invalidated)
rotated = w.key.rotate()

# Delete
w.key.delete()
```

## Keys — `ln.keys`

```python
new_key = ln.keys.rotate(0)        # slot 0 = primary, 1 = secondary
# → str (the new key, shown once)
```

## Backup — `ln.backup`

```python
passphrase = ln.backup.recovery()  # 12-word BIP-39 string

session = ln.backup.passkey_begin()
ln.backup.passkey_complete(session_id=session.session_id, attestation=cred)
```

## Restore — `ln.restore`

```python
ln = LnBot()  # no auth needed
wallet = ln.restore.recovery("word1 word2 ... word12")
# → .primary_key

session = ln.restore.passkey_begin()
wallet = ln.restore.passkey_complete(session_id=session.session_id, assertion=cred)
```

## L402 — `w.l402`

```python
# Create challenge (server side)
challenge = w.l402.create_challenge(
    amount=100,                    # sats
    description="API access",     # optional
    expiry_seconds=3600,          # optional
    caveats=["tier=pro"],         # optional (max 10)
)
# → .macaroon, .invoice, .payment_hash, .expires_at, .www_authenticate

# Pay challenge (client side)
result = w.l402.pay(
    www_authenticate='L402 macaroon="...", invoice="lnbc..."',
    max_fee=10,                   # optional
    reference="order-42",         # optional
    wait=True,                    # optional (default True)
    timeout=60,                   # optional
)
# → .authorization, .payment_hash, .preimage, .amount, .fee, .payment_number, .status

# Verify token (server side, stateless)
v = w.l402.verify(authorization="L402 <macaroon>:<preimage>")
# → .valid, .payment_hash, .caveats, .error
```

## Errors

```python
from lnbot import LnBotError, BadRequestError, UnauthorizedError, NotFoundError, ConflictError

try:
    w.invoices.get("bad")
except NotFoundError as e:
    print(e.status, e.body)
```

| Class | HTTP |
|-------|------|
| `BadRequestError` | 400 |
| `UnauthorizedError` | 401 |
| `NotFoundError` | 404 |
| `ConflictError` | 409 |
| `LnBotError` | base |

## Naming

All methods, parameters, and fields use **snake_case**: `primary_key`, `recovery_passphrase`, `api_key`, `base_url`, `webhook_id`, `session_id`, `target_wallet_key`, `passkey_begin`, `passkey_complete`.
