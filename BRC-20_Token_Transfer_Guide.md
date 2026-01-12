# BRC-20 Tokens & BTC Transactions Guide

> A comprehensive guide to understanding Bitcoin UTXOs, Ordinals inscriptions, BRC-20 tokens, transfer costs, Layer 2 solutions, and strategies for delivering BRC-20 tokens to users' Taproot wallets.

---

# 📋 Executive Takeaway (Start Here)

## First Question: What Is The Goal?

**If the goal is**: User has REAL BRC-20 tokens visible on Unisat/blockchain explorers

**Answer**: **~$0.65 per user** — there is no cheaper way.

---

## Why ~$0.65? (The Dollar Bill Analogy)

Think of BRC-20 tokens like **writing on a dollar bill**:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ┌─────────────┐                                               │
│   │   $1 BILL   │  ← You write "100 BID tokens" on this bill   │
│   │  ┌───────┐  │                                               │
│   │  │100 BID│  │  The writing is INSEPARABLE from the bill    │
│   │  └───────┘  │                                               │
│   └─────────────┘                                               │
│                                                                 │
│   To give someone the tokens:                                   │
│   → You must give them the PHYSICAL BILL                        │
│   → The bill itself costs ~$0.30 (minimum Bitcoin requires)    │
│   → Delivery/postage costs ~$0.35 (transaction fees)           │
│   → TOTAL: ~$0.65 per transfer                                 │
│                                                                 │
│   This is the cost of writing to the Bitcoin blockchain.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## What About Cheaper Options?

| Option | Transfer Cost | But... |
|--------|---------------|--------|
| **Shadow Balance** | $0 | ❌ NOT on Bitcoin. NOT visible on Unisat. Just a database entry. |
| **Merlin L2** | ~$0.05 | ❌ NOT real BRC-20. It's a "wrapped" token on a separate chain. NOT visible on Unisat. |
| **Real BRC-20 (L1)** | **~$0.65** | ✅ ON Bitcoin. Visible on Unisat. User actually owns it. |

### ⚠️ Important: Merlin L2 is NOT a shortcut

Merlin L2 at $0.05 sounds cheap, but:
- User gets a **wrapped token (M-Token)**, not real BRC-20
- **NOT visible** on Unisat or any Bitcoin explorer  
- To convert to real BRC-20: user must **bridge out** → costs **~$0.65**
- **Total if using Merlin**: $0.05 (L2) + $0.65 (bridge out) = **$0.99** (more expensive!)

**Merlin L2 only saves money if users NEVER need real L1 BRC-20.**

### What is a "Wrapped Token"?

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   REAL BRC-20 (on Bitcoin L1):                                  │
│   ┌─────────────┐                                               │
│   │ Actual $1   │  ← The real bill with "100 BID" written on it│
│   │ bill with   │  ← Exists on Bitcoin blockchain              │
│   │ writing     │  ← Visible on Unisat                         │
│   └─────────────┘                                               │
│                                                                 │
│   WRAPPED TOKEN (on Merlin L2):                                 │
│   ┌─────────────┐                                               │
│   │   Receipt   │  ← A receipt that SAYS you have 100 BID      │
│   │ "IOU 100    │  ← The real bill is locked in a vault        │
│   │    BID"     │  ← Visible on Merlin explorer (not Unisat)   │
│   └─────────────┘  ← NOT visible on Bitcoin blockchain         │
│                                                                 │
│   To get the real bill back:                                    │
│   → Return the receipt (burn wrapped token)                     │
│   → Pay ~$0.65 to retrieve from vault (bridge out to L1)       │
│   → Then you have real BRC-20 on Bitcoin                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Wrapped tokens on Merlin:**
- ✅ Visible on **Merlin's block explorer** (merlin.xyz)
- ✅ Tradeable on **Merlin DEXs**
- ❌ NOT visible on **Unisat** or Bitcoin L1 explorers
- ❌ NOT the real BRC-20 token on Bitcoin

**In short**: A wrapped token is like an IOU or receipt. It's visible and tradeable within Merlin's system, but it's not the real BRC-20 on Bitcoin L1.

---

## The Bottom Line

| Goal | Cost per User | Notes |
|------|---------------|-------|
| Real BRC-20 on Bitcoin (visible on Unisat) | **~$0.65** | No way to reduce this |
| Internal tracking only (Shadow Balance) | $0 | Not on blockchain, not verifiable |

**At scale:**

| Users | Cost for Real L1 BRC-20 |
|-------|------------------------|
| 1,000 | ~$650 |
| 100,000 | ~$65,000 |
| 1,000,000 | **~$650,000** |

---

## Why Bitcoin Transactions Cost Money

1. **Miners must be paid** - They process transactions
2. **Block space is limited** - Only ~4 MB every 10 minutes  
3. **No free transactions** - Even at minimum fees, costs exist

**Cost breakdown at 1 sat/vbyte via UniSat API:**
- Inscription transaction: ~$0.17 (182 sats)
- Send transaction: ~$0.18 (~200 sats)
- Dust limit (minimum UTXO): ~$0.30 (330 sats) ← **Fixed, cannot reduce**
- **Total: ~$0.65** (712 sats)

---

## Inscription Service: Quick Comparison

| Provider | API? | Service Fee | Total Cost |
|----------|------|-------------|------------|
| **UniSat API** | ✅ Yes | **FREE** | **~$0.65** ⭐ |
| InscribeNFTs | ❌ UI only | 1,465 sats | ~$1.83 |
| Xverse | ⚠️ In dev | 2,000 sats | ~$2.43 |
| OrdinalsBot | ⚠️ Runes only | 2,000 sats | ~$2.50 |
| Gamma | ❌ UI only | 2,500 sats | ~$3.05 |
| UniSat UI | - | 3,150 sats | ~$3.54 |

**Recommendation: UniSat API** — only production BRC-20 API with FREE service fee.

> ⚠️ **Self-hosting note**: Even if we built our own inscription infrastructure (running Bitcoin full node + ord CLI), **network fees are unavoidable** — miners must still be paid. Self-hosting would save on service fees but not network fees. We do not currently have self-hosted inscription infrastructure.

