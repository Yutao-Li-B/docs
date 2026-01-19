# BRC-20 Token Withdrawal: Spark Batching Analysis

**Date:** January 19, 2026  
**Purpose:** Answer the question: Can Spark batch BRC-20 withdrawals to reduce cost per user?  
**BTC Price Used:** $93,146 USD (January 19, 2026)

---

## Executive Summary

### The Question

Can Spark (or any batching mechanism) settle multiple users' BRC-20 tokens in a single transaction, so we don't pay ~$0.65 per user?

### The Answer

| Approach | Cost Per User | Works Today? | Notes |
|----------|---------------|--------------|-------|
| BRC-20 (individual) | ~$0.65 | ✅ Yes | Each user = 1 transaction |
| BRC-20 (batched) | ~$0.40 | ✅ Yes | 50 users = 1 TX, but still 50 inscriptions |
| BRC-20 Single-Step | ~$0.15-0.30 | ⏳ Coming soon | On testnet, mainnet timeline TBD |
| Absolute minimum | ~$0.31 | — | Bitcoin protocol floor (dust limit) |

**Bottom line:** Spark batching can reduce BRC-20 costs from ~$0.65 to ~$0.40 per user (38% savings). The ~$0.31 dust limit is the absolute floor—this is a Bitcoin protocol rule that cannot be avoided.

---

## Why Does Each User Need a UTXO?

This is the core question. Here's why:

### How Bitcoin Works (vs Ethereum)

| Concept | Ethereum | Bitcoin |
|---------|----------|---------|
| How balances work | Contract stores: `{address: balance}` | No balances—only UTXOs |
| What users "own" | Entry in a contract's database | Physical UTXOs they control |
| To prove ownership | Contract says you own it | You control the private key to a UTXO |
| To transfer | Update the contract's database | Create new UTXOs for recipients |

**Key insight:** Bitcoin doesn't have a concept of "balance." It only knows about UTXOs (Unspent Transaction Outputs)—think of them as digital cash bills.

### Why BRC-20 Requires UTXOs

BRC-20 tokens are built on Bitcoin, so they inherit Bitcoin's model:

```
BRC-20 Token Ownership = Controlling the UTXO that contains the inscribed satoshi

To give someone BRC-20 tokens, you must:
1. Create an inscription that says "transfer X tokens"
2. Put that inscription in a UTXO
3. Send that UTXO to the recipient's address

The recipient now "owns" the tokens because they control the UTXO.
```

### Why Can't We Just Inscribe "Everyone's Balances"?

The question: "Why can't Spark push 1 settlement to mainnet that says User A has 100 tokens, User B has 200 tokens, etc.?"

**Three reasons:**

**1. BRC-20 protocol doesn't support it**

The BRC-20 standard only recognizes these operations:
- `deploy` - Create a new token
- `mint` - Create tokens for yourself  
- `transfer` - Move tokens to ONE recipient

There's no `batch_allocate` or `group_transfer` operation. If we inscribed one, indexers would ignore it.

**2. No way to prove ownership without UTXOs**

On Ethereum, ownership is proven by the contract's internal state. On Bitcoin, ownership is proven by controlling a UTXO's private key.

If we inscribed "User A has 100 tokens" without giving User A a UTXO:
- User A has no private key to prove ownership
- User A cannot transfer the tokens (nothing to sign)
- It's just text on the blockchain with no cryptographic backing

**3. This is a fundamental Bitcoin limitation**

Bitcoin's security model is based on UTXOs, not account balances. Every recipient needs to receive something they can cryptographically control. That something is a UTXO.

### The Dust Limit (~$0.31)

Bitcoin nodes reject UTXOs smaller than 330 satoshis (~$0.31 at current BTC price). This prevents blockchain bloat.

Since each user needs a UTXO, and each UTXO must be at least 330 sats, **~$0.31 per user is the absolute floor for BRC-20.**

---

## How BRC-20 Works: The Three Operations

BRC-20 has three major operations. Each operation is an inscription (witness data) attached to a UTXO on the Bitcoin L1 blockchain.

