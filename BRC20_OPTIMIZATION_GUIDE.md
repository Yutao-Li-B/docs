# BRC-20 Inscription & Transfer Optimization Guide

> **Last Updated:** January 2026  
> **Network:** Bitcoin Mainnet  
> **Status:** Production-ready optimizations implemented

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [BRC-20 Protocol Overview](#brc-20-protocol-overview)
3. [Cost Breakdown](#cost-breakdown)
4. [Optimization Strategies](#optimization-strategies)
5. [What Can Be Batched](#what-can-be-batched)
6. [What Cannot Be Batched (and Why)](#what-cannot-be-batched-and-why)
7. [Attempted Optimizations That Don't Work](#attempted-optimizations-that-dont-work)
8. [Minting Experiments](#minting-experiments)
9. [Transaction Proof URLs](#transaction-proof-urls)
10. [Implementation Reference](#implementation-reference)

---

## Executive Summary

### Current Optimized Costs (Bitcoin Mainnet)

| Operation | Method | Fee Rate | Cost | Confirmation | Notes |
|-----------|--------|----------|------|--------------|-------|
| **Transfer to User** | Internal + Auto-Fee | 0.2 sat/vB | **~398 sats** | ~45 min | ✅ Best balance: cost + speed |
| **Transfer to User** | Internal + Auto-Fee | 0.15 sat/vB | **~382 sats** | ~2.5 hours | ✅ Max savings (time flexible) |
| **Transfer to User** | Internal + Auto-Fee | 0.4 sat/vB | **~465 sats** | ~10 min | Fast, 30% savings |
| **Transfer to User** | UniSat + Batched Send | 1 sat/vB | **~671 sats** | ~10 min | Fixed rate, reliable |
| **Mint (5-byte)** | UniSat (optimized) | 1 sat/vB | **679 sats** | ~10 min | `outputValue: 330` |

### Key Findings Summary

| # | Finding | Evidence | Section |
|---|---------|----------|---------|
| 1 | **~371 sats at 0.12 sat/vB** | 44% savings, confirmed working | [Experiment 10](#experiment-10-fee-rate-012-satvb--confirmed) |
| 2 | **~398 sats at 0.2 sat/vB** | 40% savings, confirmed working | [Experiment 8](#experiment-8-ultra-low-fee-rate-02-satvb--success) |
| 3 | **~671 sats with UniSat** | Fixed 1 sat/vB, reliable | [Current Implementation](#current-implementation-unisat--batched-sends) |
| 4 | **0.12 sat/vB is lowest confirmed** | 0.11 sat/vB rejected | [Experiment 10](#experiment-10-fee-rate-012-satvb--confirmed) |
| 5 | **Batching reveals FAILS** for BRC-20 | Only 1st inscription recognized | [Experiment 2](#experiment-2-internal-batched-minting-service-ytctw) |
| 6 | **Batching commits WORKS** | 85% commit cost reduction | [1→N Transactions](#1n-transactions-commit) |
| 7 | **Batching sends WORKS** | 42% send cost reduction | [N→N Transactions](#nn-transactions-batch-send) |
| 8 | **330 sats dust is hard floor** | Network rejects < 330 sats | [Experiment 6](#experiment-6-sub-dust-inscription-200-sats) |
| 9 | **~0.105 sat/vB is network minimum** | 0.11, 0.1 sat/vB rejected | [Experiment 10](#experiment-10-fee-rate-012-satvb--confirmed) |

### Experiments Conducted

| # | Experiment | Result | Link |
|---|------------|--------|------|
| 1 | UniSat 5-byte Mint (YTCTW) | ✅ Success | [Details](#experiment-1-unisat-5-byte-mint-ytctw) |
| 2 | Internal Batched Mint (YTCTW) | ❌ Failed (cannot batch reveals) | [Details](#experiment-2-internal-batched-minting-service-ytctw) |
| 3 | Internal Mint with Deploy UTXO (AAE26) | ⚠️ Learned | [Details](#experiment-3-internal-minting-service-aae26) |
| 4 | Transfer Inscriptions (SIGAI) | ❌ Failed (cannot batch reveals) | [Details](#experiment-4-batched-transfer-inscriptions-sigai) |
| 5 | Batched Send | ✅ Success | [Details](#experiment-5-batched-send-transfer-inscriptions-to-receiver) |
| 6 | Sub-Dust Inscription (200 sats) | ❌ Failed (network rejects) | [Details](#experiment-6-sub-dust-inscription-200-sats) |
| 7 | Sub-Sat Fee Rates (0.4 sat/vB) | ✅ Success (30% savings) | [Details](#experiment-7-sub-sat-fee-rates-04-satvb--success) |
| 8 | Ultra-Low Fee Rate (0.2 sat/vB) | ✅ Success (40% savings) | [Details](#experiment-8-ultra-low-fee-rate-02-satvb--success) |
| 9 | Fee Rate 0.15 sat/vB | ✅ Confirmed (43% savings) | [Details](#experiment-9-fee-rate-015-satvb--confirmed) |
| 10 | Fee Rate 0.12 sat/vB | ✅ Confirmed (44% savings) | [Details](#experiment-10-fee-rate-012-satvb--confirmed) |

### Current Implementation: UniSat + Batched Sends

Our production implementation uses **UniSat API for inscription creation** combined with **batched send transactions**:

| Step | Method | Cost |
|------|--------|------|
| Create inscription | UniSat API (`outputValue: 330`) | 341 sats (183 fee + 158 commit) |
| Inscription dust | Held by inscription | 330 sats |
| Batched send to user | Internal (batch of 20+) | ~106 sats |
| **Total per user settlement** | | **~671 sats** |

**Why this approach:**
- ✅ **No UTXO management** - UniSat handles commit/reveal complexity
- ✅ **No inscription tracking** - UniSat returns ready-to-send inscriptions  
- ✅ **Batched sends** - We batch the final send step ourselves (42% send cost reduction)
- ✅ **Reliable** - UniSat's infrastructure handles edge cases

**Alternative: Full Internal Inscription**

| Step | Method | Cost |
|------|--------|------|
| Batched commit | Internal | ~49 sats (batch of 20) |
| Individual reveal | Internal | 150 sats |
| Inscription dust | Held by inscription | 330 sats |
| Batched send | Internal | ~106 sats |
| **Total per user settlement** | | **~635 sats** |

**Savings:** ~36 sats per inscription (5% reduction)

**Trade-offs:**
- ❌ **UTXO management** - Must track and consolidate UTXOs regularly
- ❌ **Inscription tracking** - Must track inscription locations across reveals
- ❌ **Error handling** - Must handle stuck transactions, RBF, mempool issues
- ❌ **Operational complexity** - More infrastructure to maintain

**Recommendation:** The ~36 sats savings (~5%) does not justify the added operational complexity for most use cases. UniSat + batched sends is the pragmatic choice.

### What's NOT Possible (Tested & Failed)

| Optimization Attempt | Why It Fails | Section |
|---------------------|--------------|---------|
| Batch BRC-20 reveals | Indexer: 1 inscription per TX | [Experiment 2](#experiment-2-internal-batched-minting-service-ytctw) |
| Sub-dust outputs (200 sats) | Network rejects: "dust output must be 0-fee" | [Experiment 6](#experiment-6-sub-dust-inscription-200-sats) |
| Non-Taproot addresses | Inscriptions require P2TR | [Failed Optimizations](#attempted-optimizations-that-dont-work) |
| Skip send step (transfers) | Protocol requires 2-step | [BRC-20 Overview](#transfer-process-2-step-required) |
| Parallel 5-byte minting | Deploy UTXO bottleneck | [Critical Limitation](#️-critical-limitation-5-byte-minting-cannot-be-parallelized) |
| BRC-2.0 for cheaper transfers | Same cost as BRC-20 | [BRC-2.0 Analysis](#attempted-optimizations-that-dont-work) |

---

## BRC-20 Protocol Overview

### What is BRC-20?

BRC-20 is a fungible token standard built on Bitcoin Ordinals inscriptions. Tokens are created and transferred by inscribing JSON data onto satoshis.

### Token Types

| Type | Ticker Length | Minting | Deploy |
|------|---------------|---------|--------|
| 4-byte | 4 chars | Public | Standard |
| 5-byte | 5 chars | Deployer only | Standard |
| 6-byte (BRC-2.0) | 6 chars | Configurable | PSBT signing |

### Transfer Process (2-Step Required)

```
Step 1: CREATE transfer inscription (to yourself)
        → Locks your BRC-20 tokens to that inscription
        
Step 2: SEND the inscription to receiver
        → BRC-20 tokens transfer to receiver
```

**Why 2 steps?** The BRC-20 indexer validates that:
1. Transfer inscription was created at **sender's address** (proves ownership)
2. Inscription was then sent **from sender → receiver** (proves intentional transfer)

---

## Cost Breakdown

### Anatomy of a BRC-20 Transfer

| Step | Transaction | Size (vBytes) | Cost @ 1 sat/vB |
|------|-------------|---------------|-----------------|
| 1. Commit | Batched (1 TX for N inscriptions) | ~50/inscription | ~50 sats |
| 2. Reveal | Individual (1 TX per inscription) | ~150 | 150 sats |
| 3. Send | Batched (1 TX for N sends) | ~106/inscription | ~106 sats |
| 4. Dust | Inscription UTXO value | - | 330 sats |
| **TOTAL** | | | **~636 sats** |

### Cost by Batch Size

| Batch Size | Commit/each | Reveal/each | Send/each | Dust | **Total/each** |
|------------|-------------|-------------|-----------|------|----------------|
| 1 | 154 | 150 | 211 | 330 | 845 |
| 5 | 65 | 150 | 123 | 330 | 668 |
| 10 | 54 | 150 | 112 | 330 | 646 |
| 20 | 49 | 150 | 106 | 330 | 635 |
| 50 | 45 | 150 | 103 | 330 | 628 |
| 100 | 44 | 150 | 102 | 330 | 626 |

---

## Optimization Strategies

### ✅ Implemented Optimizations

#### 1. Batched Commit Transactions

Instead of creating separate commit TXs for each inscription, we batch all commit outputs into one transaction.

```
BEFORE: 5 inscriptions = 5 commit TXs = 5 × 154 sats = 770 sats
AFTER:  5 inscriptions = 1 commit TX  = 326 sats total = 65 sats each
SAVINGS: 85% reduction in commit costs
```

#### 2. Precision-Funded Commit Outputs

Each commit output is funded with exactly: `DUST_LIMIT + reveal_fee`

```javascript
const commitOutputValue = 330 + 150;  // 480 sats
```

This eliminates the need for a separate fee input in the reveal TX.

#### 3. Batched Send Transactions

Instead of sending inscriptions individually, we batch all sends into one TX.

```
BEFORE: 5 sends = 5 TXs = 5 × 211 sats = 1,055 sats
AFTER:  5 sends = 1 TX  = 613 sats total = 123 sats each
SAVINGS: 42% reduction in send costs
```

#### 4. Safe UTXO Selection

We filter out UTXOs that:
- Have inscription-like values (330 or 546 sats)
- Actually contain inscriptions (verified via UniSat API)

This prevents accidentally spending inscriptions as fees.

#### 5. Optimized Inscription Envelope

```
JSON: {"p":"brc-20","op":"transfer","tick":"SIGAI","amt":"100"}
Size: 57 bytes (no spaces, minimal separators)
Content-type: text/plain;charset=utf-8 (standard, compatible)
```

---

## What Can Be Batched

### ✅ Commit Transactions

**Works:** Multiple inscription commits in one transaction

```
1 TX with N outputs (one per inscription commit)
Each output goes to a pre-computed tweaked Taproot address
```

### ✅ Send Transactions

**Works:** Multiple inscription sends in one transaction

```
1 TX with:
  - N inscription inputs (one per inscription)
  - N receiver outputs (one per recipient)
  - 1 fee input + 1 change output
```

---

## What Cannot Be Batched (and Why)

### ❌ BRC-20 Reveal Transactions

**Does NOT work:** Multiple BRC-20 inscriptions in one reveal TX

#### The Problem

The BRC-20 indexer only recognizes **ONE BRC-20 inscription per transaction**.

```
Reveal TX with 5 BRC-20 inscriptions:
  - Inscription 0: ✅ Recognized as BRC-20
  - Inscription 1: ❌ Ignored by indexer
  - Inscription 2: ❌ Ignored by indexer
  - Inscription 3: ❌ Ignored by indexer
  - Inscription 4: ❌ Ignored by indexer
```

#### Proof

We tested batched reveals and only the first inscription was recognized:

- **Reveal TX:** [c0577ab1468eb718418ef52519a19ff218e7f096b5a8494870c27aa2589ca49f](https://mempool.space/tx/c0577ab1468eb718418ef52519a19ff218e7f096b5a8494870c27aa2589ca49f)
- **Result:** 5 inscriptions received, only 1 recognized as BRC-20

#### Why This Happens

The BRC-20 protocol specification states that only one BRC-20 operation is processed per transaction. This is a protocol-level limitation, not an implementation bug.

### ❌ 5-Byte Mint Parallelization

**Does NOT work:** Parallel minting of 5-byte tokens

#### The Problem

5-byte tokens require the **deploy inscription UTXO** to be included in each mint transaction. Since a UTXO can only be spent once per block, mints must be sequential.

```
Mint 1: Spends deploy UTXO at txid:0 → Creates new UTXO at mint1:N
Mint 2: Must wait for Mint 1, spends deploy UTXO at mint1:N → Creates mint2:N
Mint 3: Must wait for Mint 2, spends deploy UTXO at mint2:N → Creates mint3:N
```

---

## Attempted Optimizations That Don't Work

### ❌ Sub-Dust Commit Outputs

**Idea:** Use commit outputs below 330 sats to save on dust

**Why it fails:**
- Standard Bitcoin nodes reject transactions with outputs below dust limit
- Would require direct miner submission or package relay (not widely supported)
- Final inscription MUST be ≥330 sats to be spendable

### ❌ Non-Taproot Addresses for Inscriptions

**Idea:** Use P2WPKH (294 sat dust) instead of P2TR (330 sat dust)

**Why it fails:**
- Ordinals inscriptions **require Taproot** (P2TR)
- Inscription data is embedded in Taproot witness
- Non-Taproot addresses cannot hold inscriptions

| Address Type | Dust Limit | Can hold inscriptions? |
|--------------|------------|------------------------|
| P2WPKH | 294 sats | ❌ No |
| P2TR | 330 sats | ✅ Yes |

### ❌ Reveal Directly to Receiver (for BRC-20 transfers)

**Idea:** Skip the "hold inscription at sender" step

**Why it fails:**
- BRC-20 protocol requires 2-step process
- Indexer validates that sender owned tokens BEFORE transfer
- Direct reveal = receiver has inscription but never owned tokens = invalid

### ❌ Shorter Content-Type

**Idea:** Use `text/plain` instead of `text/plain;charset=utf-8`

**Why it's risky:**
- Saves only 14 bytes (~4 sats)
- Some indexers may not recognize the inscription
- Not worth the compatibility risk

### ❌ BRC-2.0 (6-byte) for Cheaper Transfers

**Idea:** Use BRC-20 6-byte for cost savings

**Reality:** BRC-2.0 offers features (activation height, minting control) but **same transfer cost**

> "The transfer process is exactly the same as standard BRC-20"  
> — [UniSat BRC-20 6-byte Guide](https://github.com/unisat-wallet/unisat-dev-docs/blob/master/open-api/brc20-6byte-guide.md)

---

## Minting Experiments

### Experiment 1: UniSat 5-Byte Mint (YTCTW)

**Token:** YTCTW (5-byte, selfMint)  
**Amount:** 150 tokens  
**Receiver:** `bc1pyx4hqjc2gxgn54agxqt9ahgd0smm62cnzrsgpqm0zt5p7yc3x4csne77kn`

#### UniSat Order Details

```json
{
  "orderId": "202601-BRC20-41QIZLPB",
  "status": "minted",
  "amount": 895,
  "outputValue": 546,
  "feeRate": 1,
  "minerFee": 349,
  "serviceFee": 0,
  "files": [{
    "filename": "{\"p\":\"brc-20\",\"op\":\"mint\",\"tick\":\"YTCTW\",\"amt\":\"150\"}",
    "size": 53,
    "inscriptionId": "36ab44889d7e06425348fef1166279e35449688719dc16a714e3d4835d160c4ai0"
  }]
}
```

#### Transaction URL

| Transaction | URL |
|-------------|-----|
| UniSat Reveal | [36ab44889d7e06425348fef1166279e35449688719dc16a714e3d4835d160c4a](https://mempool.space/tx/36ab44889d7e06425348fef1166279e35449688719dc16a714e3d4835d160c4a) |

#### Cost Breakdown (UniSat 5-byte Mint)

| Component | Default | Optimized |
|-----------|---------|-----------|
| Miner Fee | 349 sats | 349 sats |
| Output Value | 546 sats | **330 sats** ✅ |
| Service Fee | 0 sats | 0 sats |
| **Total** | **895 sats** | **679 sats** |

**Optimization:** UniSat API accepts `outputValue: 330` for Taproot addresses.

```javascript
// UniSat API - optimized 5-byte mint request
{
  "receiveAddress": "bc1p...",  // Must be Taproot (P2TR)
  "feeRate": 1,
  "outputValue": 330,           // ← Use 330 instead of default 546
  "brc20Ticker": "YTCTW",
  "brc20Amount": "150"
}
```

**Savings:** 216 sats per mint (24% reduction) by using 330 sat output.

---

### Experiment 2: Internal Batched Minting Service (YTCTW)

**Token:** YTCTW (5-byte, selfMint)  
**Amounts:** 100, 200, 300, 400, 500 tokens (batch of 5)  
**Receiver:** `bc1pyx4hqjc2gxgn54agxqt9ahgd0smm62cnzrsgpqm0zt5p7yc3x4csne77kn`  
**Goal:** Test if we could batch 5 mint inscriptions into a single reveal transaction

#### Transaction URLs

| Transaction | URL |
|-------------|-----|
| Commit TX | [8175f6f8d53a1292ecf227d1256f2e80a3d32cd580056c18aa30ae27d10eeb0a](https://mempool.space/tx/8175f6f8d53a1292ecf227d1256f2e80a3d32cd580056c18aa30ae27d10eeb0a) |
| Reveal TX (batched) | [50ddad5fe05014fe106d4deab41ed62d8b047edbe8f0201ea76388f5e83d5e90](https://mempool.space/tx/50ddad5fe05014fe106d4deab41ed62d8b047edbe8f0201ea76388f5e83d5e90) |

#### Result: ❌ FAILED - Cannot Batch Reveal Operations

**What We Tried:** Created 5 commit outputs in 1 TX, then revealed all 5 inscriptions in a single reveal TX.

**What Happened:** All 5 inscriptions were created on-chain, but only the FIRST inscription was recognized as BRC-20. The remaining 4 inscriptions existed as ordinals but had NO BRC-20 token balance associated.

**Root Cause:** BRC-20 indexer only recognizes **1 BRC-20 inscription per transaction**. This is a protocol-level limitation, not an implementation bug.

```
Reveal TX with 5 BRC-20 mint inscriptions:
  - Inscription 0: ✅ Recognized as BRC-20 mint (100 YTCTW)
  - Inscription 1: ❌ Ignored by indexer (200 YTCTW lost)
  - Inscription 2: ❌ Ignored by indexer (300 YTCTW lost)
  - Inscription 3: ❌ Ignored by indexer (400 YTCTW lost)
  - Inscription 4: ❌ Ignored by indexer (500 YTCTW lost)
```

**Critical Lesson Learned:** 
- ❌ **Cannot batch reveal transactions** for BRC-20 operations (mints or transfers)
- ✅ **Can batch commit transactions** (1 TX with N commit outputs)
- ✅ **Can batch send transactions** (1 TX with N inscription inputs → N receiver outputs)
- Each BRC-20 reveal must be in its **own separate transaction**

---

### Experiment 3: Internal Minting Service (AAE26)

**Token:** AAE26 (5-byte, selfMint)  
**Amounts:** 100, 200, 300, 400, 500 tokens (batch of 5)  
**Deploy Inscription:** [`b2d19d4d03e0960ffe8523df7e165b4908f18942e90936d2f50e6fc151d4c329i0`](https://mempool.space/tx/b2d19d4d03e0960ffe8523df7e165b4908f18942e90936d2f50e6fc151d4c329)

#### Key Finding: Deploy UTXO Requirement

5-byte minting requires the **deploy inscription UTXO** to be spent in each mint transaction:

```
Mint TX Structure:
  Inputs:
    0: Inscription reveal (commit output)
    1: Deploy inscription UTXO ← REQUIRED for authorization
    2: Fee input
  Outputs:
    0: Minted inscription to receiver
    1: Deploy inscription pass-through ← Returns to deployer
    2: Change
```

**Implication:** Mints must be **sequential** (can't parallelize). The deploy UTXO can only be in one place at a time.

---

### Experiment 4: Batched Transfer Inscriptions (SIGAI)

**Token:** SIGAI  
**Amounts:** 100, 200, 300, 400, 500 tokens (batch of 5)  
**Receiver (inscription holder):** `bc1pauxwzzh78js3r7gzn2ckf26qp9gly5wqmsl5k03yug3cfsk0xjrq5dyp2m`  
**Goal:** Test if transfer inscriptions could be batched (since they don't need deploy UTXO)

#### Transaction URLs

| Transaction | URL |
|-------------|-----|
| Reveal TX (batched) | [c0577ab1468eb718418ef52519a19ff218e7f096b5a8494870c27aa2589ca49f](https://mempool.space/tx/c0577ab1468eb718418ef52519a19ff218e7f096b5a8494870c27aa2589ca49f) |

#### Result: ❌ FAILED - Cannot Batch Reveal Operations

**What We Tried:** Created 5 transfer inscription reveals in a single transaction (no deploy UTXO needed for transfers).

**What Happened:** Same result as Experiment 2 - only the FIRST transfer inscription was recognized as BRC-20.

```
Reveal TX with 5 BRC-20 transfer inscriptions:
  - Inscription 0: ✅ Recognized as BRC-20 transfer (100 SIGAI)
  - Inscription 1-4: ❌ Ignored by indexer
```

**Stats from TX:**
- Total weight: 3264 WU
- Total vSize: 816 vBytes
- Per inscription: 163 vBytes
- Fee: 553 sats (111 sats per inscription)

**Confirmed:** The "1 BRC-20 inscription per transaction" limitation applies to ALL BRC-20 operations (mints AND transfers), regardless of whether deploy UTXO is involved.

---

### Experiment 5: Batched Send (Transfer Inscriptions to Receiver)

**Inscriptions:** 5 transfer inscriptions  
**Final Receiver:** `bc1pyx4hqjc2gxgn54agxqt9ahgd0smm62cnzrsgpqm0zt5p7yc3x4csne77kn`

#### Transaction Structure

```
Send TX:
  Inputs: 5 inscription UTXOs + 1 fee UTXO
  Outputs: 5 receiver outputs + 1 change
```

#### Result: ✅ SUCCESS

Batched sending WORKS. All 5 inscriptions sent in 1 transaction.

---

### Experiment 6: Sub-Dust Inscription (200 sats)

**Token:** SIGAI  
**Amount:** 100 tokens  
**Dust limit tested:** 200 sats (vs standard 330 sats)  
**Final Receiver:** `bc1pyx4hqjc2gxgn54agxqt9ahgd0smm62cnzrsgpqm0zt5p7yc3x4csne77kn`  
**Goal:** Test if we can reduce inscription UTXO value below Taproot dust limit to save 130 sats

#### Transaction URLs

| Transaction | Status | URL |
|-------------|--------|-----|
| Commit TX 1 | ✅ Broadcast | [9973f5930d6e4309b08d29bfa4ffd4083ab6bc890a5103b8cc86b9ed3445c445](https://mempool.space/tx/9973f5930d6e4309b08d29bfa4ffd4083ab6bc890a5103b8cc86b9ed3445c445) |
| Commit TX 2 | ✅ Broadcast | [bb7623376638e9a6f539f162a3d132f75abe29cd5149c193064e86ecbfda7ab4](https://mempool.space/tx/bb7623376638e9a6f539f162a3d132f75abe29cd5149c193064e86ecbfda7ab4) |
| Reveal TX | ❌ REJECTED | N/A - Node rejected |

#### Result: ❌ FAILED - Network Rejects Sub-Dust Outputs

**What We Tried:** Created commit output with 350 sats (200 dust + 150 reveal fee), then tried to reveal with 200 sat output.

**Error Response from Bitcoin Network:**

```
sendrawtransaction RPC error: {
  "code": -26,
  "message": "dust, tx with dust output must be 0-fee"
}
```

**Error Analysis:**

| Aspect | Finding |
|--------|---------|
| **Error code** | `-26` (RPC_VERIFY_REJECTED) - Mempool policy rejection |
| **Root cause** | 200 sats < 330 sats (Taproot dust limit) |
| **Network response** | "dust output must be 0-fee" |
| **Meaning** | Sub-dust only allowed with ephemeral dust (0-fee, spent same block) |

**What is Ephemeral Dust?**

Bitcoin Core 28+ introduced "ephemeral dust" - outputs below dust limit that are:
1. Part of a 0-fee transaction
2. Spent in the same block via package relay
3. Effectively never exist as UTXOs

**Why We Can't Use Ephemeral Dust:**
- ❌ Requires Bitcoin Core 28+ (not universally deployed)
- ❌ Requires package relay (not widely supported by mempool.space)
- ❌ Transaction must be 0-fee (requires child to pay)
- ❌ Complex coordination with miners

**Conclusion:**

```
┌─────────────────────────────────────────────────────────────┐
│  330 SATS IS THE HARD FLOOR FOR INSCRIPTION UTXOS           │
│                                                             │
│  This is enforced by Bitcoin network policy, not BRC-20.    │
│  No practical way to reduce this on current mainnet.        │
└─────────────────────────────────────────────────────────────┘
```

**Stuck UTXOs:** The two commit TXs created 350 sat outputs that will confirm and need to be spent (recovered) later since the reveals failed.

---

### Experiment 7: Sub-Sat Fee Rates (0.4 sat/vB) ✅ SUCCESS

**Token:** SIGAI  
**Amounts:** 100, 200, 300, 400, 500 tokens (batch of 5)  
**Fee Rate:** 0.4 sat/vB (dynamic, based on mempool minimum × 2, capped)  
**Final Receiver:** `bc1pyx4hqjc2gxgn54agxqt9ahgd0smm62cnzrsgpqm0zt5p7yc3x4csne77kn`  
**Goal:** Test if sub-1 sat/vB fee rates are accepted by the network

#### Transaction URLs

| Step | TX ID | Fee | vSize | Fee Rate |
|------|-------|-----|-------|----------|
| **Commit** | [8b1d2095356533264ed80422b7f4e10a890ffa52a22fd68c80429946a7eb8066](https://mempool.space/tx/8b1d2095356533264ed80422b7f4e10a890ffa52a22fd68c80429946a7eb8066) | 131 sats | 326 vB | 0.40 sat/vB |
| **Reveal 1** | [253493bd5fba531f43cfcf0c3e39ff22d5858e4ecf95dd211404b8aed49579ca](https://mempool.space/tx/253493bd5fba531f43cfcf0c3e39ff22d5858e4ecf95dd211404b8aed49579ca) | 60 sats | 152 vB | 0.39 sat/vB |
| **Reveal 2** | [f9c8adfc5233a5e6c8086ac4740e49b388cc68e3cbd0db13b3acbff5294a73a3](https://mempool.space/tx/f9c8adfc5233a5e6c8086ac4740e49b388cc68e3cbd0db13b3acbff5294a73a3) | 60 sats | 152 vB | 0.39 sat/vB |
| **Reveal 3** | [d47ccd47503457acec59063f6d7ea789fd064c9875e2e0ad590001fe6526cd17](https://mempool.space/tx/d47ccd47503457acec59063f6d7ea789fd064c9875e2e0ad590001fe6526cd17) | 60 sats | 152 vB | 0.39 sat/vB |
| **Reveal 4** | [39a46b2a336e070c0d1076568749157d832dd0c7346aae285087fce6dda72cca](https://mempool.space/tx/39a46b2a336e070c0d1076568749157d832dd0c7346aae285087fce6dda72cca) | 60 sats | 152 vB | 0.39 sat/vB |
| **Reveal 5** | [f5300b25de4c9dcd9da089f8556badf04093cbb097e193778602bbbabb9d4430](https://mempool.space/tx/f5300b25de4c9dcd9da089f8556badf04093cbb097e193778602bbbabb9d4430) | 60 sats | 152 vB | 0.39 sat/vB |
| **Batch Send** | [7f3b912ca3d2f42ee159bb248533e104f73ce895413f96f8dbc3ace6e6c37f3f](https://mempool.space/tx/7f3b912ca3d2f42ee159bb248533e104f73ce895413f96f8dbc3ace6e6c37f3f) | 244 sats | 614 vB | 0.40 sat/vB |

#### Result: ✅ SUCCESS - Sub-Sat Fee Rates Work!

All 7 transactions (1 commit + 5 reveals + 1 batch send) were accepted by the network at ~0.4 sat/vB!

**Cost Breakdown (5 inscriptions @ 0.4 sat/vB):**

| Component | Total | Per Inscription |
|-----------|-------|-----------------|
| Commit | 131 sats | 26.2 sats |
| Reveals (5×60) | 300 sats | 60 sats |
| Batch Send | 244 sats | 48.8 sats |
| Dust (5×330) | 1,650 sats | 330 sats |
| **TOTAL** | **2,325 sats** | **465 sats** |

#### Sub-Sat Fee Rate Comparison (Batch of 5)

| Fee Rate | Commit | Reveal | Send | Dust | **Total/each** | vs 1 sat/vB |
|----------|--------|--------|------|------|----------------|-------------|
| **1.0 sat/vB** | 65 sats | 150 sats | 123 sats | 330 sats | **668 sats** | - |
| **0.4 sat/vB** | 26 sats | 60 sats | 49 sats | 330 sats | **465 sats** | **-30%** |
| **0.2 sat/vB** | 13 sats | 30 sats | 25 sats | 330 sats | **398 sats** | **-40%** |
| **0.1 sat/vB** | 7 sats | 15 sats | 12 sats | 330 sats | **364 sats** | **-45%** |

#### Cost Calculator by Fee Rate

```
Formula (batch of N):
  Commit:  (111 + 43×N) × feeRate / N
  Reveal:  150 × feeRate
  Send:    (111 + 100.5×N) × feeRate / N
  Dust:    330 (fixed)
  
  Total = Commit + Reveal + Send + Dust
```

**Single inscription costs by fee rate:**

| Fee Rate | Commit | Reveal | Send | Dust | **Total** |
|----------|--------|--------|------|------|-----------|
| 1.0 sat/vB | 154 sats | 150 sats | 212 sats | 330 sats | **846 sats** |
| 0.4 sat/vB | 62 sats | 60 sats | 85 sats | 330 sats | **537 sats** |
| 0.2 sat/vB | 31 sats | 30 sats | 42 sats | 330 sats | **433 sats** |
| 0.1 sat/vB | 15 sats | 15 sats | 21 sats | 330 sats | **381 sats** |

**Batch of 20 costs by fee rate:**

| Fee Rate | Commit | Reveal | Send | Dust | **Total/each** |
|----------|--------|--------|------|------|----------------|
| 1.0 sat/vB | 49 sats | 150 sats | 106 sats | 330 sats | **635 sats** |
| 0.4 sat/vB | 20 sats | 60 sats | 42 sats | 330 sats | **452 sats** |
| 0.2 sat/vB | 10 sats | 30 sats | 21 sats | 330 sats | **391 sats** |
| 0.1 sat/vB | 5 sats | 15 sats | 11 sats | 330 sats | **361 sats** |

#### Key Findings

1. **Sub-sat fee rates ARE accepted** by Bitcoin nodes during low congestion
2. **The minimum relay fee is not strictly 1 sat/vB** - it depends on mempool conditions
3. **Dust (330 sats) becomes the dominant cost** at low fee rates
4. **At 0.1 sat/vB, dust is ~91% of total cost** (330/361)
5. **Dynamic fee rates can save 30-45%** compared to fixed 1 sat/vB

#### Implementation

```javascript
// Fetch mempool recommended fees and use 2× minimum, capped
async function getOptimizedFeeRate(maxRate = 0.4) {
  const fees = await fetchRecommendedFees();  // mempool.space API
  const baseFee = fees.minimumFee;
  let optimizedFee = baseFee * 2;             // 2× for safety margin
  optimizedFee = Math.min(optimizedFee, maxRate);  // Cap at maxRate
  optimizedFee = Math.max(optimizedFee, 0.1);      // Minimum 0.1 sat/vB
  return optimizedFee;
}

// Usage
node batch_inscriber.js --op transfer-optimized --auto-fee --max-fee 0.4 --broadcast
```

---

### Experiment 8: Ultra-Low Fee Rate (0.2 sat/vB) ✅ SUCCESS

**Token:** SIGAI  
**Amounts:** 1, 2, 3, 4, 5 tokens (batch of 5)  
**Fee Rate:** 0.2 sat/vB (fixed)  
**Final Receiver:** `bc1pyx4hqjc2gxgn54agxqt9ahgd0smm62cnzrsgpqm0zt5p7yc3x4csne77kn`  
**Goal:** Test the lowest practical fee rate for production use

#### Transaction URLs

| Step | TX ID | Fee | vSize | Fee Rate |
|------|-------|-----|-------|----------|
| **Commit** | [37cda0490ecf825ae6538bdef13c74654ad99e420efa858dd285338be40a3778](https://mempool.space/tx/37cda0490ecf825ae6538bdef13c74654ad99e420efa858dd285338be40a3778) | 66 sats | 326 vB | 0.20 sat/vB |
| **Reveal 1** | [334f3c42c15eb50b6b98ffad36b8a1bbf26c0c0905188a91092277e8fc14dfe6](https://mempool.space/tx/334f3c42c15eb50b6b98ffad36b8a1bbf26c0c0905188a91092277e8fc14dfe6) | 30 sats | 151 vB | 0.20 sat/vB |
| **Reveal 2** | [7bc4fc68e2aa6d7a427a1d7fe0938ce8a832205c5b8b773a22913d1aae537b2a](https://mempool.space/tx/7bc4fc68e2aa6d7a427a1d7fe0938ce8a832205c5b8b773a22913d1aae537b2a) | 30 sats | 151 vB | 0.20 sat/vB |
| **Reveal 3** | [c2e34d0fb59939f0e7b1c16505b69f89027b36accb8473c00a5a258bda64ffbe](https://mempool.space/tx/c2e34d0fb59939f0e7b1c16505b69f89027b36accb8473c00a5a258bda64ffbe) | 30 sats | 151 vB | 0.20 sat/vB |
| **Reveal 4** | [d62634770d664debdc8497423a6dc0d4d0a9a5a933239a048c948d5cdc69665c](https://mempool.space/tx/d62634770d664debdc8497423a6dc0d4d0a9a5a933239a048c948d5cdc69665c) | 30 sats | 151 vB | 0.20 sat/vB |
| **Reveal 5** | [f7fe38a87c7223883192c024c6d0528c8545e27afcd2188abc851351ca9a2053](https://mempool.space/tx/f7fe38a87c7223883192c024c6d0528c8545e27afcd2188abc851351ca9a2053) | 30 sats | 151 vB | 0.20 sat/vB |
| **Batch Send** | [69e45098b9c7cbadc75e7496e0c77e4c375d629a21854ceb36b8a521836cfe36](https://mempool.space/tx/69e45098b9c7cbadc75e7496e0c77e4c375d629a21854ceb36b8a521836cfe36) | 122 sats | 614 vB | 0.20 sat/vB |

#### Result: ✅ SUCCESS

**Cost Breakdown (5 inscriptions @ 0.2 sat/vB):**

| Component | Total | Per Inscription |
|-----------|-------|-----------------|
| Commit | 66 sats | 13.2 sats |
| Reveals (5×30) | 150 sats | 30 sats |
| Batch Send | 122 sats | 24.4 sats |
| Dust (5×330) | 1,650 sats | 330 sats |
| **TOTAL** | **1,988 sats** | **398 sats** |

#### Comparison: All Tested Fee Rates

| Fee Rate | Status | Fees/each | Dust | **Total/each** | vs UniSat | Confirmation Time |
|----------|--------|-----------|------|----------------|-----------|-------------------|
| 1.0 sat/vB | ✅ Confirmed | 338 sats | 330 sats | 668 sats | - | ~10 min (1 block) |
| 0.4 sat/vB | ✅ Confirmed | 135 sats | 330 sats | 465 sats | -30% | ~10 min (1 block) |
| **🎯 0.2 sat/vB** | ✅ **Confirmed** | **68 sats** | **330 sats** | **398 sats** | **-40%** | **~45 min (5 blocks)** |
| **🎯 0.15 sat/vB** | ✅ **Confirmed** | **51 sats** | **330 sats** | **382 sats** | **-43%** | **~2.5 hours (11 blocks)** |
| 0.12 sat/vB | ✅ Confirmed | 41 sats | 330 sats | 371 sats | -44% | ~2.5 hours (11 blocks) |
| 0.11 sat/vB | ❌ Rejected | - | - | - | Too low | - |
| 0.1 sat/vB | ❌ Rejected | - | - | - | Too low | - |

> **🎯 Recommended Fee Rates:**
> - **0.2 sat/vB** - Best balance of cost (40% savings) and speed (~45 min)
> - **0.15 sat/vB** - Maximum savings (43%) when time is not critical (~2.5 hours)

#### Confirmation Time Analysis

| Fee Rate | Block Confirmed | Blocks Waited | Time Waited | Trade-off |
|----------|-----------------|---------------|-------------|-----------|
| 0.4 sat/vB | 934208 | ~1 | ~10 min | Fast, 30% savings |
| **🎯 0.2 sat/vB** | **934213** | **~5** | **~45 min** | **Best balance: speed + savings** |
| **🎯 0.15 sat/vB** | **934224** | **~11** | **~2.5 hours** | **Max savings when time flexible** |
| 0.12 sat/vB | 934224 | ~11 | ~2.5 hours | Diminishing returns (only +1%) |

**Key Insight:** Below 0.2 sat/vB, confirmation times increase significantly. The extra 3% savings (398→382 sats) comes at the cost of ~2 hours longer confirmation time. Choose based on urgency:
- **Time-sensitive:** Use 0.2 sat/vB (40% savings, ~45 min)
- **Time-flexible:** Use 0.15 sat/vB (43% savings, ~2.5 hours)

#### Key Findings

1. **0.2 sat/vB is the recommended minimum** for reliable confirmation
2. **Dust (330 sats) is 83% of total cost** at this fee rate
3. **40% savings** compared to UniSat's fixed 1 sat/vB pricing
4. Lower rates (0.1-0.12 sat/vB) may work but confirmation times are unpredictable

#### Network Minimum Discovery

Through testing, we found the true network minimum relay fee:

| Fee Rate | Result | Error Message |
|----------|--------|---------------|
| 0.2 sat/vB | ✅ Accepted | - |
| 0.12 sat/vB | ✅ Accepted (pending) | - |
| 0.11 sat/vB | ❌ Rejected | "min relay fee not met, 36 < 39" |
| 0.1 sat/vB | ❌ Rejected | "min relay fee not met, 15 < 16" |

**True minimum:** ~0.105 sat/vB (or 16 sats minimum per transaction)

---

### Experiment 9: Fee Rate 0.15 sat/vB ✅ CONFIRMED

**Token:** SIGAI  
**Amounts:** 11, 22, 33, 44, 55 tokens (batch of 5)  
**Fee Rate:** 0.15 sat/vB  
**Final Receiver:** `bc1pyx4hqjc2gxgn54agxqt9ahgd0smm62cnzrsgpqm0zt5p7yc3x4csne77kn`  
**Block Confirmed:** 934224  
**Confirmation Time:** ~2.5 hours (~11 blocks waited)

#### Transaction URLs

| Step | TX ID | Fee | Status |
|------|-------|-----|--------|
| **Commit** | [84ada0eac6901fc38bdf36b92fe3b96ef98d5a8db1b0f9f854e818e37ab15542](https://mempool.space/tx/84ada0eac6901fc38bdf36b92fe3b96ef98d5a8db1b0f9f854e818e37ab15542) | 49 sats | ✅ Confirmed |
| **Reveal 1** | [1c10ccd0f44d882c8a9b71513f434a2ac75d4d8736b605b5dc3b8aabbbbc2908](https://mempool.space/tx/1c10ccd0f44d882c8a9b71513f434a2ac75d4d8736b605b5dc3b8aabbbbc2908) | 23 sats | ✅ Confirmed |
| **Reveal 2** | [f6baa69bbd3b8e6fb9cee8a2c041e7cf328ef94a5f8dd575ecaffb7495ef0279](https://mempool.space/tx/f6baa69bbd3b8e6fb9cee8a2c041e7cf328ef94a5f8dd575ecaffb7495ef0279) | 23 sats | ✅ Confirmed |
| **Reveal 3** | [67e56d9ba8ba7b700fa2cd027279010fb50f244e80cc5fcaa83af0317ffb12a7](https://mempool.space/tx/67e56d9ba8ba7b700fa2cd027279010fb50f244e80cc5fcaa83af0317ffb12a7) | 23 sats | ✅ Confirmed |
| **Reveal 4** | [b3d1f538d3dec46f67994120d21601aca6e1beed514b3e73d0290e71e71834ee](https://mempool.space/tx/b3d1f538d3dec46f67994120d21601aca6e1beed514b3e73d0290e71e71834ee) | 23 sats | ✅ Confirmed |
| **Reveal 5** | [0e9edf157c5fbe0a852b278d1074bc6f3302ed9dc6353a347d441044d0353e5e](https://mempool.space/tx/0e9edf157c5fbe0a852b278d1074bc6f3302ed9dc6353a347d441044d0353e5e) | 23 sats | ✅ Confirmed |
| **Batch Send** | [ca8f893a733eb1f6d9ee7f08387a6f24bc62660730e31a3501ba967a468add77](https://mempool.space/tx/ca8f893a733eb1f6d9ee7f08387a6f24bc62660730e31a3501ba967a468add77) | 92 sats | ✅ Confirmed |

#### Cost: 382 sats per inscription (43% savings vs UniSat)

---

### Experiment 10: Fee Rate 0.12 sat/vB ✅ CONFIRMED

**Token:** SIGAI  
**Amounts:** 10, 20, 30, 40, 50 tokens (batch of 5)  
**Fee Rate:** 0.12 sat/vB  
**Final Receiver:** `bc1pyx4hqjc2gxgn54agxqt9ahgd0smm62cnzrsgpqm0zt5p7yc3x4csne77kn`  
**Block Confirmed:** 934224  
**Confirmation Time:** ~2.5 hours (~11 blocks waited)

#### Transaction URLs

| Step | TX ID | Fee | Status |
|------|-------|-----|--------|
| **Commit** | [5eb2bdc82aafee600d0ccd5e13e22718db0d9b9d0894a62324deb57974d0ac77](https://mempool.space/tx/5eb2bdc82aafee600d0ccd5e13e22718db0d9b9d0894a62324deb57974d0ac77) | 40 sats | ✅ Confirmed |
| **Reveal 1** | [4e8c126eaa6b9e4027c92cd31fc371fa5ad61cef41ffc29e4294b99bfaea2dc6](https://mempool.space/tx/4e8c126eaa6b9e4027c92cd31fc371fa5ad61cef41ffc29e4294b99bfaea2dc6) | 18 sats | ✅ Confirmed |
| **Reveal 2** | [ca1d2136b17f16d5159a438381f1ea1c5ef0c5589d14dc48f3c3a92b01fdeb0f](https://mempool.space/tx/ca1d2136b17f16d5159a438381f1ea1c5ef0c5589d14dc48f3c3a92b01fdeb0f) | 18 sats | ✅ Confirmed |
| **Reveal 3** | [7e9612f078c041fe05458f3704ab6299193a1efa878a35379019cb2f61fe9bd0](https://mempool.space/tx/7e9612f078c041fe05458f3704ab6299193a1efa878a35379019cb2f61fe9bd0) | 18 sats | ✅ Confirmed |
| **Reveal 4** | [2570e301746b8930e7d51e96d4d41a7dc16f96bf87c22f2211f45b59551b18b9](https://mempool.space/tx/2570e301746b8930e7d51e96d4d41a7dc16f96bf87c22f2211f45b59551b18b9) | 18 sats | ✅ Confirmed |
| **Reveal 5** | [e688d2a0809b8b9ff5e6e40a99d32598c55e3835f7e98e085ff0a530f49d3477](https://mempool.space/tx/e688d2a0809b8b9ff5e6e40a99d32598c55e3835f7e98e085ff0a530f49d3477) | 18 sats | ✅ Confirmed |
| **Batch Send** | [78cbae8dbd465daeffe3079c639872439fd7e49140579e9be75003bef4c5592b](https://mempool.space/tx/78cbae8dbd465daeffe3079c639872439fd7e49140579e9be75003bef4c5592b) | 74 sats | ✅ Confirmed |

#### Cost: 371 sats per inscription (44% savings vs UniSat)

**Note:** 0.12 sat/vB is very close to the network minimum (~0.105 sat/vB). Lower rates (0.11, 0.1 sat/vB) were rejected.

---

### Minting Cost Comparison

| Method | Cost per Mint | Notes |
|--------|---------------|-------|
| **UniSat 5-byte (default)** | 895 sats | 546 sat output, 349 fee |
| **UniSat 5-byte (optimized)** | **679 sats** | `outputValue: 330` ✅ |
| **Internal (batched commits, individual reveals)** | 635 sats | Batch of 10+ commits, each revealed separately |
| **Internal (fully batched - FAILED)** | N/A | Cannot batch reveals - only 1st recognized |

**Best Option for UniSat API:** Always set `outputValue: 330` for Taproot addresses to save 216 sats per mint.

**Important:** Internal minting can only batch the commit step. Each reveal must be a separate transaction due to BRC-20 protocol limitations (see [Experiment 2](#experiment-2-internal-batched-minting-service-ytctw)).

### Mint vs Transfer Comparison

| Operation | Reveal Size | Deploy UTXO? | Parallelizable? | Cost |
|-----------|-------------|--------------|-----------------|------|
| **Mint (5-byte)** | ~251 vBytes | ✅ Required | ❌ No | 626-735 sats |
| **Transfer** | ~150 vBytes | ❌ Not needed | ✅ Yes | 626-668 sats |

**Why Mint costs more:**
- Mint reveal includes deploy UTXO input (+57.5 vBytes)
- Mint reveal includes deploy UTXO output (+43 vBytes)
- Extra ~100 vBytes per mint

**Why Mint is still cheaper overall:**
- Mint goes directly to receiver (no send step needed)
- Transfer requires separate send transaction

---

### ⚠️ Critical Limitation: 5-Byte Minting Cannot Be Parallelized

#### The Problem

5-byte BRC-20 tokens require the **deploy inscription UTXO** to be spent in every mint transaction. This creates a **sequential bottleneck**:

```
Deploy UTXO Location Timeline:
  
  Block 1: Deploy UTXO at txid_deploy:0
              ↓
           Mint TX 1 spends it
              ↓
  Block 2: Deploy UTXO now at mint1_txid:1
              ↓
           Mint TX 2 spends it
              ↓
  Block 3: Deploy UTXO now at mint2_txid:1
              ↓
           ... and so on
```

#### Why This Matters

| Scenario | Impact |
|----------|--------|
| **Low fees during congestion** | Mint TX stuck in mempool for hours/days |
| **Stuck mint TX** | ALL subsequent mints blocked until it confirms |
| **High demand** | Only ~144 mints/day possible (1 per block) |
| **Multiple workers** | Cannot run parallel minting services |

#### Example: Stuck Transaction Scenario

```
Timeline:
  
  00:00 - Mint TX 1 broadcast with 1 sat/vB fee
  00:10 - Network congestion spikes, 1 sat/vB TXs not confirming
  
  00:15 - User requests Mint TX 2
          ❌ BLOCKED - Mint TX 1 still unconfirmed
          
  06:00 - User requests Mint TX 3
          ❌ BLOCKED - Mint TX 1 still unconfirmed
          
  12:00 - Mint TX 1 finally confirms
          ✅ Now Mint TX 2 can be created
          
Result: 12 hours of minting blocked by single slow TX
```

#### Mitigation Strategies

| Strategy | Pros | Cons |
|----------|------|------|
| **Higher fee rate** | Faster confirmation | More expensive |
| **RBF (Replace-By-Fee)** | Can bump stuck TXs | Requires monitoring |
| **CPFP (Child-Pays-For-Parent)** | Can accelerate stuck TXs | Complex, uses more sats |
| **Use transfers instead** | Parallelizable | Requires pre-minted tokens |

#### Architectural Recommendation

For high-volume operations, consider this hybrid approach:

```
INITIAL SETUP (sequential, one-time):
  Mint large batch to treasury address
  e.g., Mint 1,000,000 tokens to deployer wallet

ONGOING OPERATIONS (parallel, scalable):
  Transfer from treasury to users
  ✅ No deploy UTXO needed
  ✅ Can run multiple workers
  ✅ Can batch sends
```

#### Code Implementation Note

The deploy UTXO must be fetched dynamically before each mint:

```javascript
// Fetch current deploy UTXO location (changes after each mint)
const deployUtxo = await fetchDeployInscriptionUTXO(DEPLOY_INSCRIPTION_ID);

// Build mint TX with deploy UTXO
const revealTx = await buildRevealTx(
  commitTxid,
  inscriptionData,
  receiver,
  deployUtxo,  // ← Must be current location
  feeUtxo
);
```

---

### Reinscription Issue (Experiment Finding)

During transfer inscription testing, we encountered "reinscription" errors:

**Problem:** Some inscriptions showed ♻️ tag on ordinals.com and weren't recognized as BRC-20.

**Root Cause:** We accidentally used UTXOs that already contained inscriptions as fee inputs. Creating a new inscription on a satoshi that already has one = "reinscription" = invalid BRC-20.

**Fix Implemented:**

```javascript
async function fetchSafeUTXOs(address) {
  const allUtxos = await fetchUTXOs(address);
  
  // Filter 1: Skip inscription-like values (330, 546 sats)
  let candidates = allUtxos.filter(utxo => !isLikelyInscription(utxo.value));
  
  // Filter 2: Verify with UniSat API that UTXO has no inscriptions
  for (const utxo of candidates) {
    const hasInscriptions = await utxoHasInscriptions(utxo.txid, utxo.vout);
    if (!hasInscriptions) {
      safeUtxos.push(utxo);
    }
  }
  
  return safeUtxos;
}
```

---

## Transaction Proof URLs

### Our Batched Reveal Test

| Transaction | URL |
|-------------|-----|
| Commit TX | [8175f6f8d53a1292ecf227d1256f2e80a3d32cd580056c18aa30ae27d10eeb0a](https://mempool.space/tx/8175f6f8d53a1292ecf227d1256f2e80a3d32cd580056c18aa30ae27d10eeb0a) |
| Reveal TX | [50ddad5fe05014fe106d4deab41ed62d8b047edbe8f0201ea76388f5e83d5e90](https://mempool.space/tx/50ddad5fe05014fe106d4deab41ed62d8b047edbe8f0201ea76388f5e83d5e90) |

**Result:** Inscriptions received but not recognized as BRC-20 (proved batched reveals don't work)

### Transfer Inscriptions Test

| Transaction | URL |
|-------------|-----|
| Reveal TX | [c0577ab1468eb718418ef52519a19ff218e7f096b5a8494870c27aa2589ca49f](https://mempool.space/tx/c0577ab1468eb718418ef52519a19ff218e7f096b5a8494870c27aa2589ca49f) |

**Stats:**
- 5 inscriptions in 1 TX
- 816 vBytes total (163 vBytes per inscription)
- Only first inscription recognized as BRC-20

### UniSat Reference Transaction

| Transaction | URL |
|-------------|-----|
| UniSat Reveal | [854f03dbd0e96b80bed6fb2827668589f007001670981a5bf11c0b89167e636f](https://mempool.space/tx/854f03dbd0e96b80bed6fb2827668589f007001670981a5bf11c0b89167e636f) |

**Stats:**
- 152 vBytes
- 513 sats input → 330 sats output + 183 fee
- Precision-funded commit output

---

## Implementation Reference

### Files

| File | Purpose |
|------|---------|
| `src/js/batch_inscriber.js` | Main inscription module |
| `src/js/send_inscription.js` | Single inscription sender |
| `src/api/unisat_router.py` | UniSat API integration |

### Key Functions

```javascript
// Optimized flow: batched commit + individual reveals + batched send
batchTransferOptimized(ticker, amounts, receivers, feeRate, broadcast)

// Build precision-funded commit TX
buildBatchedCommitTx(inscriptionData, feeRate)

// Build minimal reveal TX (1 input, 1 output)
buildSingleRevealTx(commitTxid, commitVout, inscriptionData, receiver, commitOutputValue, feeRate)

// Batch send inscriptions
batchSendInscriptions(inscriptionIds, receivers, feeRate, broadcast)
```

### Usage

```bash
# Optimized BRC-20 transfer flow
node src/js/batch_inscriber.js --op transfer-optimized \
  --ticker SIGAI \
  --amount "100,200,300,400,500" \
  --receivers "bc1p...,bc1p...,bc1p...,bc1p...,bc1p..." \
  --broadcast
```

---

## Batching Optimization Calculators

### Transaction Type Classification

| Type | Structure | Example | Batching Benefit |
|------|-----------|---------|------------------|
| **1→N** | 1 input, N outputs | Commit TX, Distribution | High (overhead amortized) |
| **N→N** | N inputs, N outputs | Send TX | Medium (linear scaling) |
| **N→1** | N inputs, 1 output | Consolidation | High (overhead amortized) |
| **1→1** | 1 input, 1 output | Reveal TX | ❌ Cannot batch for BRC-20 |

---

### 1→N Transactions (Commit)

**Structure:** 1 funding input → N commit outputs + 1 change

```
Formula:
  vBytes = 10.5 (overhead) + 57.5 (input) + 43 × (N + 1) (outputs)
  vBytes = 68 + 43N + 43
  vBytes = 111 + 43N

Cost per inscription = (111 + 43N) / N = 111/N + 43
```

| Batch Size (N) | Total vBytes | Cost @ 1 sat/vB | **Per Inscription** | Marginal Savings |
|----------------|--------------|-----------------|---------------------|------------------|
| 1 | 154 | 154 sats | **154.0 sats** | - |
| 2 | 197 | 197 sats | **98.5 sats** | 55.5 sats (36%) |
| 5 | 326 | 326 sats | **65.2 sats** | 33.3 sats (34%) |
| 10 | 541 | 541 sats | **54.1 sats** | 11.1 sats (17%) |
| 20 | 971 | 971 sats | **48.6 sats** | 5.5 sats (10%) |
| 50 | 2261 | 2261 sats | **45.2 sats** | 3.4 sats (7%) |
| 100 | 4411 | 4411 sats | **44.1 sats** | 1.1 sats (2%) |
| ∞ | - | - | **43.0 sats** | Theoretical floor |

**Diminishing Returns:** After batch size ~25, savings drop below 5 sats per additional inscription.

```
Optimization Curve (1→N):

Cost/inscription
     │
 154 ┤ ●
     │   ╲
 100 ┤     ╲
     │       ╲
  65 ┤         ●────
     │              ╲────●────●────●───→ 43 (floor)
  43 ┤─────────────────────────────────
     └──┬──┬──┬───┬───┬───┬───┬───┬───→ Batch size
        1  2  5  10  20  50 100 200
```

---

### N→N Transactions (Batch Send)

**Structure:** N inscription inputs + 1 fee input → N receiver outputs + 1 change

```
Formula:
  Input vBytes  = 57.5 × (N + 1)  // N inscriptions + 1 fee
  Output vBytes = 43 × (N + 1)    // N receivers + 1 change
  Total vBytes  = 10.5 + 57.5(N+1) + 43(N+1)
                = 10.5 + 100.5(N+1)
                = 10.5 + 100.5N + 100.5
                = 111 + 100.5N

Cost per inscription = (111 + 100.5N) / N = 111/N + 100.5
```

| Batch Size (N) | Total vBytes | Cost @ 1 sat/vB | **Per Inscription** | Marginal Savings |
|----------------|--------------|-----------------|---------------------|------------------|
| 1 | 212 | 212 sats | **212.0 sats** | - |
| 2 | 312 | 312 sats | **156.0 sats** | 56.0 sats (26%) |
| 5 | 614 | 614 sats | **122.8 sats** | 33.2 sats (21%) |
| 10 | 1116 | 1116 sats | **111.6 sats** | 11.2 sats (9%) |
| 20 | 2121 | 2121 sats | **106.1 sats** | 5.5 sats (5%) |
| 50 | 5136 | 5136 sats | **102.7 sats** | 3.4 sats (3%) |
| 100 | 10161 | 10161 sats | **101.6 sats** | 1.1 sats (1%) |
| ∞ | - | - | **100.5 sats** | Theoretical floor |

**Diminishing Returns:** After batch size ~20, savings drop below 5 sats per additional inscription.

```
Optimization Curve (N→N):

Cost/inscription
     │
 212 ┤ ●
     │   ╲
 156 ┤     ●
     │       ╲
 123 ┤         ●────
     │              ╲────●────●───→ 100.5 (floor)
 100 ┤─────────────────────────────────
     └──┬──┬──┬───┬───┬───┬───┬───┬───→ Batch size
        1  2  5  10  20  50 100 200
```

---

### 1→1 Transactions (Reveal) - Cannot Batch

**Structure:** 1 commit input → 1 inscription output

```
Formula:
  vBytes = 10.5 (overhead) + 97 (reveal input) + 43 (output)
  vBytes = 150.5 ≈ 150 vBytes

Cost per inscription = 150 sats (FIXED, cannot batch)
```

| Batch Size (N) | Total TXs | Total vBytes | **Per Inscription** |
|----------------|-----------|--------------|---------------------|
| 1 | 1 | 150 | **150 sats** |
| 5 | 5 | 750 | **150 sats** |
| 10 | 10 | 1500 | **150 sats** |
| 100 | 100 | 15000 | **150 sats** |

**No diminishing returns - each reveal is independent due to BRC-20 protocol limitation.**

---

### Combined Flow: Commit + Reveal + Send

```
Total Cost = Commit(1→N) + Reveal(1→1 × N) + Send(N→N) + Dust(N)

           = (111 + 43N)/N + 150 + (111 + 100.5N)/N + 330
           
           = 111/N + 43 + 150 + 111/N + 100.5 + 330
           
           = 222/N + 623.5
```

| Batch Size (N) | Commit | Reveal | Send | Dust | **Total/each** | vs Single |
|----------------|--------|--------|------|------|----------------|-----------|
| 1 | 154 | 150 | 212 | 330 | **846 sats** | - |
| 2 | 98 | 150 | 156 | 330 | **734 sats** | -13% |
| 5 | 65 | 150 | 123 | 330 | **668 sats** | -21% |
| 10 | 54 | 150 | 112 | 330 | **646 sats** | -24% |
| 20 | 49 | 150 | 106 | 330 | **635 sats** | -25% |
| 50 | 45 | 150 | 103 | 330 | **628 sats** | -26% |
| 100 | 44 | 150 | 102 | 330 | **626 sats** | -26% |
| ∞ | 43 | 150 | 100.5 | 330 | **623.5 sats** | -26% |

---

### Optimal Batch Size Analysis

```
Marginal Savings Chart:

Savings/inscription when increasing batch size
     │
  56 ┤ ●  (1→2: biggest jump)
     │
  33 ┤    ●  (2→5)
     │
  22 ┤        ●  (5→10)
     │
  11 ┤            ●  (10→20)
     │
   5 ┤                ●  (20→50)
     │    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ "Worth it" threshold
   2 ┤                    ●  (50→100)
     │
   0 └──┬───┬───┬───┬───┬───┬───→ Batch transition
        1→2 2→5 5→10 10→20 20→50 50→100
```

### Recommendations by Volume

| Daily Volume | Recommended Batch | Cost/inscription | Efficiency |
|--------------|-------------------|------------------|------------|
| 1-5 | 5 | 668 sats | 79% |
| 5-20 | 10-20 | 635-646 sats | 76-77% |
| 20-100 | 25-50 | 628-632 sats | 74-75% |
| 100+ | 50-100 | 626-628 sats | 74% |

**Sweet spot:** Batch size 20-25 captures 95% of possible savings with manageable complexity.

---

### Calculator: Estimate Your Costs

```javascript
// JavaScript cost calculator
function calculateBRC20TransferCost(batchSize, feeRate = 1) {
  const DUST = 330;
  
  // Commit (1→N): 111 + 43N vBytes
  const commitVB = 111 + 43 * batchSize;
  const commitPerInscription = (commitVB * feeRate) / batchSize;
  
  // Reveal (1→1 × N): 150 vBytes each, cannot batch
  const revealPerInscription = 150 * feeRate;
  
  // Send (N→N): 111 + 100.5N vBytes  
  const sendVB = 111 + 100.5 * batchSize;
  const sendPerInscription = (sendVB * feeRate) / batchSize;
  
  // Total
  const totalPerInscription = commitPerInscription + revealPerInscription + sendPerInscription + DUST;
  
  return {
    batchSize,
    feeRate,
    commit: Math.round(commitPerInscription),
    reveal: revealPerInscription,
    send: Math.round(sendPerInscription),
    dust: DUST,
    total: Math.round(totalPerInscription),
    efficiency: Math.round((1 - totalPerInscription / 846) * 100) + '%'
  };
}

// Example usage:
console.log(calculateBRC20TransferCost(20, 1));
// { batchSize: 20, feeRate: 1, commit: 49, reveal: 150, send: 106, dust: 330, total: 635, efficiency: '25%' }
```

### Fee Rate Impact

| Batch Size | 1 sat/vB | 2 sat/vB | 5 sat/vB | 10 sat/vB |
|------------|----------|----------|----------|-----------|
| 1 | 846 sats | 1362 sats | 2910 sats | 5490 sats |
| 10 | 646 sats | 962 sats | 1910 sats | 3490 sats |
| 50 | 628 sats | 926 sats | 1820 sats | 3310 sats |

**Key insight:** At higher fee rates, the fixed 330 sat dust becomes a smaller percentage of total cost, making batching even more valuable.

---

## Summary

### Production Recommendation

**Use Internal Inscription + Auto-Fee during low congestion: ~465 sats per user settlement**

| Component | Provider | Cost @ 0.4 sat/vB |
|-----------|----------|-------------------|
| Batched commit | Internal | ~26 sats |
| Individual reveal | Internal | ~60 sats |
| Batched send | Internal | ~49 sats |
| Inscription dust | - | 330 sats |
| **Total** | | **~465 sats** |

**Fallback: UniSat API + Batched Sends: ~671 sats (fixed rate, reliable)**

| Component | Provider | Cost @ 1 sat/vB |
|-----------|----------|-----------------|
| Inscription creation | UniSat API | 341 sats |
| Inscription dust | - | 330 sats |
| Batched send to user | Internal | ~106 sats (batch 20+) |
| **Total** | | **~671 sats** |

### Why Not Full Internal Inscription?

| Factor | UniSat + Batch | Full Internal |
|--------|----------------|---------------|
| Cost per settlement | ~671 sats | ~635 sats |
| **Savings** | - | **36 sats (5%)** |
| UTXO management | ❌ Not needed | ✅ Required |
| Inscription tracking | ❌ Not needed | ✅ Required |
| Error handling | Minimal | Complex |
| Infrastructure | Low | High |

**With sub-sat fee rates (0.4 sat/vB):**

| Factor | UniSat (1 sat/vB) | Internal (0.4 sat/vB) |
|--------|-------------------|----------------------|
| Cost per settlement | ~671 sats | **~465 sats** |
| **Savings** | - | **31%** |

**Conclusion:** With sub-sat fee rates during low congestion, internal inscription saves **~206 sats (31%)** per inscription - this **does** justify the operational complexity for high-volume operations.

### The Hard Limits

| Limitation | Reason |
|------------|--------|
| 150 sats reveal fee | Fixed inscription envelope size |
| 330 sats dust | Taproot dust limit (required for inscriptions) |
| 1 BRC-20 per TX | Protocol specification |
| 2-step transfers | Indexer validation requirement |

### Cost Comparison

| Method | Fee Rate | Cost | Savings | Notes |
|--------|----------|------|---------|-------|
| **Internal + Auto-Fee** | 0.1 sat/vB | **~364 sats** | 46% | Theoretical minimum |
| **Internal + Auto-Fee** | 0.2 sat/vB | **~398 sats** | 41% | Very low congestion |
| **Internal + Auto-Fee** | 0.4 sat/vB | **~465 sats** | 31% | ✅ **Tested & working** |
| UniSat + Batched Send | 1 sat/vB | ~671 sats | - | Fixed rate, reliable |
| Full Internal | 1 sat/vB | ~635 sats | 5% | Requires UTXO management |
| Single inscription | 1 sat/vB | ~846 sats | -26% | Not recommended |

**Key insight:** Sub-sat fee rates (0.4 sat/vB) during low congestion periods provide the best cost savings while maintaining reliable confirmation times.

---

*Document generated from optimization research and testing. All transaction URLs are from Bitcoin mainnet.*
