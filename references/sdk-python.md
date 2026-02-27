# Python SDK Reference

Package: `lnbot` | Install: `pip install lnbot`
Requires: Python 3.10+ | Env var: `LNBOT_API_KEY`

## Constructor

```python
from lnbot import LnBot, AsyncLnBot

# Sync
LnBot()                                        # reads LNBOT_API_KEY env var
LnBot(api_key="key_...")                       # explicit key
LnBot(api_key="key_...", base_url="https://api.ln.bot", timeout=30.0)

# Async — context manager
async with AsyncLnBot(api_key="key_...") as ln:
    wallet = await ln.wallets.current()
```

## Wallets — `ln.wallets`

```python
# Create (no auth required)
wallet = ln.wallets.create("my-agent")
# → .primary_key, .address, .recovery_passphrase

# Get current
wallet = ln.wallets.current()
# → .available (int, sats)

# Update
ln.wallets.update("new-name")
```

## Invoices — `ln.invoices`

```python
# Create (reference is Python-specific — for correlating with external systems)
invoice = ln.invoices.create(amount=1000, memo="Payment", reference="order-42")
# → .bolt11, .number

# List (cursor-based)
invoices = ln.invoices.list(limit=10, after="cursor")

# Get
invoice = ln.invoices.get("inv_number")

# Watch (sync)
for event in ln.invoices.watch(invoice.number, timeout=60):
    if event.event == "settled":
        break

# Watch (async)
async for event in ln.invoices.watch(invoice.number):
    if event.event == "settled":
        break
```

## Payments — `ln.payments`

```python
# Pay Lightning address
ln.payments.create(target="alice@ln.bot", amount=500)

# Pay BOLT11 invoice
ln.payments.create(target="lnbc1...")

# List
payments = ln.payments.list(limit=10, after="cursor")

# Get
payment = ln.payments.get("pay_number")
```

## Addresses — `ln.addresses`

```python
ln.addresses.create()              # random
ln.addresses.create("alice")       # vanity → alice@ln.bot
addresses = ln.addresses.list()
ln.addresses.delete("alice")
ln.addresses.transfer("alice", target_wallet_key="key_...")
```

## Transactions — `ln.transactions`

```python
txs = ln.transactions.list(limit=50, after="cursor")
```

## Webhooks — `ln.webhooks`

```python
webhook = ln.webhooks.create("https://example.com/hook")
# → .id, .secret (secret shown once). Max 10 per wallet.

webhooks = ln.webhooks.list()
ln.webhooks.delete("webhook_id")
```

## Keys — `ln.keys`

```python
keys = ln.keys.list()              # metadata only
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

## Errors

```python
from lnbot import LnBotError, BadRequestError, UnauthorizedError, NotFoundError, ConflictError

try:
    ln.invoices.get("bad")
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