### Operation 1: Deploy

The token creator inscribes deployment data to their own address:

```
DEPLOY INSCRIPTION (sent to deployer's own address):
{
  "p": "brc-20",
  "op": "deploy",
  "tick": "BID",
  "max": "210000000",
  "lim": "1000"
}
```

- The deployer receives a UTXO with this inscription
- This does NOT give the deployer any tokens—it only registers the token exists
- 5-letter tickers can restrict minting to owner only; 4-letter tickers are open to anyone

**What indexers see:** A transaction where address X received a UTXO with "deploy" inscription. They record: "Token BID was deployed by address X at block Y."

### Operation 2: Mint

A user mints tokens by inscribing a mint operation to their own address:

```
MINT INSCRIPTION (sent to minter's own address):
{
  "p": "brc-20",
  "op": "mint",
  "tick": "BID",
  "amt": "1000"
}
```

- The minter receives a UTXO with this inscription
- Indexers read this and credit the minter's balance
- Multiple mints = multiple additions to balance (like entries in a ledger)

**What indexers see:** A transaction where address X received a UTXO with "mint 1000 BID". They add 1000 to X's balance.

### Operation 3: Transfer

To send tokens, the sender creates a transfer inscription, then sends that UTXO to the recipient:

```
STEP 1: Create transfer inscription (sender sends to themselves first):
{
  "p": "brc-20",
  "op": "transfer",
  "tick": "BID",
  "amt": "500"
}

STEP 2: Send the UTXO containing the transfer inscription to recipient
```

- Two transactions required: inscribe, then send
- Recipient receives the UTXO with the transfer inscription
- Indexers deduct from sender, credit to recipient

**What indexers see:** 
1. Address X created a "transfer 500 BID" inscription
2. Address X sent that UTXO to address Y
3. Deduct 500 from X's balance, add 500 to Y's balance

### How Indexers Calculate Balances

Indexers scan the entire Bitcoin blockchain (which is public and permanent) and maintain a ledger:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BRC-20 INDEXER LEDGER (Example)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Block    TX        Operation           Address        Amount    Balance    │
│  ──────   ──────    ─────────           ───────        ──────    ───────    │
│  800001   abc123    deploy BID          Alice          -         -          │
│  800002   def456    mint BID            Alice          +1000     1000       │
│  800003   ghi789    mint BID            Alice          +1000     2000       │
│  800004   jkl012    mint BID            Bob            +500      500        │
│  800005   mno345    transfer BID        Alice→Bob      -500      Alice:1500 │
│                                                                   Bob:1000  │
│  800006   pqr678    transfer BID        Bob→Carol      -300      Bob:700    │
│                                                                   Carol:300 │
│                                                                              │
│  FINAL BALANCES:                                                            │
│  ├── Alice: 1500 BID                                                        │
│  ├── Bob: 700 BID                                                           │
│  └── Carol: 300 BID                                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

Indexers (Unisat, OKX, BestInSlot) read every transaction, parse the inscriptions, and compute current balances.

### Why No Group Send Exists for BRC-20

Each transfer requires its own inscription. There's no operation for:

```
// THIS DOES NOT EXIST IN BRC-20:
{
  "p": "brc-20",
  "op": "group_transfer",  // ← NOT A VALID OPERATION
  "tick": "BID",
  "recipients": [
    {"addr": "Alice", "amt": "100"},
    {"addr": "Bob", "amt": "200"},
    {"addr": "Carol", "amt": "300"}
  ]
}
```

If we inscribed this, indexers would ignore it because:
- BRC-20 standard only recognizes: `deploy`, `mint`, `transfer`
- Custom operations are not parsed or recognized
- No balance changes would occur

**Result:** To send BRC-20 to 50 users, we need 50 transfer inscriptions = 50 UTXOs = 50 × 330 sats minimum.

---

## Why Can't Spark Lower BRC-20 Costs Further?

Spark operates **off-chain**. It's essentially a database tracking who owns what.

