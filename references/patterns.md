# Integration Patterns

## 1. Agent receives payment for a task

Create invoice, wait for settlement, then execute the task.

```typescript
import { LnBot } from "@lnbot/sdk";

const ln = new LnBot({ apiKey: "key_..." });

async function chargeForTask(amountSats: number, memo: string) {
  // Create invoice
  const invoice = await ln.invoices.create({ amount: amountSats, memo });

  // Share invoice.bolt11 with the payer

  // Wait for settlement (SSE — no polling)
  for await (const event of ln.invoices.watch(invoice.number)) {
    if (event.event === "settled") {
      break;
    }
    if (event.event === "expired") {
      throw new Error("Invoice expired");
    }
  }

  // Payment confirmed — execute task
  return executeTask();
}
```

```python
from lnbot import LnBot

ln = LnBot(api_key="key_...")

def charge_for_task(amount_sats: int, memo: str):
    invoice = ln.invoices.create(amount=amount_sats, memo=memo)

    # Share invoice.bolt11 with the payer

    for event in ln.invoices.watch(invoice.number):
        if event.event == "settled":
            break
        if event.event == "expired":
            raise Exception("Invoice expired")

    return execute_task()
```

## 2. Agent pays for external services

Pay a BOLT11 invoice from an external service. Use `target` with the invoice string.

```typescript
const ln = new LnBot({ apiKey: "key_..." });

// Pay BOLT11 invoice (amount encoded in invoice)
await ln.payments.create({ target: "lnbc10u1pj..." });

// Pay Lightning address with explicit amount
await ln.payments.create({ target: "service@provider.com", amount: 1000 });
```

```python
ln = LnBot(api_key="key_...")

# Pay BOLT11 invoice
ln.payments.create(target="lnbc10u1pj...")

# Pay Lightning address
ln.payments.create(target="service@provider.com", amount=1000)
```

## 3. Webhook-driven backend

Register a webhook, then handle incoming POST notifications.

```typescript
import { LnBot } from "@lnbot/sdk";

const ln = new LnBot({ apiKey: "key_..." });

// Register webhook (max 10 per wallet)
const webhook = await ln.webhooks.create({ url: "https://example.com/hooks/lnbot" });
// Store webhook.secret for signature verification

// In your HTTP handler:
app.post("/hooks/lnbot", (req, res) => {
  // Verify webhook signature using webhook.secret
  const payload = req.body;
  // Process the event
  res.sendStatus(200);
});
```

```python
ln = LnBot(api_key="key_...")

webhook = ln.webhooks.create("https://example.com/hooks/lnbot")
# Store webhook.secret for signature verification
```

## 4. Multi-agent wallet management

A coordinator creates wallets and distributes funds to worker agents.

```typescript
import { LnBot } from "@lnbot/sdk";

// Coordinator creates wallets for agents
const unauthenticated = new LnBot();
const agent1 = await unauthenticated.wallets.create({ name: "agent-1" });
const agent2 = await unauthenticated.wallets.create({ name: "agent-2" });

// Store keys securely: agent1.primaryKey, agent1.recoveryPassphrase
// Store keys securely: agent2.primaryKey, agent2.recoveryPassphrase

// Fund agents from coordinator wallet
const coordinator = new LnBot({ apiKey: "key_coordinator..." });
await coordinator.payments.create({ target: agent1.address, amount: 10000 });
await coordinator.payments.create({ target: agent2.address, amount: 10000 });

// Each agent operates independently
const ln1 = new LnBot({ apiKey: agent1.primaryKey });
const balance = await ln1.wallets.current();
```

## 5. Zero-downtime key rotation

Rotate secondary key first, update all clients, then rotate primary.

```typescript
const ln = new LnBot({ apiKey: currentPrimaryKey });

// Step 1: Rotate secondary key (index 1)
const newSecondary = await ln.keys.rotate(1);
// newSecondary.key — deploy to backup/failover systems

// Step 2: Verify new secondary works
const test = new LnBot({ apiKey: newSecondary.key });
await test.wallets.current(); // should succeed

// Step 3: Rotate primary key (index 0)
const newPrimary = await ln.keys.rotate(0);
// newPrimary.key — deploy to all primary systems

// Step 4: Update main client
const updated = new LnBot({ apiKey: newPrimary.key });
```

## 6. MCP config for LLM agents

Generate and use MCP config:

```bash
# Generate local config
lnbot mcp config

# Generate remote config
lnbot mcp config --remote
```

Paste into your AI agent's MCP settings file. The MCP server exposes ln.bot operations as tools the agent can call.

## 7. Wallet recovery

Restore access using the 12-word BIP-39 passphrase:

```typescript
const ln = new LnBot(); // no auth needed
const restored = await ln.restore.recovery({
  passphrase: "word1 word2 word3 word4 word5 word6 word7 word8 word9 word10 word11 word12"
});
// restored.primaryKey, restored.secondaryKey — new keys issued
```

```python
ln = LnBot()  # no auth needed
wallet = ln.restore.recovery("word1 word2 ... word12")
# wallet.primary_key — new key issued
```

```bash
lnbot restore
# Follow prompts to enter passphrase
```

---

## Anti-patterns

**Don't poll for invoice status.** Use `watch()` / SSE streaming instead. Polling wastes resources and adds latency.

```typescript
// BAD — polling
while (true) {
  const inv = await ln.invoices.get(number);
  if (inv.status === "settled") break;
  await sleep(1000);
}

// GOOD — SSE streaming
for await (const event of ln.invoices.watch(number)) {
  if (event.event === "settled") break;
}
```

**Don't store recovery passphrases in code.** Use environment variables or a secrets manager.

```python
# BAD
ln.restore.recovery("abandon abandon abandon ...")

# GOOD
import os
ln.restore.recovery(os.environ["WALLET_RECOVERY_PASSPHRASE"])
```

**Don't skip idempotency in automated payments.** Without unique identifiers, retries may cause duplicate payments.

**Don't hardcode amounts.** Accept amounts as parameters or configuration to avoid accidental overpayment.

**Don't ignore errors.** All SDKs provide typed errors — handle `NotFoundError`, `ConflictError`, etc. explicitly in production code.

**Don't create one wallet per request.** Wallets are identities, not sessions. Reuse wallets across requests.