*For detailed provider comparison, see [BRC-20 Inscription Service Providers (Detailed)](#brc-20-inscription-service-providers-detailed) below.*

---

*For detailed technical explanations, see the full guide below.*

---

## Executive Summary: Quick Cost Reference

### Key Facts

| Fact | Value |
|------|-------|
| **BTC Price** | ~$91,756 USD (January 2026) |
| **1 BTC** | 100,000,000 satoshis (sats) |
| **Our fee rate** | 1 sat/vbyte (minimum rate) |
| **BRC-20 transfer cost (L1)** | **~$0.65 per transfer** |
| **Highest cost component** | Dust limit (~$0.30, or 53% of total) |

### Cost at Scale (@ 1 sat/vbyte)

| Users | Cost per Transfer | Total Cost (L1) |
|-------|-------------------|-----------------|
| 1,000 | ~$0.65 | ~$650 |
| 100,000 | ~$0.65 | ~$65,000 |
| 1,000,000 | ~$0.65 | ~**$650,000** |

### Why Bitcoin Transactions Cost Money

- **Miners must be paid** to include transactions in blocks
- **Block space is limited** (~4 MB per block, ~10 min per block)
- **Fee market** means higher fees = faster confirmation
- **No fee = transaction not processed** (stays in mempool indefinitely)

*See Section 5 for detailed explanation of how Bitcoin L1 works.*

---

## ⚠️ Critical Concept: L1 vs Off-Chain (Shadow Balance / L2)

```
┌─────────────────────────────────────────────────────────────────┐
│           THE FUNDAMENTAL COST DIFFERENCE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   L1 (On-Chain):                                                │
│   └── Every transfer = ~$0.65 (@ 1 sat/vB)                     │
│   └── Data written to Bitcoin blockchain                        │
│   └── Verifiable on Unisat, OrdScan, any L1 explorer           │
│   └── User owns REAL BRC-20 token                              │
│                                                                 │
│   Shadow Balance / L2 (Off-Chain):                              │
│   └── Transfers = ~$0 to ~$0.05 (cheap!)                       │
│   └── Data stored in database or L2 chain                       │
│   └── NOT visible on Bitcoin L1 blockchain                      │
│   └── Balance CANNOT be verified on L1 explorers               │
│   └── User has representation, NOT real L1 BRC-20              │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   🔴 TO GET REAL BRC-20 ON L1 (verifiable on mainnet):         │
│                                                                 │
│      User MUST do an on-chain operation:                        │
│      • Inscribe transfer inscription                            │
│      • Send inscribed UTXO to user's wallet                     │
│      • Cost: ~$0.65 per user (@ 1 sat/vB)                      │
│                                                                 │
│      This applies whether coming from:                          │
│      • Shadow Balance → L1 redemption                           │
│      • Merlin L2 → Bridge out to L1                            │
│      • Direct L1 transfer                                       │
│                                                                 │
│   There is NO way to avoid this cost if the goal is:           │
│   "Real BRC-20 in user's Taproot wallet, visible on Unisat"    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| Approach | Transfer Cost | Visible on L1? | Real BRC-20? | To Get Real L1 BRC-20 |
|----------|---------------|----------------|--------------|----------------------|
| **L1 Direct** | ~$0.65 | ✅ Yes | ✅ Yes | Already on L1 |
| **Shadow Balance** | $0 | ❌ No | ❌ No (database) | Redeem: +$0.65 |
| **Merlin L2** | ~$0.05 | ❌ No (wrapped) | ❌ No (M-Token) | Bridge out: +$0.65 |
| **Spark (LRC-20)** | ~$0.01-0.05 | ❌ No | ❌ No (different token) | ❌ Cannot convert |

**Bottom Line**: The ~$0.65 L1 inscription cost is unavoidable if you want real, verifiable BRC-20 tokens on the Bitcoin mainnet.

---

## Table of Contents

1. [Bitcoin UTXOs Explained](#1-bitcoin-utxos-explained)
2. [Ordinals Protocol & Inscriptions](#2-ordinals-protocol--inscriptions)
3. [BRC-20 Token Standard](#3-brc-20-token-standard)
4. [Address Types & Dust Limits](#4-address-types--dust-limits)
5. [How Bitcoin L1 Works (On-Chain Mechanics)](#5-how-bitcoin-l1-works-on-chain-mechanics)
6. [Cost of BRC-20 Token Transfers (L1)](#6-cost-of-brc-20-token-transfers-l1)
7. [Bitcoin Layer 2 Solutions for BRC-20](#7-bitcoin-layer-2-solutions-for-brc-20)
8. [How Merlin Chain L2 Works (On-Chain vs Off-Chain)](#8-how-merlin-chain-l2-works-on-chain-vs-off-chain)
9. [L2 Withdrawal Costs (Bridge to L1 BRC-20)](#9-l2-withdrawal-costs-bridge-to-l1-brc-20)
10. [Comparison: Shadow Balance vs L2 vs L1](#10-comparison-shadow-balance-vs-l2-vs-l1)
11. [Alternative Token Standards on Bitcoin Blockchain](#11-alternative-token-standards-on-bitcoin-blockchain)
    - [11.1 Bitcoin L1 Token Standards Comparison](#111-bitcoin-l1-token-standards-comparison)
    - [11.2 Detailed Token Standard Explanations](#112-detailed-token-standard-explanations)
    - [11.3 L2/Sidechain Token Options](#113-l2sidechain-token-options)
    - [11.4 Recommendation Matrix](#114-recommendation-matrix)
    - [11.5 Cost Comparison Summary](#115-cost-comparison-summary)
12. [Single Operation Cost Breakdown](#12-single-operation-cost-breakdown)
13. [Mass Distribution Cost Analysis](#13-mass-distribution-cost-analysis)
14. [Recommendations for BRC-20 to Taproot Wallets](#14-recommendations-for-brc-20-to-taproot-wallets)
15. [BRC-20 Inscription Service Providers (Detailed)](#brc-20-inscription-service-providers-detailed)

---

## 1. Bitcoin UTXOs Explained

### What is a UTXO?

**UTXO** stands for **Unspent Transaction Output**. 

**Think of it like a physical wallet with cash bills:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE WALLET ANALOGY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   REAL WALLET:                    BITCOIN WALLET:               │
│   ┌─────────────────┐            ┌─────────────────┐           │
│   │ $20 bill        │            │ UTXO: 20,000 sats│           │
│   │ $10 bill        │     =      │ UTXO: 10,000 sats│           │
│   │ $5 bill         │            │ UTXO: 5,000 sats │           │
│   │ $5 bill         │            │ UTXO: 5,000 sats │           │
│   ├─────────────────┤            ├─────────────────┤           │
│   │ Total: $40      │            │ Total: 40,000 sats│          │
│   └─────────────────┘            └─────────────────┘           │
│                                                                 │
│   • Your wallet = Your Bitcoin address                         │
│   • Each bill = Each UTXO                                      │
│   • Bill value = Satoshis in the UTXO                          │
│   • Total cash = Sum of all UTXOs                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key Facts

- **1 BTC = 100,000,000 satoshis (sats)**
- Each UTXO has a specific sat value
- UTXOs are created when you receive Bitcoin
- UTXOs are marked as "spent" when you send Bitcoin
- Any leftover amount creates a new "change" UTXO

### How UTXOs Work (The Bill Analogy)

| Real World (Bills) | Bitcoin (UTXOs) |
|-------------------|-----------------|
| You receive a $20 bill | A new UTXO is created with 20,000 sats |
| The $20 bill sits in your wallet | The UTXO exists on the blockchain, spendable by you |
| You hand the $20 bill to someone | You "spend" the UTXO (mark it as used) |
| The store gives you change | A new "change" UTXO is created for you |
| Your original $20 bill is now with the store | Your original UTXO is marked "spent" (can never be used again) |

**Important clarification:** UTXOs are NOT destroyed. They are **marked as "spent"** on the blockchain. The data still exists forever, but a spent UTXO can never be used again—just like once you hand over a $20 bill, you can't spend that same bill again.

### Transaction Example (Paying with Bills)

```
SCENARIO: You want to buy something for $12

YOUR WALLET BEFORE:
  └── One $20 bill (1 UTXO worth 20,000 sats)

WHAT HAPPENS:
  1. You hand over your $20 bill (UTXO marked as "spent")
  2. Store receives $12 (new UTXO created: 12,000 sats → store's address)
  3. You receive $8 change (new UTXO created: 8,000 sats → your address)
  
  (In Bitcoin, some also goes to miners as a fee)

YOUR WALLET AFTER:
  └── One $8 bill (new change UTXO)

STORE'S WALLET AFTER:
  └── One $12 payment (new UTXO)

YOUR ORIGINAL $20 BILL:
  └── Still exists in history, but marked as "spent"
  └── You can NEVER spend it again
  └── It's now an "STO" (Spent Transaction Output), not a "UTXO"
```

### Multi-Input Transaction Example (Combining Bills)

```
SCENARIO: You want to send $35 to a friend

YOUR WALLET:
  ├── $20 bill (UTXO 1)
  ├── $10 bill (UTXO 2)
  └── $5 bill  (UTXO 3)
  
TOTAL: $35

WHAT HAPPENS:
  1. You combine all three bills ($35 total)
  2. All three UTXOs are marked as "spent"
  3. One new UTXO created: $35 → friend's address
     (minus fee for the transaction)

AFTER:
  Your wallet: Empty (all UTXOs marked "spent")
  Friend's wallet: One new $35 UTXO
  
Your original bills: Still exist in blockchain history,
                     but permanently marked as "spent"
```

### Why This Matters for BRC-20 Tokens

BRC-20 tokens use inscriptions that are **attached to specific satoshis within UTXOs**. Think of it like writing on a specific bill:

```
ANALOGY: BRC-20 = Writing on a Dollar Bill

1. You have a $1 bill (UTXO with 330+ sats)

2. You write "100 BID tokens" on that $1 bill
   (Inscription attached to a specific satoshi in the UTXO)

3. To transfer those 100 BID tokens to someone:
   └── You must give them the $1 bill with the writing on it
   └── The writing travels WITH the bill
   └── You cannot separate the writing from the bill

4. Once they have the bill:
   └── They own the 100 BID tokens
   └── Your original UTXO is marked "spent"
   └── They have a new UTXO with the inscription attached
```

**This is why every BRC-20 transfer requires:**
- A UTXO with at least the **dust limit** (330 sats) = the minimum "bill size" recognized by the network
- You need a valid UTXO (bill) to hold the inscribed satoshi (the writing)

---

## 2. Ordinals Protocol & Inscriptions

### What is the Ordinals Protocol?

The Ordinals protocol assigns a **unique serial number** to every satoshi, allowing individual sats to be tracked and identified. This enables "inscribing" data onto specific satoshis.

```
┌─────────────────────────────────────────────────────────────────┐
│                ORDINALS: SATOSHI NUMBERING SYSTEM               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Bitcoin's 21 million BTC = 2,100,000,000,000,000 satoshis    │
│                                                                 │
│   Ordinals assigns each satoshi a unique number:                │
│     Satoshi #0           → First sat ever mined (Block 0)      │
│     Satoshi #1           → Second sat                          │
│     Satoshi #2,100,000,000,000,000 → Last sat (year ~2140)    │
│                                                                 │
│   Like serial numbers on dollar bills, but for every satoshi   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### What is an Inscription?

An inscription is data that is:
1. **Physically stored** in the witness field of a Bitcoin transaction
2. **Logically attached** to a specific satoshi via the Ordinals numbering system

| Aspect | Description |
|--------|-------------|
| **What can be inscribed** | Images, text, JSON, any data |
| **Where data is stored** | In the **witness field** of the transaction |
| **What it's attached to** | A **specific satoshi** in the output UTXO (via Ordinals) |
| **How to transfer** | Spend the UTXO containing the inscribed sat—the inscription moves with that sat |
| **Value sources** | Both the sat value AND the inscription's perceived value |

### The $100 Bill Analogy for Inscriptions

```
┌─────────────────────────────────────────────────────────────────┐
│             INSCRIPTION = WRITING ON A DOLLAR BILL              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   BEFORE INSCRIPTION:                                           │
│   ┌─────────────────┐                                           │
│   │     $100        │    Value = $100 (face value only)        │
│   │                 │                                           │
│   └─────────────────┘                                           │
│                                                                 │
│   AFTER INSCRIPTION (write "Mona Lisa" on it):                 │
│   ┌─────────────────┐                                           │
│   │     $100        │    Face value = $100                     │
│   │   ┌─────────┐   │    Inscription value = $0 to $millions   │
│   │   │Mona Lisa│   │    (market decides what it's worth)      │
│   │   └─────────┘   │                                           │
│   └─────────────────┘    Total = Face value + Inscription value│
│                                                                 │
│   KEY INSIGHTS:                                                 │
│   • If you spend it as a regular $100, you LOSE the inscription│
│   • The inscription is INSEPARABLE from the physical bill      │
│   • You can only transfer both together                        │
│   • The "writing" lives in the witness field, but is           │
│     LOGICALLY attached to a specific sat (via Ordinals)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### How Inscriptions Are Stored (Technical Detail)

```
┌─────────────────────────────────────────────────────────────────┐
│                WHERE INSCRIPTION DATA LIVES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   PHYSICAL STORAGE (Witness Field):                             │
│   └── The actual data (image, JSON, text) is stored in the     │
│       witness field of the Bitcoin transaction                  │
│   └── This is on-chain, permanent, and publicly visible         │
│                                                                 │
│   LOGICAL ATTACHMENT (Ordinals Protocol):                       │
│   └── Ordinals assigns a unique number to every satoshi         │
│   └── The inscription is "attached" to a specific sat number   │
│   └── When that sat moves (UTXO is spent), the inscription     │
│       follows that specific satoshi to its new UTXO             │
│                                                                 │
│   THINK OF IT LIKE:                                             │
│   └── Physical storage = The ink that writes on the bill       │
│   └── Logical attachment = Which specific bill has the writing │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Inscribed vs Uninscribed UTXOs

| Type | Contains | Value Source | Example |
|------|----------|--------------|---------|
| **Uninscribed UTXO** | Just sats | Sat value only | 10,000 sats = ~$9.18 |
| **Inscribed UTXO** | Sats + witness data | Sat value + inscription value | 330 sats + BRC-20 inscription |

### Why Inscriptions Must Travel with UTXOs

```
SCENARIO: You have an inscribed UTXO

┌────────────────────┐
│ UTXO: 10,000 sats  │
│ Inscription: 🎨    │  ← Inscription attached to satoshi #12345
└────────────────────┘

IF YOU SPEND THIS UTXO:
  Option A: Send to someone who understands inscriptions
    └── They receive the UTXO with the inscription
    └── Inscription value preserved ✅

  Option B: Send to a wallet that doesn't track inscriptions
    └── They receive the sats
    └── Inscription "disappears" from their view ⚠️
    └── Data still exists on-chain, just not tracked

  Option C: Spend as regular Bitcoin (e.g., pay for coffee)
    └── UTXO gets consumed in regular transaction
    └── Inscription may get sent to wrong recipient
    └── Inscription value LOST ❌
```

---

## 3. BRC-20 Token Standard

### What is BRC-20?

BRC-20 is an **experimental token standard** on Bitcoin that uses inscriptions to create **fungible tokens**. Unlike Ethereum's ERC-20, BRC-20 doesn't use smart contracts—it uses JSON inscriptions.

```
┌─────────────────────────────────────────────────────────────────┐
│                    BRC-20 vs ERC-20                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ERC-20 (Ethereum):                                            │
│   └── Uses smart contracts                                      │
│   └── Balances stored in contract state                         │
│   └── Single transaction to transfer                            │
│   └── Natively supported by Ethereum                            │
│                                                                 │
│   BRC-20 (Bitcoin):                                             │
│   └── Uses JSON inscriptions (text on satoshis)                 │
│   └── Balances computed by off-chain indexers                   │
│   └── Two transactions to transfer (inscribe + send)            │
│   └── NOT native to Bitcoin—relies on Ordinals protocol         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### How BRC-20 Works

BRC-20 tokens are created and managed through three operations:

| Operation | Purpose | Example JSON | What Happens |
|-----------|---------|--------------|--------------|
| **Deploy** | Create a new token | `{"p":"brc-20","op":"deploy","tick":"BID","max":"210000000","lim":"1000"}` | Establishes token rules |
| **Mint** | Create new tokens | `{"p":"brc-20","op":"mint","tick":"BID","amt":"1000"}` | Assigns tokens to minter |
| **Transfer** | Send tokens | `{"p":"brc-20","op":"transfer","tick":"BID","amt":"100"}` | Moves tokens between wallets |

### The $1 Bill Analogy for BRC-20

```
┌─────────────────────────────────────────────────────────────────┐
│                    BRC-20 TRANSFER PROCESS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   STEP 1: DEPLOY (One-time token creation)                     │
│   ┌─────────────────┐                                           │
│   │ Deploy: BID     │  "I'm creating a token called BID"       │
│   │ Max: 210M       │  "Maximum supply is 210 million"          │
│   │ Limit: 1000     │  "Each mint can create up to 1000"       │
│   └─────────────────┘                                           │
│                                                                 │
│   STEP 2: MINT (Create tokens for yourself)                    │
│   ┌─────────────────┐                                           │
│   │     $1 bill     │  You take a $1 bill (330+ sats UTXO)     │
│   │ ┌─────────────┐ │                                           │
│   │ │ Mint: 1000  │ │  You write "Mint 1000 BID" on it         │
│   │ │    BID      │ │                                           │
│   │ └─────────────┘ │  You now have 1000 BID tokens!           │
│   └─────────────────┘                                           │
│                                                                 │
│   STEP 3: TRANSFER (Two-step process!)                         │
│                                                                 │
│   Transaction 1: INSCRIBE TRANSFER                             │
│   ┌─────────────────┐                                           │
│   │    NEW $1 bill  │  Take a NEW $1 bill (new UTXO)           │
│   │ ┌─────────────┐ │                                           │
│   │ │Transfer 100 │ │  Write "Transfer 100 BID" on it          │
│   │ │    BID      │ │                                           │
│   │ └─────────────┘ │  This is still in YOUR wallet            │
│   └─────────────────┘                                           │
│                                                                 │
│   Transaction 2: SEND UTXO                                     │
│   ┌─────────────────┐                                           │
│   │    $1 bill ────────────────────▶ RECIPIENT                 │
│   │ with inscription│  Give the $1 bill to recipient           │
│   └─────────────────┘                                           │
│                                                                 │
│   RESULT:                                                       │
│   ├── You: 900 BID remaining (original mint inscription)       │
│   ├── Recipient: 100 BID (new transfer inscription)            │
│   └── Cost: Two transaction fees + dust limit                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Critical Insight: The Inseparable Value Problem

```
┌─────────────────────────────────────────────────────────────────┐
│              THE INSEPARABLE VALUE PROBLEM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   You have a $1 bill inscribed with "10 BID tokens"            │
│   ┌─────────────────┐                                           │
│   │     $1 bill     │                                           │
│   │ ┌─────────────┐ │                                           │
│   │ │  10 BID     │ │                                           │
│   │ └─────────────┘ │                                           │
│   └─────────────────┘                                           │
│                                                                 │
│   OPTION 1: Spend as a $1 bill (buy coffee)                    │
│     └── You get $1 worth of coffee                             │
│     └── You LOSE the 10 BID tokens FOREVER ❌                  │
│     └── The inscription goes to the coffee shop                │
│     └── (They probably don't know they have BID tokens)        │
│                                                                 │
│   OPTION 2: Keep as BID token holder                           │
│     └── You keep the 10 BID value                              │
│     └── You CANNOT use the $1 separately                       │
│     └── The $1 is "locked" as the carrier for BID ✅           │
│                                                                 │
│   THE BOTTOM LINE:                                              │
│   The $1 (330 sats) and the 10 BID are INSEPARABLE.            │
│   This is why the dust limit is part of the "cost" of a token. │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### How Balances are Tracked

BRC-20 token balances are **NOT stored on-chain directly**. Instead:

```
┌─────────────────────────────────────────────────────────────────┐
│                HOW BRC-20 BALANCES WORK                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   BITCOIN BLOCKCHAIN (On-chain):                                │
│   ├── Block 800001: Deploy BID (max: 210M)                     │
│   ├── Block 800002: Mint 1000 BID → Address A                  │
│   ├── Block 800003: Transfer 100 BID (inscribed)               │
│   ├── Block 800004: UTXO sent → Address B                      │
│   └── ... millions of inscriptions ...                         │
│                                                                 │
│   INDEXER (Off-chain computation):                              │
│   ├── Reads ALL blocks from genesis                            │
│   ├── Parses ALL BRC-20 inscriptions                           │
│   ├── Tracks UTXO movements                                    │
│   └── Computes current balances                                │
│                                                                 │
│   RESULT:                                                       │
│   ├── Address A: 900 BID                                       │
│   ├── Address B: 100 BID                                       │
│   └── These balances are COMPUTED, not stored                  │
│                                                                 │
│   INDEXERS: Unisat, OrdScan, OKX, etc.                         │
│   (Different indexers should agree on balances)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

1. **Indexers/Scanners** read the entire blockchain
2. They parse all deploy/mint/transfer inscriptions
3. They calculate who owns what based on inscription + UTXO history
4. Your "balance" is a **computed value**, not stored data

---

## 4. Address Types & Dust Limits

### Bitcoin Address Type Evolution

| Type | Era | Prefix | Security | Efficiency | Features |
|------|-----|--------|----------|------------|----------|
| **Legacy (P2PKH)** | Original | 1... | Basic | Poor | Basic transactions |
| **SegWit (P2SH)** | 2017 | 3... | Better | Good | Backward compatible |
| **Native SegWit (P2WPKH)** | 2017 | bc1q... | Better | Better | More efficient |
| **Taproot (P2TR)** | 2021 | bc1p... | Best | Best | Enables Ordinals, BRC-20 |

### Why Taproot for BRC-20?

```
┌─────────────────────────────────────────────────────────────────┐
│                    TAPROOT ADVANTAGES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ✅ Lower dust limit (330 sats vs 330 sats)                    │
│   ✅ More efficient witness data storage                        │
│   ✅ Native support for complex scripts (Ordinals)              │
│   ✅ Lower transaction fees overall                             │
│   ✅ Required by most BRC-20 wallets (Unisat, OKX, Xverse)      │
│                                                                 │
│   Taproot addresses start with: bc1p...                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Dust Limits Explained

The **dust limit** is the minimum satoshi amount for a UTXO to be recognized by the network. UTXOs below this limit are considered "dust" and rejected.

| Address Type | Dust Limit | USD Value (@ $91,756/BTC) |
|--------------|------------|---------------------------|
| Legacy (P2PKH) | 330 sats | ~$0.30 |
| Native SegWit (P2WPKH) | 330 sats | ~$0.30 |
| Taproot (P2TR) | 330 sats | ~$0.30 |

### The Dust Limit Analogy

```
┌─────────────────────────────────────────────────────────────────┐
│              DUST LIMIT = MINIMUM BILL SIZE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   REAL WORLD ANALOGY:                                           │
│   └── Imagine if the government said: "Bills under $0.50       │
│       are too small to be worth handling"                       │
│   └── You couldn't create or transfer $0.25 bills              │
│   └── The minimum "bill" would be $0.50                        │
│                                                                 │
│   BITCOIN'S DUST LIMIT:                                         │
│   └── UTXOs smaller than 330 sats are too small to be worth    │
│       the transaction fee to spend them                         │
│   └── The network rejects these as "dust"                      │
│   └── This prevents blockchain bloat from tiny UTXOs           │
│                                                                 │
│   FOR BRC-20:                                                   │
│   └── Every BRC-20 inscription needs a UTXO to live on         │
│   └── That UTXO must have at least the dust limit in sats      │
│   └── This is the "paper cost" of the token                    │
│                                                                 │
│   MINIMUM BRC-20 UTXO:                                          │
│   ├── Native SegWit: 330 sats (~$0.30)                         │
│   └── Taproot: 330 sats (~$0.30)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why Dust Limits Can't Be Avoided

```
BRC-20 Inscription Requirements:
  ├── Minimum sats: 546 (Native SegWit) or 330 (Taproot)
  ├── This is the "paper cost" of the $1 bill in our analogy
  ├── You CANNOT inscribe on less than dust limit
  └── Even at 1 sat/vbyte fees, dust is still ~50% of total cost

AT 1 SAT/VBYTE:
  ├── Transaction fees: ~$0.44 (can be reduced with lower rates)
  └── Dust limit: ~$0.30 (FIXED, cannot be reduced) ← Largest cost!
```

---

## 5. How Bitcoin L1 Works (On-Chain Mechanics)

### What Gets Written to Bitcoin L1 Blockchain

| Data Type | Where Stored | Size | Permanent? |
|-----------|--------------|------|------------|
| **Transaction inputs** | Transaction data | ~41 bytes each | Yes |
| **Transaction outputs** | Transaction data | ~34 bytes each | Yes |
| **Witness data** | Segregated witness | Variable | Yes |
| **BRC-20 JSON** | Witness data (inscription) | ~80-200 bytes | Yes |
| **Signatures** | Witness data | ~64 bytes | Yes |

### BRC-20 Inscription: What's Actually On-Chain

```
┌─────────────────────────────────────────────────────────────────┐
│                    BITCOIN TRANSACTION                          │
├─────────────────────────────────────────────────────────────────┤
│ INPUTS:                                                         │
│   └── Previous UTXO reference (txid:vout)                      │
│                                                                 │
│ OUTPUTS:                                                        │
│   └── New UTXO: 330 sats → recipient's Taproot address         │
│       └── Contains satoshis, one of which "carries" the        │
│           inscription (tracked by Ordinals protocol)            │
│                                                                 │
│ WITNESS DATA (Taproot script) - WHERE INSCRIPTION DATA LIVES:  │
│   └── OP_FALSE OP_IF                                           │
│       └── "ord" (ordinals marker)                              │
│       └── content-type: "text/plain;charset=utf-8"             │
│       └── {"p":"brc-20","op":"transfer","tick":"BID","amt":"100"}│
│       OP_ENDIF                                                  │
│   └── Schnorr signature                                        │
│                                                                 │
│ ORDINALS PROTOCOL (Off-chain indexing):                        │
│   └── Tracks which satoshi the inscription is attached to      │
│   └── When UTXO is spent, inscription follows that specific sat│
└─────────────────────────────────────────────────────────────────┘
```

**Key distinction:**
- **Data storage**: Witness field of the transaction (on-chain)
- **Sat association**: Ordinals protocol numbering (tracked by indexers)

### Confirmation Process (L1)

| Stage | Time | What Happens | Finality |
|-------|------|--------------|----------|
| **Broadcast** | Instant | TX enters mempool | ❌ Not confirmed |
| **1 Confirmation** | ~10 min | Included in a block | ⚠️ Weak finality |
| **3 Confirmations** | ~30 min | 2 more blocks built on top | ✅ Generally accepted |
| **6 Confirmations** | ~60 min | Standard for large values | ✅✅ Strong finality |

### Can BRC-20 Transfers Be Verified On-Chain?

**YES** - BRC-20 transfers are fully on-chain and can be verified:

| Verification Method | What It Shows |
|---------------------|---------------|
| **Block Explorers** (mempool.space, blockstream.info) | Raw transaction, witness data |
| **Ordinals Explorers** (ordinals.com, ord.io) | Inscription content, ownership |
| **BRC-20 Indexers** (unisat.io, ordiscan.com) | Token balances, transfer history |

**User's View**: When a user receives BRC-20 tokens to their Taproot wallet, they can:
1. See the UTXO with sats in any Bitcoin wallet
2. See the BRC-20 balance on Unisat, OKX Wallet, or other Ordinals-aware wallets
3. Verify the inscription on ordinals explorers

---

## 6. Cost of BRC-20 Token Transfers (L1)

### How Bitcoin Transaction Fees Work

#### How Miners Process Transactions

Bitcoin is a decentralized network where **miners** (or validators) process transactions:

```
┌─────────────────────────────────────────────────────────────────┐
│                    BITCOIN NETWORK                              │
│                                                                 │
│   SENDER                      MINERS                 RECIPIENT  │
│    │                            │                         │     │
│    │  ──── Transaction ────▶   │                         │     │
│    │       + Fee               │                         │     │
│    │                           │                         │     │
│    │                    [Process TX]                     │     │
│    │                    [Add to block]                   │     │
│    │                    [Solve puzzle]                   │     │
│    │                           │                         │     │
│    │                           │  ──── Confirmed ────▶   │     │
│    │                           │                         │     │
│    │                    [Fee compensation]               │     │
│                                                                 │
│   Transaction Flow:                                             │
│    1. Sender broadcasts transaction with fee                   │
│    2. Miners include in block based on fee priority            │
│    3. Transaction confirmed after block is mined               │
│    4. Recipient can spend funds                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Miners are compensated through:
- **Block rewards**: Currently 3.125 BTC per block (halves every ~4 years)
- **Transaction fees**: Fees attached to each transaction by users

#### Block Space Economics

Bitcoin blocks have limited capacity:

| Parameter | Value |
|-----------|-------|
| **Block size** | ~4 MB (with SegWit) |
| **Block time** | ~10 minutes average |
| **Transactions per block** | ~2,000-4,000 depending on size |
| **Transactions per second (TPS)** | ~7 TPS maximum |

Since block space is limited, transactions with higher fees are prioritized. This creates a **fee market** where users pay based on how quickly they need confirmation.

#### Transaction Fee Calculation

```
Transaction Fee = Transaction Size (vbytes) × Fee Rate (sats/vbyte)
```

**Example:**
- Simple BTC transfer: ~150 vbytes
- Fee rate: 1 sat/vbyte
- Fee = 150 × 1 = 150 sats ≈ **$0.14** (@ $91,756/BTC)

#### Fee Levels and Confirmation Times

| Fee Level | Typical Result |
|-----------|----------------|
| **No fee (0 sats)** | Transaction not relayed by nodes |
| **Very low fee** | May wait days/weeks for confirmation |
| **Low fee (1-5 sat/vbyte)** | Transaction confirms in hours to days |
| **Normal fee (15-30 sat/vbyte)** | Transaction confirms in ~10-60 minutes |
| **High fee (60+ sat/vbyte)** | Transaction confirms in next block (~10 min) |

### BRC-20 Transfer Cost Components

A BRC-20 transfer requires **TWO Bitcoin transactions**:

| Step | What Happens | Transaction Size | On-Chain? |
|------|--------------|------------------|-----------|
| **1. Inscribe Transfer** | Write "transfer X tokens" onto a new UTXO | ~300-400 vbytes | ✅ Yes |
| **2. Send UTXO** | Transfer the inscribed UTXO to recipient | ~150-200 vbytes | ✅ Yes |

Plus the **dust limit** sats required for the new UTXO.

#### Cost Component 1: Inscription Transaction Fee

To transfer a BRC-20 token, you must **inscribe data** onto the Bitcoin blockchain:

```
Inscription = Data stored in the witness field of a transaction, 
              logically attached to a specific satoshi in the output UTXO
              via the Ordinals protocol's satoshi numbering system
```

**How it works:**
- The inscription data (JSON) is **physically stored** in the transaction's witness field
- The Ordinals protocol assigns a unique serial number to every satoshi
- This links the inscription to a **specific sat** within the output UTXO
- When that UTXO is spent and the sat moves, the inscription "travels" with it

The inscription transaction is **larger** than a normal BTC transaction because it contains:
- The standard transaction data
- PLUS the BRC-20 JSON payload (e.g., `{"p":"brc-20","op":"transfer","tick":"BID","amt":"100"}`)

#### Cost Component 2: Send Transaction Fee

After inscribing the transfer, you must **send the inscribed UTXO** to the recipient. This is a second, smaller transaction.

#### Cost Component 3: Dust Limit (The "Paper" for the Token)

Every BRC-20 token must be attached to a UTXO with a **minimum value** (dust limit):

| Address Type | Dust Limit | USD Value |
|--------------|------------|-----------|
| Native SegWit | 330 sats | ~$0.30 |
| Taproot | 330 sats | ~$0.30 |

This is like the **cost of the paper** you write the token on. Even if the "ink" (inscription) were free, you still need paper.

### Cost Calculation (January 2026)

**Reference Values:**
- BTC Price: ~$91,756 USD
- Fee rate used: 1 sat/vbyte (our standard minimum rate)

#### Cost Breakdown at 1 sat/vbyte (via UniSat API)

| Component | Calculation | Sats | USD | % of Total |
|-----------|-------------|------|-----|------------|
| Inscribe TX | UniSat network fee | 182 | $0.17 | 20% |
| Send TX | ~200 vbytes × 1 sat | 200 | $0.18 | 22% |
| Dust Limit | Fixed (Taproot) | 330 | $0.30 | **46%** |
| **TOTAL** | | **928** | **$0.65** | 100% |

**🔴 Highest Cost Component: Dust Limit (59%)** - At minimum fee rates, the fixed dust limit becomes the largest cost component. This cannot be reduced—it's the minimum "bill size" the network accepts.

#### Cost at Different Fee Rates

| Fee Rate | Inscribe TX | Send TX | Dust Limit | Total | Highest Cost |
|----------|-------------|---------|------------|-------|--------------|
| 1 sat/vbyte | $0.17 | $0.18 | $0.50 | **$0.65** | Dust (59%) |
| 10 sat/vbyte | $1.67 | $1.83 | $0.50 | $4.00 | Send TX |
| 30 sat/vbyte | $5.01 | $5.50 | $0.50 | $11.01 | Send TX |
| 60 sat/vbyte | $10.02 | $11.00 | $0.50 | $21.52 | Send TX |

*Note: At 1 sat/vbyte, transactions may take hours or days to confirm depending on network congestion.*

### Cost Summary Reference Table (@ 1 sat/vbyte)

| Action | Cost | Notes |
|--------|------|-------|
| Send BTC (simple) | ~$0.14 | Paid to miners |
| Receive BTC | $0 | No cost to receive |
| Deploy BRC-20 token | ~$0.30-1.00 | One-time creation cost |
| **Transfer BRC-20 token** | **~$0.65** | Per transfer (2 TXs + dust) |
| Hold BRC-20 token | $0 | No ongoing cost |
| View balance | $0 | Free to check balance |

### Cost Projections at Scale (@ 1 sat/vbyte)

| Users | Cost per Transfer | Total Cost (L1) |
|-------|-------------------|-----------------|
| 1,000 | ~$0.65 | ~$650 |
| 10,000 | ~$0.65 | ~$6,500 |
| 100,000 | ~$0.65 | ~$65,000 |
| 1,000,000 | ~$0.65 | ~**$650,000** |

*Note: ~50% of the cost (~$0.30) is the dust limit, which is fixed regardless of fee rate. Costs vary with BTC price.*

---

## 7. Bitcoin Layer 2 Solutions for BRC-20

### Overview of L2 Solutions

Layer 2 solutions process transactions **off the main Bitcoin blockchain**, then periodically settle on-chain. This reduces costs and increases speed.

| Solution | Type | BRC-20 Support | Visible on L1? | How It Works |
|----------|------|----------------|----------------|--------------|
| **Merlin Chain** | ZK-Rollup | ✅ Yes (bridge) | ❌ Only on L2, ✅ after bridge-out | Bridge BRC-20 → M-Token on L2; bridge back for L1 visibility |
| **Spark Network** | L2 (Lightning-based) | ❌ No (LRC-20 only) | ❌ Spark wallets only | Native LRC-20 tokens, completely different from BRC-20 |
| **Dovi** | EVM Sidechain | ✅ Yes (bridge) | ❌ Only on L2, ✅ after bridge-out | Cross-chain bridge for BRC-20 |
| **Lightning Network** | Payment Channels | ❌ No | N/A | Not designed for tokens/inscriptions |

### What "BRC-20 Support" Actually Means

**Important**: L2 solutions don't run native BRC-20 inscriptions. They:

1. **Lock** your BRC-20 tokens on L1 (in a bridge contract)
2. **Mint** equivalent "wrapped" tokens on L2
3. **Allow transfers** of wrapped tokens on L2 (cheap & fast)
4. **Unlock** original BRC-20 when you bridge back to L1

```
Native BRC-20 (L1)  ──bridge in──▶  Wrapped BRC-20 (L2)
     ▲                                     │
     │                                     │
     └───────────bridge out────────────────┘
```

---

## 8. How Merlin Chain L2 Works (On-Chain vs Off-Chain)

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     BITCOIN L1 (On-Chain)                       │
│                                                                 │
│  ┌─────────────┐    ┌─────────────────────────────────────┐    │
│  │ Your BRC-20 │───▶│ Bridge Contract (Multi-sig/Timelock)│    │
│  │ Inscription │    │ - Locks your BRC-20 UTXO            │    │
│  └─────────────┘    │ - Records deposit on L1             │    │
│                     └─────────────────────────────────────┘    │
│                                                                 │
│  What's on L1:                                                 │
│   • Original BRC-20 inscription (locked in bridge)             │
│   • Bridge deposit transaction                                  │
│   • Periodic state root commitments from L2                     │
│   • Bridge withdrawal transactions                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MERLIN CHAIN L2 (Off-Chain)                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    ZK-Rollup State                       │   │
│  │  • Wrapped BRC-20 balances (like ERC-20 on Ethereum)    │   │
│  │  • Transaction history                                   │   │
│  │  • Smart contract state                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  What happens on L2:                                           │
│   • User A transfers 100 wBID to User B (instant, ~$0.05)     │
│   • User B transfers 50 wBID to User C (instant, ~$0.05)      │
│   • Hundreds of transactions accumulated                       │
│                                                                 │
│  NOT on Bitcoin L1:                                            │
│   • Individual L2 transfers                                    │
│   • L2 balances (stored on Merlin nodes)                       │
│   • L2 transaction details                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SETTLEMENT (Periodic)                        │
│                                                                 │
│  Every N blocks or M transactions:                             │
│   1. L2 generates ZK-proof of all state transitions            │
│   2. Proof + state root published to Bitcoin L1                │
│   3. One L1 transaction covers 1000s of L2 transactions        │
│                                                                 │
│  L1 Transaction Contains:                                      │
│   • Merkle root of L2 state                                    │
│   • ZK-proof (validity proof)                                  │
│   • Batch ID                                                   │
│                                                                 │
│  Can be verified:                                              │
│   ✅ State root on L1 (proves L2 state is valid)               │
│   ✅ ZK-proof verifiable by anyone                             │
│   ⚠️  Individual L2 transactions only on L2 explorers          │
└─────────────────────────────────────────────────────────────────┘
```

### L2 Confirmation Process

| Stage | Time | What Happens | Where Visible |
|-------|------|--------------|---------------|
| **L2 TX Broadcast** | Instant | TX in L2 mempool | L2 Explorer |
| **L2 Confirmation** | 1-5 sec | Included in L2 block | L2 Explorer |
| **L2 Finality** | ~10-30 sec | L2 considers final | L2 Explorer |
| **L1 Settlement** | Hours-Days | Batch proof on Bitcoin | Bitcoin Explorer |
| **Full Finality** | After L1 confirmation | Backed by Bitcoin security | Both |

### What Users See

| Wallet Type | What They See |
|-------------|---------------|
| **Standard Bitcoin Wallet** | Nothing (L2 tokens invisible) |
| **Merlin-Compatible Wallet** | Wrapped BRC-20 balance on L2 |
| **After Bridge Out to L1** | Native BRC-20 in Taproot wallet |

---

## 9. L2 Withdrawal Costs (Bridge to L1 BRC-20)

### The Critical Question

> **Goal**: User must have real BRC-20 tokens in their Taproot wallet, viewable on chain explorers like Unisat or OrdScan.

This requires **bridging from L2 back to L1**, which is the expensive part.

### Bridge-Out (Withdrawal) Process

```
┌─────────────────────────────────────────────────────────────────┐
│                    BRIDGE-OUT PROCESS                           │
│                                                                 │
│  Step 1: User requests withdrawal on L2                        │
│    └── L2 Fee: ~$0.05-0.10                                     │
│    └── Burns wrapped tokens on L2                              │
│                                                                 │
│  Step 2: Wait for settlement batch                             │
│    └── Time: Hours to days (depends on L2)                     │
│    └── L2 accumulates withdrawal requests                      │
│                                                                 │
│  Step 3: L1 withdrawal transaction                             │
│    └── Bridge contract releases locked BRC-20                  │
│    └── Creates NEW inscription for user's portion              │
│    └── 🔴 THIS IS THE EXPENSIVE PART                           │
│                                                                 │
│  What happens on L1:                                           │
│    • Inscription TX: Create "transfer X BID" inscription       │
│    • Send TX: Send inscribed UTXO to user's Taproot address   │
│    • Dust: 330 sats locked in the UTXO                        │
└─────────────────────────────────────────────────────────────────┘
```

### Bridge-Out Cost Breakdown (Per User @ 1 sat/vbyte)

| Component | Cost | Who Pays? | On-Chain? |
|-----------|------|-----------|-----------|
| L2 Withdrawal Request | ~$0.05 | User or Beem | L2 only |
| L1 Inscription TX | ~$0.28 | **Beem** (bridge operator) | ✅ L1 |
| L1 Send TX | ~$0.17 | **Beem** (bridge operator) | ✅ L1 |
| Dust Limit (330 sats) | ~$0.30 | Beem | ✅ L1 |
| **TOTAL per user** | **~$1.00** | | |

**Key Insight**: At 1 sat/vbyte, the dust limit (~$0.30) represents 50% of the bridge-out cost.

### Who Pays Bridge-Out Costs?

| Model | Description | Cost to Beem per User |
|-------|-------------|----------------------|
| **Beem Pays All** | Beem covers all withdrawal costs | ~$1.00 |
| **User Pays L1** | User pays L1 fees at withdrawal | ~$0.05 (L2 only) |
| **Shared Cost** | Beem pays dust, user pays fees | ~$0.30 |

### Total Cost Analysis with Bridge-Out (@ 1 sat/vbyte)

#### Scenario: 1,000,000 Users, All Eventually Withdraw to L1

| Approach | Distribution Cost | Bridge-Out Cost | **TOTAL** |
|----------|-------------------|-----------------|-----------|
| **Pure L1** | ~$1M | $0 (already on L1) | **~$1M** |
| **L2 + Bridge Out (Beem pays)** | $150K | ~$1M | **~$1.15M** |
| **L2 + Bridge Out (User pays)** | $150K | $0 | **$150K** |

**Conclusion**: At 1 sat/vbyte, the L2 approach with user-paid withdrawals provides significant savings. The L2 savings are maximized when:
- Users pay their own withdrawal fees, OR
- Many users never withdraw (keep on L2), OR
- Users do multiple L2 transfers before withdrawing

---

## 10. Comparison: Shadow Balance vs L2 vs L1

### Your Current Shadow Balance System

Based on your `ShadowBalanceService`, you're running an **off-chain ledger system**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEEM SHADOW BALANCE                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                PostgreSQL Database                       │   │
│  │  • shadow_balances table (user_id, balance, currency)   │   │
│  │  • processed_events table (idempotency)                 │   │
│  │  • processing_state table (cursor position)             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  How it works:                                                 │
│   1. User does action (qr_verified) → ledger event            │
│   2. ShadowBalanceService processes event                      │
│   3. Balance updated in PostgreSQL (ACID, optimistic locking) │
│   4. User sees balance in Beem app                             │
│                                                                 │
│  What's on Bitcoin L1: NOTHING                                 │
│  What users can verify externally: NOTHING                     │
│  Trust model: Trust Beem's database                            │
└─────────────────────────────────────────────────────────────────┘
```

### Comparison Table

| Aspect | Shadow Balance | L2 (Merlin) | L1 (BRC-20) |
|--------|---------------|-------------|-------------|
| **Where balance lives** | Beem PostgreSQL | L2 chain state | Bitcoin blockchain |
| **Transfer cost** | **$0** | **~$0.05** | **~$0.65** |
| **Transfer speed** | Instant | Seconds | ~10+ minutes |
| **Data on Bitcoin L1?** | ❌ No | ❌ No (until bridge-out) | ✅ Yes |
| **Verifiable on Unisat/L1?** | ❌ No | ❌ No (until bridge-out) | ✅ Yes |
| **Trust model** | Trust Beem | Trust L2 operators | Trust Bitcoin |
| **Token is "real" BRC-20** | ❌ No (database entry) | ❌ No (wrapped M-Token) | ✅ Yes |
| **Can sell on L1 DEX (Unisat)** | ❌ No | ❌ No (until bridge-out) | ✅ Yes |

### 🔴 The Critical Insight: Cost to Get Real L1 BRC-20

```
┌─────────────────────────────────────────────────────────────────┐
│     EVERY PATH TO REAL L1 BRC-20 COSTS ~$0.65                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   FROM SHADOW BALANCE:                                          │
│   ┌──────────────┐         ┌──────────────┐                    │
│   │ Shadow       │ ──$0.65──▶ │ Real L1     │                    │
│   │ Balance      │  redeem  │ BRC-20      │                    │
│   │ (database)   │         │ (on-chain)  │                    │
│   └──────────────┘         └──────────────┘                    │
│                                                                 │
│   FROM MERLIN L2:                                               │
│   ┌──────────────┐         ┌──────────────┐                    │
│   │ Wrapped      │ ──$0.65──▶ │ Real L1     │                    │
│   │ M-Token      │ bridge   │ BRC-20      │                    │
│   │ (L2 chain)   │   out    │ (on-chain)  │                    │
│   └──────────────┘         └──────────────┘                    │
│                                                                 │
│   DIRECT L1:                                                    │
│   ┌──────────────┐         ┌──────────────┐                    │
│   │ Your L1      │ ──$0.65──▶ │ User's L1   │                    │
│   │ BRC-20       │ transfer │ BRC-20      │                    │
│   └──────────────┘         │ (on-chain)  │                    │
│                            └──────────────┘                    │
│                                                                 │
│   THE ~$0.65 L1 COST IS UNAVOIDABLE because:                   │
│   • Inscription TX (~$0.28) - write data to Bitcoin            │
│   • Send TX (~$0.17) - transfer UTXO to user                   │
│   • Dust limit (~$0.30) - minimum sats for UTXO                │
│                                                                 │
│   This is the cost of writing to the Bitcoin blockchain.        │
│   Shadow Balance and L2 are cheap because they DON'T write     │
│   to L1 - that's why balances aren't verifiable on L1.         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why Shadow Balance / L2 transfers are cheap:**
- Data is stored **off-chain** (database or L2 chain)
- No Bitcoin L1 transaction required
- No miners to pay, no block space to consume
- **Trade-off**: Balance is NOT visible or verifiable on Bitcoin L1

**Why L1 transfers cost ~$0.65:**
- Data is written **on-chain** to Bitcoin blockchain
- Requires paying miners for block space
- Inscription + Send transactions + Dust limit
- **Benefit**: Token is REAL, verifiable, tradeable on L1

### Shadow Balance ≈ Centralized L2

Your shadow balance system is essentially a **centralized Layer 2**:

| L2 Concept | Shadow Balance Equivalent |
|------------|---------------------------|
| L2 Chain | PostgreSQL database |
| L2 Transactions | ledger_events table |
| L2 Balances | shadow_balances table |
| Bridge In | User earns BID (qr_verified) |
| Bridge Out | `redeem_balance()` → L1 BRC-20 |
| Settlement | N/A (never settles to L1) |

### Cost Comparison for "Redemption" (Getting Real BRC-20 @ 1 sat/vbyte)

When a user wants to convert shadow balance to real BRC-20:

| From | To | Cost per User | Time |
|------|-----|---------------|------|
| Shadow Balance | L1 BRC-20 | ~$0.92-1.04 | Hours-Days |
| Shadow Balance | L2 Wrapped BRC-20 | ~$0.05 | Seconds |
| L2 Wrapped | L1 BRC-20 | ~$0.92-1.04 | Hours-Days |

**Bottom Line**: At 1 sat/vbyte, getting real BRC-20 to a user's Taproot wallet costs ~$1 per user. The savings from L2/Shadow Balance come from:
1. Delaying the L1 inscription until the user actually needs it
2. Having users pay their own withdrawal fees
3. Users who never withdraw (stay on L2 or shadow balance)

---

## 11. Alternative Token Standards on Bitcoin Blockchain

This section covers all major token standards available on the Bitcoin blockchain that could serve as alternatives or replacements for BRC-20.

---

### 11.1 Bitcoin L1 Token Standards Comparison

| Standard | Cost (1 sat/vB) | Data On L1? | Visible on L1 Explorer? | Verifiable Via | Adoption |
|----------|-----------------|-------------|-------------------------|----------------|----------|
| **BRC-20** | ~$0.65 | ✅ Witness | ✅ Unisat, OKX, OrdScan | L1 Ordinals indexers | ⭐⭐⭐⭐⭐ High |
| **BRC2.0** | ~$0.65 | ✅ Witness | ✅ Unisat + BRC2.0 indexer | L1 Ordinals + BRC2.0 indexers | ⭐⭐⭐ Growing |
| **Runes** | ~$0.40 | ✅ OP_RETURN | ✅ Unisat, Magic Eden | L1 Runes indexers | ⭐⭐⭐⭐ Medium-High |
| **LRC-20** | ~$0.01-0.05 | ⚠️ Deploy only | ❌ Spark wallets only | Spark L2 indexer only | ⭐⭐ New |
| **ORC-20** | ~$0.65 | ✅ Witness | ⚠️ ORC-20 explorers | L1 ORC-20 indexers | ⭐⭐ Low |

---

### 11.2 Detailed Token Standard Explanations

#### **BRC-20 (Original)**

**What it is**: The original Bitcoin fungible token standard, created by pseudonymous developer "Domo" in March 2023.

**How it works**:
- Uses Ordinals protocol to inscribe JSON data onto individual satoshis
- Three operations: `deploy`, `mint`, `transfer`
- Requires off-chain indexers to track balances
- Each transfer requires TWO transactions (inscribe + send)

**Example inscription**:
```json
{"p":"brc-20","op":"transfer","tick":"BID","amt":"100"}
```

| Pros | Cons |
|------|------|
| ✅ Highest adoption | ❌ ~$0.65/transfer @ 1 sat/vB (higher at congestion) |
| ✅ Supported by all major wallets | ❌ No smart contracts |
| ✅ Viewable on Unisat, OKX, etc. | ❌ Requires 2 transactions |
| ✅ Established ecosystem | ❌ Relies on indexers |

---

#### **BRC2.0 (EVM-Compatible Upgrade)**

**What it is**: Major upgrade to BRC-20 activated at Bitcoin block 912,690 (September 2025) that adds Ethereum Virtual Machine (EVM) compatibility.

**How it works**:
- Embeds EVM functionality directly into the BRC-20 indexer
- Allows Ethereum-style smart contracts on Bitcoin
- Enables DeFi, dApps, and programmable tokens
- Backward compatible with BRC-20 tokens

**Key innovation**: Developers can now deploy Solidity smart contracts that interact with BRC-20 tokens without bridges.

| Pros | Cons |
|------|------|
| ✅ Smart contract support | ❌ Same base cost as BRC-20 |
| ✅ EVM compatibility | ❌ Newer, less tested |
| ✅ DeFi on Bitcoin | ❌ Relies on indexer upgrade |
| ✅ Backward compatible | ❌ Not all wallets support yet |

**Best for**: Projects needing programmable tokens with DeFi functionality on Bitcoin.

---

#### **Runes Protocol**

**What it is**: A simpler, more efficient token standard created by Casey Rodarmor (creator of Ordinals), launched at Bitcoin halving (April 2024).

**How it works**:
- Uses **OP_RETURN** field instead of witness data
- OP_RETURN limited to 80 bytes (very efficient)
- Single transaction for transfers (not two like BRC-20)
- UTXO-native design, better wallet compatibility

**Example**:
```
OP_RETURN: RUNE_ID | AMOUNT | DESTINATION
```

| Aspect | BRC-20 | Runes |
|--------|--------|-------|
| **Data location** | Witness (large) | OP_RETURN (80 bytes max) |
| **Transactions per transfer** | 2 | 1 |
| **Transfer cost @ 1 sat/vB** | ~$0.65 | ~$0.40 |
| **Transfer cost @ 30 sat/vB** | ~$13-14 | ~$3-8 |
| **Creator** | Domo | Casey Rodarmor |
| **UTXO compatible** | Partial | ✅ Native |
| **Layer 2 compatible** | Limited | ✅ Better |

| Pros | Cons |
|------|------|
| ✅ 60-75% cheaper than BRC-20 | ❌ Lower adoption than BRC-20 |
| ✅ Single transaction transfers | ❌ Different ecosystem/wallets |
| ✅ Better L2 compatibility | ❌ Not BRC-20 compatible |
| ✅ Smaller on-chain footprint | ❌ Still requires indexers |

**Best for**: New projects prioritizing cost efficiency over BRC-20 compatibility.

---

#### **LRC-20 (Spark Network)**

**What it is**: A token standard native to the **Spark network**, a Bitcoin Layer 2 solution launched by Lightspark in Summer 2024. Designed for very low-cost, instant token transfers.

**How it works**:
- **Token deployment**: Broadcast a transaction on Bitcoin mainnet with token metadata in OP_RETURN output
- **Token transfers**: Happen on Spark L2, not on Bitcoin L1
- **Settlement**: Uses Bitcoin as the settlement layer, Spark as the execution layer
- **Architecture**: Similar to Lightning Network but optimized for tokens

```
┌─────────────────────────────────────────────────────────────────┐
│                    LRC-20 ON SPARK NETWORK                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   BITCOIN L1:                                                   │
│   └── Token deployment (OP_RETURN with metadata)                │
│   └── Settlement/anchoring of L2 state                          │
│                                                                 │
│   SPARK L2 (Execution Layer):                                   │
│   └── Token minting (only issuer wallet can mint)               │
│   └── Token transfers (instant, ~$0.01-0.05)                    │
│   └── Token freezing/burning (issuer controls)                  │
│                                                                 │
│   KEY FEATURES:                                                 │
│   ├── Very low cost (~$0.01-0.05 per transfer)                  │
│   ├── Instant transactions                                      │
│   ├── Lightning Network compatible                              │
│   └── Self-custodial wallets supported                          │
│                                                                 │
│   LIMITATIONS:                                                  │
│   ├── No fair launch (only issuer can mint)                     │
│   ├── Issuer can freeze tokens on any address                   │
│   ├── Only works on Spark network                               │
│   └── NOT viewable on Unisat/Ordinals explorers                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| ✅ Very cheap (~$0.01-0.05/transfer) | ❌ Centralized issuance (only issuer mints) |
| ✅ Instant transactions | ❌ Issuer can freeze tokens |
| ✅ Lightning Network compatible | ❌ Not visible on Unisat/Ordinals |
| ✅ Self-custodial | ❌ Requires Spark-compatible wallet |
| ✅ Bitcoin settlement layer | ❌ Newer, smaller ecosystem |

**Best for**: Projects needing very low cost transfers, willing to accept centralized token control, and don't need BRC-20/Ordinals visibility.

**Example Token**: SPARKY - First auto-distributed LRC-20 token (21M supply)

---

#### **ORC-20 (Ordinals-based, Flexible)**

**What it is**: An improvement on BRC-20 designed to fix its limitations, launched May 2023.

**How it works**:
- Built on Ordinals like BRC-20
- Adds flexible supply (can increase/decrease)
- Supports partial transfers
- Uses namespacing to prevent ticker collisions

**Key improvements over BRC-20**:
- Mutable token supply
- Cancel/upgrade token parameters
- Whitelist minting
- Migration from BRC-20 tokens

| Pros | Cons |
|------|------|
| ✅ More flexible than BRC-20 | ❌ Lower adoption |
| ✅ Upgradeable tokens | ❌ Same cost as BRC-20 |
| ✅ Better token management | ❌ Less wallet support |

**Best for**: Projects needing flexible token supply or upgradeable parameters.

---

### 11.3 L2/Sidechain Token Options

If staying on Bitcoin L1 is not required, these options offer significant cost savings:

#### **Spark Network (LRC-20)**

| Aspect | Details |
|--------|---------|
| **Token Type** | LRC-20 (native to Spark, NOT BRC-20) |
| **BRC-20 Support** | ❌ **No** - completely different token system |
| **Visibility on L1** | ❌ NOT visible on Unisat/Ordinals (only deploy TX in OP_RETURN) |
| **Visibility** | Spark-compatible wallets only |
| **Compatible Wallets** | Spark protocol wallets |
| **DEX Trading** | Limited (Spark ecosystem) |
| **Cost per transfer** | **~$0.01-0.05** |
| **Lightning compatible** | ✅ Yes |
| **Who controls minting?** | Token issuer only (centralized) |
| **Can issuer freeze tokens?** | ⚠️ Yes |

**⚠️ Important**: LRC-20 is a completely different token standard from BRC-20. LRC-20 tokens are NOT visible on any Bitcoin L1 explorer or Ordinals indexer (Unisat, OrdScan, etc.).

#### **Merlin Chain (Wrapped BRC-20)**

| Aspect | Details |
|--------|---------|
| **Token Type** | M-Token (wrapped BRC-20, ERC-20 style) |
| **BRC-20 Support** | ✅ Yes - bridges real BRC-20 from L1 |
| **Visibility on L2** | Merlin block explorer (M-Token balance) |
| **Visibility on L1** | ✅ **After bridge-out** → Real BRC-20 on Unisat |
| **Compatible Wallets** | MetaMask, OKX, Unisat (Merlin mode) |
| **DEX Trading** | Yes (Merlin DEXs) |
| **Cost per transfer** | ~$0.05 on L2 |
| **Bridge-out cost** | ~$0.65 (L1 inscription + send + dust) |
| **Bridge-out time** | ~24 hours (L2) or ~3 days (L1) |

**How BRC-20 Bridge Works:**
1. **Bridge IN**: Lock BRC-20 on L1 → Receive M-Token on Merlin L2
2. **Trade on L2**: Transfer M-Tokens cheaply (~$0.05)
3. **Bridge OUT**: Burn M-Token → Receive real BRC-20 on L1 (visible on Unisat)

### 11.4 Recommendation Matrix

| Your Goal | Recommended | Why | Visible on Unisat? |
|-----------|-------------|-----|-------------------|
| **Maximum adoption, viewable on Unisat** | **BRC-20** | Highest ecosystem support | ✅ Yes |
| Smart contracts on Bitcoin L1 | **BRC2.0** | EVM on BRC-20 indexer | ✅ Yes |
| Cheapest L1 fungible tokens | **Runes** | ~$0.40 vs BRC-20's ~$0.65 | ✅ Yes |
| Flexible/upgradeable L1 tokens | **ORC-20** | Mutable parameters | ⚠️ ORC explorers |
| **Cheap L2 + eventual L1 visibility** | **Merlin L2** | ~$0.05 on L2, bridge out to L1 | ✅ After bridge-out |
| Cheap L2 with Lightning | **LRC-20 (Spark)** | ~$0.01-0.05 | ❌ Never |

---

### 11.5 Cost Comparison Summary

| Standard | Cost @ 1 sat/vB | L1/L2 | Data on L1 Bitcoin? | Visible on Unisat/Ordinals? | BRC-20 Compatible? |
|----------|-----------------|-------|---------------------|-----------------------------|--------------------|
| **BRC-20** | ~$0.65 | L1 | ✅ Yes (witness) | ✅ Yes | ✅ Native |
| **BRC2.0** | ~$0.65 | L1 | ✅ Yes (witness) | ✅ Yes (+ BRC2.0 indexer) | ✅ Extension |
| **Runes** | ~$0.40 | L1 | ✅ Yes (OP_RETURN) | ✅ Yes (Runes explorers) | ❌ Different |
| **ORC-20** | ~$0.65 | L1 | ✅ Yes (witness) | ⚠️ ORC-20 explorers | ❌ Different |
| **Merlin L2** | ~$0.05 | L2 | ❌ Until bridge-out | ✅ **After bridge-out only** | ✅ Via bridge |
| **LRC-20 (Spark)** | ~$0.01-0.05 | L2 | ⚠️ Deploy only | ❌ Never | ❌ Different |

**Key Insights**:
- **For Unisat/OKX visibility**: Must use L1 standards (BRC-20, BRC2.0, Runes) OR bridge out from Merlin
- **Cheapest L1 option**: Runes at ~$0.40 per transfer
- **Cheapest with eventual L1 visibility**: Merlin L2 (~$0.05 on L2, ~$0.65 to bridge out)
- **LRC-20 is NOT BRC-20** - completely different, never visible on Ordinals explorers

---

## 12. Single Operation Cost Breakdown (@ 1 sat/vbyte)

### Goal: Deliver 1 BRC-20 Token to 1 User's Taproot Wallet (Viewable on Explorer)

#### Approach A: Pure L1 (Direct Inscription)

| Step | Description | Cost | Highest? |
|------|-------------|------|----------|
| 1 | Inscribe "transfer X BID" | ~$0.28 | |
| 2 | Send UTXO to user's Taproot | ~$0.17 | |
| 3 | Dust locked in UTXO | ~$0.30 | 🔴 **YES (53%)** |
| **TOTAL** | | **~$0.65** | |

#### Approach B: Shadow Balance → L1 Redemption

| Step | Description | Cost | Highest? |
|------|-------------|------|----------|
| 1 | User earns BID (shadow balance update) | $0 | |
| 2 | User requests redemption | $0 | |
| 3 | Beem inscribes "transfer X BID" | ~$0.28 | |
| 4 | Beem sends UTXO to user | ~$0.17 | |
| 5 | Dust locked in UTXO | ~$0.30 | 🔴 **YES (53%)** |
| **TOTAL** | | **~$0.65** | |

#### Approach C: L2 Distribution → Bridge Out to L1

| Step | Description | Cost | Highest? |
|------|-------------|------|----------|
| 1 | Beem bridges tokens to L2 (one-time) | ~$1-2 | Amortized |
| 2 | Beem transfers to user on L2 | ~$0.05 | |
| 3 | User requests bridge out | ~$0.05 | |
| 4 | Bridge inscribes "transfer X BID" | ~$0.28 | |
| 5 | Bridge sends UTXO to user | ~$0.17 | |
| 6 | Dust locked in UTXO | ~$0.30 | 🔴 **YES (53%)** |
| **TOTAL** | | **~$1.05** | |

### Key Insight (@ 1 sat/vbyte)

**At minimum fee rates, the dust limit (~$0.30) becomes the highest cost component (~53%).**

The dust limit is a fixed cost that cannot be reduced—it's the minimum sats required for the blockchain to recognize the UTXO.

---

## 13. Mass Distribution Cost Analysis (@ 1 sat/vbyte)

### Scenario: 1,000,000 Users Need Real BRC-20 in Taproot Wallets

#### Model 1: Beem Pays All Costs

| Approach | Distribution | Bridge Out | **Total Cost** | **Per User** |
|----------|--------------|------------|----------------|--------------|
| Pure L1 | ~$650K | - | **~$650K** | **~$0.65** |
| Shadow → L1 | $0 | ~$650K | **~$650K** | **~$0.65** |
| L2 → L1 | $50K | ~$650K | **~$990K** | **~$0.99** |

**Conclusion**: At 1 sat/vbyte, all approaches cost ~$1M for 1 million users if Beem pays everything.

#### Model 2: User Pays Bridge-Out/Redemption

| Approach | Beem Pays | User Pays | **Total to Beem** |
|----------|-----------|-----------|-------------------|
| Pure L1 | ~$650K | $0 | **~$650K** |
| Shadow → L1 | $0 | ~$650K | **$0** |
| L2 → L1 | $50K | ~$650K | **$50K** |

**Conclusion**: If users pay their own L1 fees, Beem pays only $0-50K.

#### Model 3: Only 10% of Users Withdraw to L1

| Approach | Active Users (L1) | Passive Users | **Total to Beem** |
|----------|-------------------|---------------|-------------------|
| Pure L1 | All users get L1 | N/A | **~$650K** |
| Shadow Balance | 100K get L1 | 900K stay shadow | **~$94K** |
| L2 | 100K bridge out | 900K stay L2 | **~$144K** |

**Conclusion**: Shadow balance or L2 saves ~90% if most users never withdraw.

---

## 14. Recommendations for BRC-20 to Taproot Wallets

### Ultimate Goal Recap

> Users must have real BRC-20 tokens in their Taproot wallets, viewable on chain explorers (Unisat, OrdScan, etc.)

### Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                    DECISION TREE                                │
│                                                                 │
│  Q1: Do ALL users need L1 BRC-20 immediately?                  │
│    ├── YES → Pure L1 (cost ~$0.65/user @ 1 sat/vB)             │
│    └── NO → Continue to Q2                                     │
│                                                                 │
│  Q2: Who pays for L1 redemption?                               │
│    ├── BEEM → Same cost as Pure L1, just deferred              │
│    └── USER → Beem cost ~$0-0.15/user (L2/Shadow only)         │
│                                                                 │
│  Q3: What % of users will actually withdraw to L1?             │
│    ├── >50% → Consider Pure L1 or user-pays model              │
│    ├── 10-50% → L2/Shadow + selective L1 redemption            │
│    └── <10% → L2/Shadow is highly cost-effective               │
└─────────────────────────────────────────────────────────────────┘
```

### Recommended Strategy: Hybrid with User-Paid Redemption

```
┌─────────────────────────────────────────────────────────────────┐
│                    RECOMMENDED APPROACH                         │
│                                                                 │
│  PHASE 1: Internal Distribution (Zero L1 Cost)                 │
│    └── Keep using Shadow Balance for earning/tracking          │
│    └── Users see balance in Beem app                           │
│    └── Cost: $0 per user                                       │
│                                                                 │
│  PHASE 2: Optional L2 Bridge (Low Cost)                        │
│    └── Users can move shadow balance → Merlin Chain            │
│    └── Enables trading on L2 DEXs                              │
│    └── Cost: ~$0.05-0.15 per user                              │
│                                                                 │
│  PHASE 3: L1 Redemption (User Pays)                            │
│    └── Users request withdrawal to Taproot wallet              │
│    └── User pays L1 inscription + send fees (~$0.44 @ 1 sat/vB)│
│    └── Beem pays only dust (~$0.30) or nothing                 │
│    └── Real BRC-20 viewable on Unisat, etc.                    │
│                                                                 │
│  RESULT:                                                        │
│    ├── Beem cost for 1M users: ~$0-500K                        │
│    ├── Users who want L1 pay their own fees                    │
│    └── Only motivated users pay for L1 (market self-selects)   │
└─────────────────────────────────────────────────────────────────┘
```

### Cost Summary by Strategy (@ 1 sat/vbyte)

| Strategy | Beem Cost (1M users) | User Cost | Real L1 BRC-20? |
|----------|---------------------|-----------|-----------------|
| Pure L1, Beem pays | ~$650K | $0 | ✅ All users |
| Shadow + User-paid L1 | ~$0 | ~$0.65 at redemption | ✅ On demand |
| L2 + User-paid L1 | ~$50K | ~$0.65 at bridge-out | ✅ On demand |
| Shadow only (no L1) | $0 | $0 | ❌ Never |
| L2 only (no bridge-out) | $50K | $0 | ❌ Wrapped only |

### Final Recommendation

**For maximum cost efficiency while still enabling real L1 BRC-20:**

1. **Continue with Shadow Balance** for internal tracking (transfers = $0)
2. **Add L2 option** (Merlin Chain) for users who want to trade (transfers = ~$0.05)
3. **Offer L1 redemption** where **user pays the L1 fees** (~$0.65 at 1 sat/vbyte)
4. **Beem covers only dust** ($0.50/user) or passes that to user too

**Total Beem cost for 1M users: $0 - $50K** (vs ~$650K for pure L1 at 1 sat/vbyte)

### Key Takeaway: The Unavoidable L1 Cost

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   🔴 TO GET REAL BRC-20 ON L1: ~$0.65 per user                 │
│                                                                 │
│   This cost exists because:                                     │
│   • Data must be written to Bitcoin blockchain                  │
│   • Miners must be paid for block space                         │
│   • Every transfer requires inscription + send + dust           │
│                                                                 │
│   Shadow Balance and L2 are cheap because they are OFF-CHAIN:  │
│   • No L1 transaction = No L1 cost                             │
│   • Trade-off: NOT verifiable on Bitcoin L1 blockchain         │
│                                                                 │
│   If the goal is "BRC-20 visible on Unisat in user's wallet",  │
│   there is NO way to avoid the ~$0.65 L1 cost.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## BRC-20 Inscription Service Providers (Detailed)

This section provides detailed comparison of inscription service providers for BRC-20 token transfers.

### How Inscription Costs Work

**Two transactions are required to deliver BRC-20 tokens to a user:**
1. **Inscribe** - Create the BRC-20 transfer inscription (service provider handles this)
2. **Send** - Transfer the inscription to user's wallet (~200 sats, handled by us)

| Step | Cost | Notes |
|------|------|-------|
| **Step 1: Inscription** | Network fee + **Service fee** | ⚠️ Service fee varies by provider |
| **Step 2: Send UTXO** | Network fee only | Same for all (~$0.18) |
| **Dust limit** | ~$0.30 | Same for all (unavoidable) |

### Provider Comparison Table

| Provider | Inscription Service? | BRC-20 API? | Service Fee | Link |
|----------|---------------------|-------------|-------------|------|
| **UniSat** | ✅ Yes | ✅ Yes | **API: FREE** (UI: 3,150 sats) | [unisat.io](https://unisat.io) |
| **Binance Web3** | ✅ Yes (Inscriptions Marketplace) | ✅ Yes (via UniSat) | ❌ Not public | [binance.com](https://www.binance.com) |
| **Xverse** | ✅ Yes | ⚠️ Under development | **2,000 sats** + 117 sats network | [xverse.app](https://www.xverse.app) |
| **OrdinalsBot** | ✅ Yes (bulk minting) | ⚠️ Runes only (no BRC-20 API) | **2,000 sats** + 199 sats network | [ordinalsbot.com/inscribe](https://ordinalsbot.com/inscribe) |
| **Gamma** | ✅ Yes | ❌ UI only | **2,500 sats** + 294 sats network | [gamma.io](https://gamma.io) |
| **InscribeNFTs** | ✅ Yes | ❌ UI only (no API) | **1,465 sats** | [inscribenfts.com](https://inscribenfts.com/) |
| **Ordinals Wallet** | ✅ Yes | ❌ UI only | ❌ Not public | [ordinalswallet.com](https://ordinalswallet.com) |

### Wallets Only (No Inscription Service)

| Provider | Notes |
|----------|-------|
| **Leather (Hiro)** | Wallet for holding/sending, no inscription service |
| **Magic Eden** | Marketplace for trading, no inscription service |

### Providers with BRC-20 Developer APIs

1. **UniSat** - Most popular, best documentation, open-source, **FREE API**
2. **Binance Web3** - Uses UniSat API internally
3. **Xverse** - API under development

### Providers with UI Only (No BRC-20 API)

- **OrdinalsBot** - UI only for BRC-20 (has API for Runes), 2,000 sats service fee
- **Gamma** - UI only, 2,500 sats service fee
- **InscribeNFTs** - UI only, 1,465 sats service fee
- **Ordinals Wallet** - UI only, pricing not public

### Detailed Cost Comparison

| Provider | Service Fee | Network Fee | Dust | Send Fee | **Total Cost** | API? |
|----------|-------------|-------------|------|----------|----------------|------|
| **UniSat API** | **FREE** | 182 sats | 330 sats | ~200 sats | **~$0.65** | ✅ Yes |
| **InscribeNFTs** | 1,465 sats | included | 330 sats | ~200 sats | **~$1.83** | ❌ UI only |
| **Xverse** | 2,000 sats | 117 sats | 330 sats | ~200 sats | **~$2.43** | ⚠️ In dev |
| **OrdinalsBot** | 2,000 sats | 199 sats | 330 sats | ~200 sats | **~$2.50** | ⚠️ Runes only |
| **Gamma** | 2,500 sats | 294 sats | 330 sats | ~200 sats | **~$3.05** | ❌ UI only |
| **UniSat UI** | 3,150 sats | 182 sats | 330 sats | ~200 sats | **~$3.54** | - |

### Example Cost Breakdowns

**OrdinalsBot:**

| Component | Sats | USD (@ $91,756/BTC) |
|-----------|------|---------------------|
| **Service fee** | **2,000** | **~$1.84** |
| Inscription sats (dust) | 330 | ~$0.30 |
| Network fee | 199 | ~$0.18 |
| Send fee | ~200 | ~$0.18 |
| **TOTAL** | **~2,945** | **~$2.50** |

**InscribeNFTs:**

| Component | Sats | USD (@ $91,756/BTC) |
|-----------|------|---------------------|
| **Service fee** | **1,465** | **~$1.34** |
| Dust limit | 330 | ~$0.30 |
| Send fee | ~200 | ~$0.18 |
| **TOTAL** | **~2,211** | **~$1.83** |

### Key Takeaways

- **UniSat API is the clear winner** - FREE service fee → **~$0.65 total per transfer**
- Total = Inscription (182 sats network + 330 sats dust) + Send (~200 sats) = **712 sats**
- UI-based service fees range from **1,465 to 3,150 sats** (~$1.34-$2.89)
- UI services significantly increase total cost to ~$1.83-$3.74
- **UniSat** is the only provider with a production BRC-20 API and transparent, free pricing
- **OrdinalsBot** has API but only for Runes, not BRC-20
- **Xverse** API is under development

### Self-Hosting Considerations

Even if we were to build our own self-hosted inscription infrastructure (running a Bitcoin full node with ord CLI):
- **Network fees are unavoidable** — miners must still be paid (~182 sats at 1 sat/vbyte)
- Self-hosting saves on service fees but NOT on network fees or dust limit
- Would require significant infrastructure investment (Bitcoin node, ord integration, maintenance)
- **We do not currently have self-hosted inscription infrastructure**

**Bottom line**: UniSat API at $0.65/transfer is the most practical and cost-effective option.

---

## Appendix: Quick Reference

### Conversion Table

| Unit | Value |
|------|-------|
| 1 BTC | 100,000,000 sats |
| 1 sat | 0.00000001 BTC |
| 1 sat (USD @ $91,756) | ~$0.00092 |
| 1,000 sats | ~$0.92 |
| 10,000 sats | ~$9.18 |

### Fee Rate Reference

| Network State | Fee Rate (sats/vbyte) | BRC-20 Transfer Cost |
|---------------|----------------------|----------------------|
| Minimum | 1 | ~$0.65 |
| Very Low | 1-5 | ~$0.65-2.50 |
| Low | 5-15 | ~$2.50-6.00 |
| Normal | 15-30 | ~$6.00-14.00 |
| High | 30-60 | ~$14.00-27.00 |
| Very High | 60-100+ | ~$27.00-50+ |

*Note: We typically use 1 sat/vbyte for transactions to minimize costs.*

### Key Takeaways

1. **Every real BRC-20 transfer needs L1 inscription** (~$0.28 at 1 sat/vbyte, ~$8-10 at 30 sat/vbyte)
2. **Total L1 cost at 1 sat/vbyte: ~$0.65 per user** (inscription + send + dust)
3. **At 1 sat/vbyte, dust limit (~$0.30) is 53% of total cost** - this is fixed and cannot be reduced
4. **L2 saves money on internal transfers**, but bridge-out still costs L1 fees
5. **Shadow Balance is essentially a centralized L2** (current system)
6. **User-pays model is most cost-effective** for Beem
7. **Only ~10-50% of users typically withdraw** to L1, so partial L1 saves money

### Verification: User's View of Real BRC-20

After receiving BRC-20 to their Taproot wallet, users can verify on:

| Explorer | URL | What They See |
|----------|-----|---------------|
| **Unisat** | unisat.io | BRC-20 balance, transfer history |
| **OrdScan** | ordiscan.com | Inscription details, ownership |
| **Ordinals.com** | ordinals.com | Inscription content |
| **OKX Explorer** | okx.com/ordinals | Balance, trading |

---

*Document generated: January 2026*
*Based on current BTC price: ~$91,756 USD*