```
SPARK (OFF-CHAIN)                    BITCOIN L1 (ON-CHAIN)
┌─────────────────────┐              ┌─────────────────────┐
│ User A: 1000 BID    │              │ No record of BID    │
│ User B: 500 BID     │  ──────────▶ │ balances. Only      │
│ User C: 750 BID     │  settlement  │ UTXOs exist.        │
│ (just a database)   │              │                     │
└─────────────────────┘              └─────────────────────┘
```

When users want to "withdraw" to L1:
- Spark's database says they own tokens
- But Bitcoin L1 has no record of this
- We must CREATE UTXOs on L1 for each user
- Each UTXO costs at least 330 sats (~$0.31)

**Spark can batch the TRANSACTION (save on fees), but cannot eliminate the need for per-user UTXOs.**

---

## Theoretical Spark → BRC-20 Bridge

A Spark → BRC-20 bridge does not currently exist. But let's explore what it would look like if we built one.

### The Proposed Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SPARK → BRC-20 BRIDGE (50 Users)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  STEP 1: Users request withdrawal on Spark                                  │
│  ─────────────────────────────────────────                                   │
│  User A: "Withdraw 1000 BID to bc1p..."                                     │
│  User B: "Withdraw 500 BID to bc1p..."                                      │
│  User C: "Withdraw 750 BID to bc1p..."                                      │
│  ... (47 more users)                                                        │
│                                                                              │
│  Spark internal ledger deducts balances (free, instant)                     │
│                                                                              │
│  STEP 2: Burn on Spark                                                      │
│  ─────────────────────                                                       │
│  Bridge burns 50 users' Spark-BID tokens                                    │
│  Total burned: 25,000 BID                                                   │
│  Cost: ~$0 (Spark internal operation)                                       │
│                                                                              │
│  STEP 3: Create BRC-20 inscriptions on Bitcoin L1                           │
│  ────────────────────────────────────────────────                            │
│  Bridge must create 50 separate transfer inscriptions:                      │
│                                                                              │
│  Inscription 1: {"op":"transfer","tick":"BID","amt":"1000"} → User A       │
│  Inscription 2: {"op":"transfer","tick":"BID","amt":"500"}  → User B       │
│  Inscription 3: {"op":"transfer","tick":"BID","amt":"750"}  → User C       │
│  ... (47 more inscriptions)                                                 │
│                                                                              │
│  Each inscription requires its own UTXO (minimum 330 sats)                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What the L1 Transaction Looks Like

Even with batching into one transaction, we still need 50 separate UTXOs (one per user):

```
BITCOIN L1 TRANSACTION (Batched 50-user distribution):

INPUTS:
  └── Treasury UTXO: 100,000 sats (covers dust + fees)

OUTPUTS:
  ├── Output 0:  330 sats → User A (with transfer inscription)
  ├── Output 1:  330 sats → User B (with transfer inscription)
  ├── Output 2:  330 sats → User C (with transfer inscription)
  ├── Output 3:  330 sats → User D (with transfer inscription)
  │   ... (46 more outputs)
  ├── Output 49: 330 sats → User 50 (with transfer inscription)
  └── Change:    remaining sats → Treasury

INSCRIPTIONS (in witness data):
  ├── Inscription 0: {"p":"brc-20","op":"transfer","tick":"BID","amt":"1000"}
  ├── Inscription 1: {"p":"brc-20","op":"transfer","tick":"BID","amt":"500"}
  │   ... (48 more inscriptions)
  └── Inscription 49: {"p":"brc-20","op":"transfer","tick":"BID","amt":"200"}
```

### Cost Breakdown (50-User Batch)

| Component | Calculation | Cost |
|-----------|-------------|------|
| Dust limit (50 UTXOs) | 50 × 330 sats | 16,500 sats (~$15.36) |
| Inscription data | 50 × ~100 bytes witness | ~5,000 sats (~$4.66) |
| Transaction overhead | ~500 vbytes × 1 sat/vB | 500 sats (~$0.47) |
| **Total** | | **~22,000 sats (~$20.49)** |
| **Per user** | | **~$0.41** |

*Calculated at $93,146/BTC: 1 sat = $0.00093146*

### Savings from Batching

| Approach | Per-User Cost | Total (50 users) |
|----------|---------------|------------------|
| Individual transactions | ~$0.65 | ~$32.50 |
| Batched (1 transaction) | **~$0.40** | **~$20.00** |
| **Savings** | ~$0.25/user | ~$12.50 (38%) |

### What We Save vs What We Can't Avoid

**Savings from batching:**
- Transaction overhead: Shared across 50 users instead of 50× separate transactions
- Network fees: One transaction instead of 50

**Fixed costs we cannot avoid:**
- Dust limit: 330 sats per user × 50 users = 16,500 sats (~$15.14)
- Individual inscriptions: BRC-20 requires one inscription per recipient

The dust limit alone accounts for ~75% of the batched cost. This is a Bitcoin protocol requirement.

---

## What Batching CAN Do

Batching helps by sharing transaction overhead:

| Without Batching | With Batching (50 users) |
|-----------------|-------------------------|
| 50 separate transactions | 1 transaction with 50 outputs |
| 50× transaction overhead | 1× transaction overhead |
| ~$0.65 per user | ~$0.40 per user |

**Savings: ~38% reduction in cost per user.**

But we still can't get below ~$0.31 because each user needs a UTXO.

---

## Future: BRC-20 Single-Step Transfer

There is a proposal being developed by UniSat called "Single-Step Transfer" that would allow:
- Combining inscribe + send into one step
- Multiple recipients in one transaction (up to 1,000)
- Potentially reducing costs to ~$0.15-0.30 per user

**Status:** Currently on Fractal testnet. Mainnet timeline TBD.

If this becomes available on mainnet, it could further reduce BRC-20 withdrawal costs.

---

## Cost at Scale

### 1 Million Users Withdrawing to L1

| Approach | Per-User Cost | Total Cost | Savings |
|----------|--------------|------------|---------|
| BRC-20 (no batching) | $0.65 | $650,000 | — |
| BRC-20 (batched via Spark) | $0.40 | $400,000 | 38% |
| BRC-20 Single-Step (future) | $0.31 | $310,000 | 52% |

---

## Summary

**Q: Can Spark batch BRC-20 withdrawals to lower costs?**

A: Yes, partially. Spark can batch the L1 transaction, saving ~38% (from $0.65 to $0.40 per user). But each user still needs a UTXO on Bitcoin L1 to represent ownership. The UTXO minimum is ~$0.31 (330 sats)—this is a Bitcoin protocol rule.

**Q: Why can't we inscribe "everyone's balances" in one settlement?**

A: Two reasons:
1. BRC-20 protocol only recognizes `deploy`, `mint`, and `transfer` operations. There's no group transfer operation—indexers would ignore it.
2. Bitcoin requires UTXOs for ownership. Without a UTXO, a user has no cryptographic proof of ownership and cannot transfer their tokens.

**Q: What's the minimum possible cost per user for BRC-20?**

A: ~$0.31 (the dust limit at current BTC price). With current batching, we can achieve ~$0.40. The BRC-20 Single-Step Transfer proposal (coming to mainnet TBD) could potentially get us closer to ~$0.31.

---

## Technical Reference

### Bitcoin UTXO Model

Bitcoin transactions consume UTXOs as inputs and create new UTXOs as outputs. Each UTXO:
- Has a specific satoshi value
- Is locked to an address (only that address's private key can spend it)
- Can only be spent once (then it's "used up")

### Dust Limit

Bitcoin nodes reject transactions that create UTXOs below the dust threshold:
- Taproot (bc1p): 330 satoshis
- Native SegWit (bc1q): 546 satoshis

**Dust Limit Cost Calculation (at $93,146/BTC):**
```
330 sats = 0.00000330 BTC
0.00000330 × $93,146 = $0.31 USD
```

BRC-20 typically uses Taproot addresses, so the 330 sat (~$0.31) limit applies.

---

*Document updated: January 19, 2026*
