# BRC-20 Batching Cost Ground Test Report

**Date:** January 26, 2026  
**Test Environment:** Bitcoin Mainnet  
**Fee Rate:** 1 sat/vB  
**Token:** SIGAI (BRC-20)  
**BTC Price:** $88,483 (for USD conversions)

---

## Executive Summary

This report documents the ground truth costs of BRC-20 token transfers comparing **singular transactions** versus **batched transactions**. All data is sourced from confirmed mainnet transactions.

| Metric | Singular | Batched (Current) | Savings |
|--------|----------|-------------------|---------|
| **Total Cost per User** | 876 sats ($0.78) | ~672 sats ($0.59) | **~204 sats / $0.18 (23.3%)** |
| **Step 1 (Payment)** | 153 sats ($0.14) | ~55 sats ($0.05) | ~98 sats / $0.09 (64%) |
| **Step 2 (Worker Fee)** | 182 sats ($0.16) | 182 sats ($0.16) | 0 sats (0%) |
| **Step 3 (Transfer)** | 211 sats ($0.19) | ~105 sats ($0.09) | ~106 sats / $0.09 (50%) |
| **Dust Limit** | 330 sats ($0.29) | 330 sats ($0.29) | 0 sats (0%) |

---

## Test Methodology

### BRC-20 Transfer Process (3 Steps)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BRC-20 TRANSFER LIFECYCLE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STEP 1: Payment to UniSat                                                  │
│  ├── Pay inscription fee to UniSat API                                      │
│  └── Includes: network fee + service fee + dust limit                       │
│                                                                             │
│  STEP 2: UniSat Internal Operation                                          │
│  ├── UniSat creates transfer inscription (commit + reveal)                  │
│  └── Sends inscription back to sender address                               │
│                                                                             │
│  STEP 3: Send Transfer Inscription                                          │
│  ├── Sender sends inscription UTXO to recipient                             │
│  └── BRC-20 indexer credits recipient's balance                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Singular Transaction Costs

### Overview
Individual transactions processed one user at a time.

| Step | Description | Cost per User | Transaction |
|------|-------------|---------------|-------------|
| **Step 1** | Payment to UniSat | **153 sats** | [mempool.space](https://mempool.space/tx/b8b7f3e0645506f40c469f6043d112414ea09400d893af25c3533e7d7807e85f) |
| **Step 2** | Worker Fee (inscription creation network fee) | **182 sats** | [mempool.space](https://mempool.space/tx/854f03dbd0e96b80bed6fb2827668589f007001670981a5bf11c0b89167e636f) |
| **Step 3** | Send inscription to recipient | **211 sats** | [mempool.space](https://mempool.space/tx/8a8146f3906eb5d4397e9ff03510ff565cc762e3c113078dff39ec0c8186e2f1) |
| **Dust** | Inscription UTXO minimum value | **330 sats** | (Protocol requirement) |
| | **TOTAL** | **876 sats** | |

### Cost Breakdown Visualization

```
SINGULAR TRANSACTION COST: 876 sats per user
├── Step 1: Payment to UniSat ██████████████████ 153 sats (17.5%)
├── Step 2: Worker Fee        █████████████████████ 182 sats (20.8%)
├── Step 3: Send Inscription  █████████████████████████ 211 sats (24.1%)
└── Dust Limit (fixed)        ██████████████████████████████████████ 330 sats (37.7%)
```

---

## Batched Transaction Costs

### Overview
Multiple users processed in single transactions where possible.

| Step | Description | Total Cost | Users | Cost per User | Transaction |
|------|-------------|------------|-------|---------------|-------------|
| **Step 1** | Batch Payment to UniSat | **1,105 sats*** | 20 | **55.25 sats** | [mempool.space](https://mempool.space/tx/1a9fb70f0867e9e0fc4d2f15d829b9fbbe011634e8406a903dfb1e28d7123d5f) |
| **Step 2** | Worker Fee (network fee per inscription) | **182 sats** | 1 | **182 sats** | [mempool.space](https://mempool.space/tx/854f03dbd0e96b80bed6fb2827668589f007001670981a5bf11c0b89167e636f) |
| **Step 3** | Batch Send inscriptions | **2,511 sats** | 24 | **104.63 sats** | [mempool.space](https://mempool.space/tx/02114be4443f13558f6067c519fb7dd05ea84db5e767f0b3c2e3430b30d147b3) |
| **Dust** | Inscription UTXO minimum value | **330 sats** | 1 | **330 sats** | (Protocol requirement) |
| | **TOTAL per user** | | | **~671.88 sats** | |

> *\*Step 1 Total Cost has been adjusted to 1 sat/vB for consistent comparison. The linked mempool transaction may show a different fee rate.*

### Cost Breakdown Visualization

```
BATCHED TRANSACTION COST: ~672 sats per user
├── Step 1: Batch Payment     ████████ 55.25 sats (8.2%)
├── Step 2: Worker Fee        ███████████████████████████ 182 sats (27.1%)
├── Step 3: Batch Send        ████████████████ 104.63 sats (15.6%)
└── Dust Limit (fixed)        █████████████████████████████████████████████████ 330 sats (49.1%)
```

---

## Savings Analysis

### Per-Step Savings

| Step | Singular | Batched (Current) | Savings (sats) | Savings (%) | Batchable? |
|------|----------|-------------------|----------------|-------------|------------|
| **Step 1** | 153 sats | 55.25 sats | **97.75 sats** | **63.9%** | ✅ Yes |
| **Step 2** | 182 sats | 182 sats | **0 sats** | **0%** | ❌ No (Worker fee) |
| **Step 3** | 211 sats | 104.63 sats | **106.37 sats** | **50.4%** | ✅ Yes |
| **Dust Limit** | 330 sats | 330 sats | **0 sats** | **0%** | ❌ No (Protocol fixed) |
| **TOTAL** | **876 sats** | **671.88 sats** | **204.12 sats** | **23.3%** | |

### Savings Visualization

```
COST COMPARISON (per user)

Singular:         ████████████████████████████████████████████████████████████████████████████████████████ 876 sats
Batched (Current): ████████████████████████████████████████████████████████████████████ 672 sats
                                                                              ├────────────────────┤
                                                                              Savings: 204 sats (23.3%)
```

### Cost Breakdown: Fixed vs Variable

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FIXED vs VARIABLE COSTS (per user)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FIXED COSTS (cannot be reduced by batching):                               │
│    • Dust Limit:        330 sats  (Bitcoin protocol)                        │
│    • Worker Fee:        182 sats  (network fee per inscription)             │
│    ─────────────────────────────                                            │
│    Subtotal:            512 sats  (58.4% of singular cost)                  │
│                                                                             │
│  VARIABLE COSTS (reduced by batching):                                      │
│    • Step 1 (Payment):  153 → 55 sats  (64% savings)                        │
│    • Step 3 (Transfer): 211 → 105 sats (50% savings)                        │
│    ─────────────────────────────────────                                    │
│    Subtotal:            364 → 160 sats (56% savings on variable)            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Step 2 Cannot Be Batched

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STEP 2: WORKER FEE (NETWORK FEE)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  This is the network fee UniSat pays to create each inscription:            │
│                                                                             │
│    Payment 1 ──→ [Commit TX] ──→ [Reveal TX] ──→ Inscription 1              │
│    Payment 2 ──→ [Commit TX] ──→ [Reveal TX] ──→ Inscription 2              │
│    Payment 3 ──→ [Commit TX] ──→ [Reveal TX] ──→ Inscription 3              │
│         ...                                                                 │
│                                                                             │
│  ⚠️  This is a per-inscription network fee, NOT a service fee              │
│  ⚠️  We have NO control over this step - it's UniSat's internal process    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Optimal Batch Size Analysis

Using the [Bitcoin Fee Calculator](https://bitbo.io/tools/fee-calculator/) methodology and actual transaction data, we can model the diminishing returns of batching.

---

### 1→N Transactions (Batch Payment)

**Use Case:** Paying for multiple UniSat inscription orders from a single wallet

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    1→N TRANSACTION STRUCTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐     ┌─────────────┐                                       │
│   │   INPUT     │     │  OUTPUT 1   │ ──→ UniSat Payment Address 1          │
│   │  (Funding   │     ├─────────────┤                                       │
│   │   Wallet)   │ ──→ │  OUTPUT 2   │ ──→ UniSat Payment Address 2          │
│   │             │     ├─────────────┤                                       │
│   │  ~68 vB     │     │  OUTPUT 3   │ ──→ UniSat Payment Address 3          │
│   └─────────────┘     ├─────────────┤                                       │
│                       │     ...     │                                       │
│                       ├─────────────┤                                       │
│                       │  OUTPUT N   │ ──→ UniSat Payment Address N          │
│                       ├─────────────┤                                       │
│                       │   CHANGE    │ ──→ Back to sender                    │
│                       └─────────────┘                                       │
│                                                                             │
│   Transaction Size = Overhead + Input + (N+1) × Output                      │
│                    = 10.5 + 68 + (N+1) × 31 vbytes                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Cost Model (Empirical from Mainnet Data)

| Component | Size | Notes |
|-----------|------|-------|
| Fixed overhead | ~10.5 vbytes | Version, locktime, etc. |
| Input (P2WPKH) | ~68 vbytes | Single funding input |
| Each output (P2WPKH) | ~31 vbytes | To UniSat addresses |
| Change output | ~31 vbytes | Back to sender |

**Formula:** `Cost(N) = 103/N + 50` sats/user (at 1 sat/vB)

#### 1→N Batch Size Analysis

| Batch Size | TX Size (vB) | Per User | vs Singular (153) | % of Max Savings |
|------------|--------------|----------|-------------------|------------------|
| **1** | 153 | 153 sats | 0% | 0% |
| **5** | 353 | 70.6 sats | 53.9% | 80% |
| **10** | 603 | 60.3 sats | 60.6% | 90% |
| **15** | 853 | 56.9 sats | 62.8% | 93% |
| **20** | 1,103 | 55.2 sats | 63.9% | 95% |
| **25** | 1,353 | 54.1 sats | 64.6% | 96% |
| **30** | 1,603 | 53.4 sats | 65.1% | 97% |
| **50** | 2,603 | 52.1 sats | 66.0% | 98% |
| **100** | 5,103 | 51.0 sats | 66.7% | 99% |
| **∞** | ∞ | 50 sats | 67.3% | 100% |

#### 1→N Diminishing Returns Chart

```
SAVINGS FOR 1→N TRANSACTIONS (Payment Batching)

 70% ─────────────────────────────────────────────────────────── MAX (67.3%)
     │                                 ○───○───○───○───○───○
 65% ─│                    ○───○───○
     │              ○───○
 60% ─│         ○
     │
 55% ─│    ○
     │
 50% ─│
     │
  0% ─│○
     └────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────
          5   10   15   20   25   30   40   50   75  100  ∞
                           Batch Size (N)

     ✅ Sweet Spot: 15-25 users (93-96% of max savings)
```

#### 1→N Recommendation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              1→N OPTIMAL BATCH SIZE: 20 USERS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  At N=20:                                                                   │
│    • Cost per user: 55.2 sats (vs 153 sats singular)                        │
│    • Savings: 63.9% per user                                                │
│    • Achieves 95% of maximum possible savings                               │
│    • Total TX size: ~1,103 vbytes (well under block limits)                 │
│                                                                             │
│  Beyond N=20:                                                               │
│    • N=30: only +1.2% more savings                                          │
│    • N=50: only +2.1% more savings                                          │
│    • Diminishing returns kick in hard                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### N→N+1 Transactions (Batch Transfer)

**Use Case:** Sending N inscriptions from sender wallet to N different recipients

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    N→N+1 TRANSACTION STRUCTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────┐     ┌─────────────┐                                       │
│   │  INPUT 1    │     │  OUTPUT 1   │ ──→ Recipient 1 (P2TR)                │
│   │ Inscription │     ├─────────────┤                                       │
│   │   ~58 vB    │     │  OUTPUT 2   │ ──→ Recipient 2 (P2TR)                │
│   ├─────────────┤     ├─────────────┤                                       │
│   │  INPUT 2    │ ──→ │  OUTPUT 3   │ ──→ Recipient 3 (P2TR)                │
│   │ Inscription │     ├─────────────┤                                       │
│   ├─────────────┤     │     ...     │                                       │
│   │     ...     │     ├─────────────┤                                       │
│   ├─────────────┤     │  OUTPUT N   │ ──→ Recipient N (P2TR)                │
│   │  INPUT N    │     ├─────────────┤                                       │
│   │ Inscription │     │   CHANGE    │ ──→ Back to sender (excess sats)      │
│   └─────────────┘     └─────────────┘                                       │
│                                                                             │
│   Transaction Size = Overhead + N × Input + (N+1) × Output                  │
│                    = 10.5 + N × 58 + (N+1) × 43 vbytes                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Cost Model (Empirical from Mainnet Data)

| Component | Size | Notes |
|-----------|------|-------|
| Fixed overhead | ~10.5 vbytes | Version, locktime, etc. |
| Each input (P2TR) | ~57.5 vbytes | Inscription UTXO |
| Each output (P2TR) | ~43 vbytes | To recipient Taproot address |
| Change output | ~43 vbytes | Excess sats back to sender |

**Formula:** `Cost(N) = 111/N + 100` sats/user (at 1 sat/vB)

#### N→N+1 Batch Size Analysis

| Batch Size | TX Size (vB) | Per User | vs Singular (211) | % of Max Savings |
|------------|--------------|----------|-------------------|------------------|
| **1** | 211 | 211 sats | 0% | 0% |
| **5** | 611 | 122.2 sats | 42.1% | 80% |
| **10** | 1,111 | 111.1 sats | 47.3% | 90% |
| **15** | 1,611 | 107.4 sats | 49.1% | 93% |
| **20** | 2,111 | 105.6 sats | 50.0% | 95% |
| **24** | 2,511 | 104.6 sats | 50.4% | 96% |
| **30** | 3,111 | 103.7 sats | 50.9% | 97% |
| **50** | 5,111 | 102.2 sats | 51.6% | 98% |
| **100** | 10,111 | 101.1 sats | 52.1% | 99% |
| **∞** | ∞ | 100 sats | 52.6% | 100% |

#### N→N+1 Diminishing Returns Chart

```
SAVINGS FOR N→N+1 TRANSACTIONS (Transfer Batching)

 55% ─────────────────────────────────────────────────────────── MAX (52.6%)
     │                                 ○───○───○───○───○───○
 50% ─│                    ○───○───○
     │              ○───○
 45% ─│         ○
     │    ○
 40% ─│
     │
 35% ─│
     │
  0% ─│○
     └────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────
          5   10   15   20   25   30   40   50   75  100  ∞
                           Batch Size (N)

     ✅ Sweet Spot: 20-25 users (95-96% of max savings)
```

#### N→N+1 Recommendation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              N→N+1 OPTIMAL BATCH SIZE: 20-24 USERS                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  At N=20:                                                                   │
│    • Cost per user: 105.6 sats (vs 211 sats singular)                       │
│    • Savings: 50.0% per user                                                │
│    • Achieves 95% of maximum possible savings                               │
│    • Total TX size: ~2,111 vbytes                                           │
│                                                                             │
│  At N=24 (tested):                                                          │
│    • Cost per user: 104.6 sats                                              │
│    • Total TX: 2,511 vbytes (confirmed on mainnet)                          │
│                                                                             │
│  ⚠️  Mempool Consideration:                                                 │
│    • Bitcoin Core limits: 25 unconfirmed ancestors/descendants              │
│    • Staying at N≤24 keeps you within safe limits                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Combined Analysis (1→N + N→N+1)

#### Transaction Size Model Summary

| Transaction Type | Fixed Overhead | Marginal per User | Formula |
|------------------|---------------|-------------------|---------|
| **1→N** (Payment) | ~103 vbytes | ~50 vbytes | `103/N + 50` sats/user |
| **N→N+1** (Transfer) | ~111 vbytes | ~100 vbytes | `111/N + 100` sats/user |

#### Combined Cost per User at Different Batch Sizes

| Batch Size | Step 1 (1→N) | Step 3 (N→N+1) | Combined | USD (@$88,483) | vs Singular |
|------------|--------------|----------------|----------|----------------|-------------|
| **1** | 153 sats | 211 sats | 364 sats | $0.32 | 0% |
| **5** | 70.6 sats | 122.2 sats | 192.8 sats | $0.17 | 47.0% |
| **10** | 60.3 sats | 111.1 sats | 171.4 sats | $0.15 | 52.9% |
| **15** | 56.9 sats | 107.4 sats | 164.3 sats | $0.15 | 54.9% |
| **20** | 55.2 sats | 105.6 sats | 160.8 sats | $0.14 | 55.8% |
| **25** | 54.1 sats | 104.4 sats | 158.5 sats | $0.14 | 56.5% |
| **30** | 53.4 sats | 103.7 sats | 157.1 sats | $0.14 | 56.8% |
| **50** | 52.1 sats | 102.2 sats | 154.3 sats | $0.14 | 57.6% |
| **100** | 51.0 sats | 101.1 sats | 152.1 sats | $0.13 | 58.2% |
| **∞** | 50 sats | 100 sats | 150 sats | $0.13 | 58.8% |

#### Combined Diminishing Returns Visualization

```
SAVINGS vs SINGULAR (Steps 1+3 Combined)

 60% ─────────────────────────────────────────────────────────────────── MAX (58.8%)
     │                                    ○───○───○───○───○───○───○
 55% ─│                           ○───○
     │                    ○───○
 50% ─│              ○
     │         ○
 45% ─│    ○
     │
 40% ─│
     │
  0% ─│○
     └─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────
           5    10    15    20    25    30    40    50    75   100   ∞
                              Batch Size (N users)

     ◀── Steep gains ──▶◀──── Diminishing returns ────▶
```

---

### Overall Optimal Batch Size Recommendation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTIMAL BATCH SIZE: 20 USERS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1→N (Payment):                                                             │
│    • 55.2 sats/user (63.9% savings vs 153 singular)                         │
│    • TX size: ~1,103 vbytes                                                 │
│                                                                             │
│  N→N+1 (Transfer):                                                          │
│    • 105.6 sats/user (50.0% savings vs 211 singular)                        │
│    • TX size: ~2,111 vbytes                                                 │
│                                                                             │
│  Combined:                                                                  │
│    • 160.8 sats/user variable costs (55.8% savings vs 364 singular)         │
│    • Achieves 95% of maximum possible savings                               │
│                                                                             │
│  Tradeoffs:                                                                 │
│    • Larger batches = more waiting time to accumulate users                 │
│    • Larger batches = higher single-transaction failure risk                │
│    • Mempool chain limits: 25 unconfirmed transactions max                  │
│                                                                             │
│  ✅ RECOMMENDATION: Batch size of 20 users                                  │
│     • Achieves 95% of maximum possible savings                              │
│     • Stays within mempool limits                                           │
│     • Reasonable accumulation time                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Total Cost with Fixed Costs Included

| Batch Size | Variable (1+3) | Fixed (2+Dust) | Total | USD (@$88,483) | vs Singular |
|------------|----------------|----------------|-------|----------------|-------------|
| **1** | 364 sats | 512 sats | 876 sats | **$0.78** | 0% |
| **10** | 171 sats | 512 sats | 683 sats | **$0.60** | 22.0% |
| **20** | 161 sats | 512 sats | 673 sats | **$0.60** | 23.2% |
| **30** | 157 sats | 512 sats | 669 sats | **$0.59** | 23.6% |
| **∞** | 150 sats | 512 sats | 662 sats | **$0.59** | 24.4% |

**Key Insight:** Fixed costs (dust + worker fee) represent 58% of total costs, limiting maximum achievable savings to ~24% regardless of batch size.

---

## Total Cost Projections

### At Scale (1 sat/vB fee rate)

| Users | Singular Total | Batched (Current) Total | Total Savings |
|-------|----------------|-------------------------|---------------|
| 10 | 8,760 sats | 6,719 sats | 2,041 sats |
| 20 | 17,520 sats | 13,438 sats | 4,082 sats |
| 30 | 26,280 sats | 20,156 sats | 6,124 sats |
| 50 | 43,800 sats | 33,594 sats | 10,206 sats |
| 100 | 87,600 sats | 67,188 sats | 20,412 sats |
| 1,000 | 876,000 sats | 671,880 sats | 204,120 sats |

### USD Equivalent (at $100,000/BTC)

| Users | Singular | Batched (Current) | Savings |
|-------|----------|-------------------|---------|
| 10 | $8.76 | $6.72 | $2.04 |
| 100 | $87.60 | $67.19 | $20.41 |
| 1,000 | $876.00 | $671.88 | $204.12 |
| 10,000 | $8,760.00 | $6,718.80 | $2,041.20 |

---

## Fixed Costs (Unavoidable)

### Dust Limit: 330 sats per inscription

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DUST LIMIT REQUIREMENT                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Every inscription requires a minimum UTXO value:                           │
│                                                                             │
│    Address Type     │  Dust Limit                                           │
│    ─────────────────┼──────────────                                         │
│    Taproot (P2TR)   │  330 sats                                             │
│    SegWit (P2WPKH)  │  294 sats                                             │
│    Legacy (P2PKH)   │  546 sats                                             │
│                                                                             │
│  ⚠️  This is a Bitcoin protocol requirement, NOT a fee                     │
│  ⚠️  These sats are NOT "spent" - they remain with the inscription         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5-Byte BRC-20 Mint Optimization

> ⚠️ **NOT IMPLEMENTED**: There is currently no implementation for BRC-20 5-byte mint in the service. The savings below are **theorized** based on UniSat API documentation. Real ground test data would require implementation and testing.

### What is 5-byte BRC-20?

BRC-20 originally supported only **4-character tickers** (like "ORDI", "SATS"). The newer **5-byte BRC-20** standard allows **5-character tickers** with a different minting mechanism that requires deployer authorization.

### UniSat API Endpoints

| Endpoint | Use Case | Special Requirements |
|----------|----------|----------------------|
| `/v2/inscribe/order/create/brc20-transfer` | Move existing tokens | None |
| `/v2/inscribe/order/create/brc20-mint` | 4-char mint | None |
| `/v2/inscribe/order/create/brc20-5byte-mint` | 5-char mint | Commit + Reveal (PSBT signing) |

---

### 5-Byte Mint Process (Commit + Reveal)

For 5-byte BRC-20 tokens, UniSat requires a **two-step process** where you sign PSBTs:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    5-BYTE BRC-20 MINT FLOW (COMMIT + REVEAL)                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STEP 1: CREATE ORDER + GET COMMIT PSBT                                     │
│    POST /v2/inscribe/order/create/brc20-5byte-mint                          │
│    Payload:                                                                 │
│      {                                                                      │
│        "receiveAddress": USER_ADDRESS,      // Mint directly to user        │
│        "deployAddress": YOUR_DEPLOY_ADDR,   // Token deployer address       │
│        "publicKey": YOUR_PUBLIC_KEY,        // Deployer's public key        │
│        "brc20Ticker": "SIGAI",                                              │
│        "brc20Amount": "10",                                                 │
│        "feeRate": 1,                                                        │
│        "outputValue": 330                                                   │
│      }                                                                      │
│    Response: Unsigned PSBT for commit transaction                           │
│                                                                             │
│  STEP 2: SIGN COMMIT PSBT                                                   │
│    • Sign the PSBT with deployer's private key                              │
│    • This authorizes you as the token deployer                              │
│                                                                             │
│  STEP 3: SUBMIT SIGNED COMMIT + GET REVEAL                                  │
│    • Submit signed commit PSBT                                              │
│    • UniSat broadcasts commit and creates reveal                            │
│    • Inscription minted directly to user's address                          │
│                                                                             │
│  NO STEP 4 NEEDED: Tokens go directly to user! ✅                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Cost Structure: 5-Byte Mint (Theoretical)

| Component | 4-Char Transfer | 5-Byte Mint | Notes |
|-----------|-----------------|-------------|-------|
| **Step 1** (Payment to UniSat) | 55 sats ($0.05) | 55 sats ($0.05) | Same |
| **Step 2** (Worker/Network Fee) | 182 sats ($0.16) | 182 sats ($0.16) | Same |
| **Step 3** (Send to User) | 105 sats ($0.09) | **0 sats ($0.00)** | **ELIMINATED!** |
| **Dust Limit** | 330 sats ($0.29) | 330 sats ($0.29) | Same |
| **TOTAL** | **672 sats ($0.59)** | **567 sats ($0.50)** | **Save $0.09 (15.6%)** |

---

### 5-Byte Mint API Payload

```python
# UniSat 5-Byte Mint Endpoint
UNISAT_API_5BYTE_ENDPOINT = "https://open-api.unisat.io/v2/inscribe/order/create/brc20-5byte-mint"

payload = {
    "receiveAddress": USER_ADDRESS,           # Mint directly to user!
    "feeRate": 1,
    "outputValue": 330,
    "devAddress": "",
    "devFee": 0,
    "brc20Ticker": "SIGAI",
    "brc20Amount": "10",
    "count": 1,
    "deployAddress": DEPLOY_ADDRESS,          # Required: Your deploy address
    "publicKey": DEPLOY_PUBLIC_KEY            # Required: Your public key
}
```

---

### At Scale Savings (5-Byte Mint vs Transfer) - Theoretical

| Users | Transfer Cost | 5-Byte Mint Cost | Total Savings |
|-------|---------------|------------------|---------------|
| 20 | 13,438 sats ($11.89) | 11,345 sats ($10.04) | **2,093 sats ($1.85)** |
| 50 | 33,594 sats ($29.73) | 28,363 sats ($25.10) | **5,231 sats ($4.63)** |
| 100 | 67,188 sats ($59.45) | 56,725 sats ($50.19) | **10,463 sats ($9.26)** |
| 1,000 | 671,880 sats ($594.51) | 567,250 sats ($501.91) | **104,630 sats ($92.59)** |

At $88,483/BTC:
- **1,000 users**: Save **$92.59** by using 5-byte mint instead of transfer

---

### Requirements for 5-Byte Mint

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PREREQUISITES FOR 5-BYTE MINT                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ✅ Your token must be a 5-character ticker                                 │
│     • Example: "SIGAI" (5 chars)                                            │
│                                                                             │
│  ✅ Your token must have REMAINING MINTABLE SUPPLY                          │
│     • Check: total minted < max supply                                      │
│                                                                             │
│  ✅ You must be the TOKEN DEPLOYER                                          │
│     • Have access to deploy address private key                             │
│     • Can sign PSBTs with deployer key                                      │
│                                                                             │
│  ✅ You need the deployer's PUBLIC KEY                                      │
│     • Compressed (33 bytes): 02/03 + 32 bytes                               │
│     • X-only for Taproot (32 bytes)                                         │
│                                                                             │
│  ⚠️  PSBT SIGNING REQUIRED                                                  │
│     • You must sign the commit PSBT with deployer's key                     │
│     • This proves you're authorized to mint                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Implementation Complexity

| Aspect | Transfer Flow | 5-Byte Mint Flow |
|--------|---------------|------------------|
| **API Calls** | 1 | 2-3 (commit + reveal) |
| **Signing Required** | No | Yes (PSBT) |
| **Key Management** | Simple | Need deployer key |
| **Steps** | 3 | 2 |
| **Cost per User** | 672 sats | 567 sats |
| **Savings** | Baseline | **15.6%** |

---

### Summary: 5-Byte Mint vs Transfer (Theoretical)

| Metric | Transfer | 5-Byte Mint | Winner |
|--------|----------|-------------|--------|
| **Cost per user** | 672 sats ($0.59) | 567 sats ($0.50) | ✅ Mint |
| **Steps required** | 3 | 2 | ✅ Mint |
| **API complexity** | Simple | PSBT signing | ⚠️ Transfer |
| **Requires deployer key** | No | Yes | ⚠️ Transfer |
| **Requires token supply** | No | Yes | ⚠️ Transfer |

---

### Recommendation (Theoretical - Not Yet Implemented)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RECOMMENDATION (THEORETICAL)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  IF you have a 5-byte token with remaining supply + deployer key access:    │
│    ✅ Use 5-BYTE MINT with receiveAddress = user's address                  │
│    ✅ Save 105 sats / $0.09 per user (15.6% reduction) *theorized*          │
│    ✅ 2 steps instead of 3 (eliminates Step 3)                              │
│    ⚠️  Requires PSBT signing with deployer key                              │
│    ⚠️  NOT YET IMPLEMENTED - requires ground testing                        │
│                                                                             │
│  IF you don't have deployer key access or token is fully minted:            │
│    ✅ Use TRANSFER endpoint (current flow - IMPLEMENTED)                    │
│    ✅ 672 sats / $0.59 per user (batched) - GROUND TESTED                   │
│                                                                             │
│  BEST CASE (5-Byte Mint + Batching) - THEORETICAL:                          │
│    • 567 sats / $0.50 per user                                              │
│    • 35.3% savings vs singular transfer (876 sats / $0.78)                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Transaction Evidence

### Step 1: Payment Transactions

| Type | Transaction ID | Mempool Link |
|------|----------------|--------------|
| Singular | `b8b7f3e0...7807e85f` | [View](https://mempool.space/tx/b8b7f3e0645506f40c469f6043d112414ea09400d893af25c3533e7d7807e85f) |
| Batched (Current, 20 users) | `1a9fb70f...7123d5f` | [View](https://mempool.space/tx/1a9fb70f0867e9e0fc4d2f15d829b9fbbe011634e8406a903dfb1e28d7123d5f) |

### Step 2: UniSat Internal Operations

| Type | Transaction ID | Mempool Link |
|------|----------------|--------------|
| Per inscription | `854f03db...167e636f` | [View](https://mempool.space/tx/854f03dbd0e96b80bed6fb2827668589f007001670981a5bf11c0b89167e636f) |

### Step 3: Transfer Transactions

| Type | Transaction ID | Mempool Link |
|------|----------------|--------------|
| Singular | `8a8146f3...8186e2f1` | [View](https://mempool.space/tx/8a8146f3906eb5d4397e9ff03510ff565cc762e3c113078dff39ec0c8186e2f1) |
| Batched (Current, 24 users) | `02114be4...d147b3` | [View](https://mempool.space/tx/02114be4443f13558f6067c519fb7dd05ea84db5e767f0b3c2e3430b30d147b3) |

---

## Conclusions

### Key Findings

1. **Batching reduces costs by 23.3%** at 1 sat/vB fee rate (including dust limit)
2. **Step 2 (Worker fee) is the bottleneck** - network fee per inscription, not batchable via UniSat
3. **Steps 1 and 3 batch effectively** - achieving 50-64% savings individually
4. **Dust limit (330 sats) is unavoidable** - Bitcoin protocol requirement, not a fee

### Recommendations

| Priority | Action | Expected Impact |
|----------|--------|-----------------|
| ✅ **Current** | Batch Steps 1 & 3 (UniSat + Batched Transfers) | 23.3% total cost reduction |
| 🔄 **Future (Not Implemented)** | 5-Byte Mint (skip Step 3) | Additional 15.6% reduction (theorized) |

### What Cannot Be Optimized

| Fixed Cost | Amount | Why |
|------------|--------|-----|
| **Dust Limit** | 330 sats ($0.29) | Bitcoin protocol minimum for Taproot UTXOs |
| **Worker Fee** | 182 sats ($0.16) | Network fee per inscription via UniSat |

---

## Appendix: Raw Calculations

```
SINGULAR COSTS:
  Step 1: 153 sats      (payment to UniSat)
  Step 2: 182 sats      (worker fee - network fee for inscription)
  Step 3: 211 sats      (send inscription to recipient)
  Dust:   330 sats      (inscription UTXO minimum)
  ─────────────────
  TOTAL:  876 sats per user

BATCHED COSTS:
  Step 1: 1105 / 20 = 55.25 sats per user   (batch payment)
  Step 2: 182 sats per user                  (worker fee - not batchable)
  Step 3: 2511 / 24 = 104.625 sats per user (batch send)
  Dust:   330 sats per user                  (not reducible)
  ───────────────────────────────────────────────────────────
  TOTAL:  671.875 sats per user

SAVINGS:
  Per user:   876 - 671.875 = 204.125 sats
  Percentage: 204.125 / 876 = 23.30%

COST BREAKDOWN BY TYPE:
  Fixed costs (per user, cannot batch):
    • Dust limit:       330 sats (37.7% of singular)
    • Worker fee:       182 sats (20.8% of singular)
    ────────────────────
    Fixed subtotal:     512 sats (58.4% of singular)

  Variable costs (can be batched):
    • Step 1 (payment):  153 → 55.25 sats  (63.9% savings)
    • Step 3 (transfer): 211 → 104.63 sats (50.4% savings)
    ────────────────────────────────────────
    Variable subtotal:   364 → 159.88 sats (56.1% savings on variable)
```

---

## UniSat API Dependence

### Current Usage

Currently we use the UniSat API for:
- **BRC-20 balance viewing** of wallets
- **Transfer inscription creation**

Transfer inscription creation requires **1 API call per user** (each settlement action requires at least 1 API call).

### Current Plan

On the **free plan**, we have **2,000 free API calls per day**.

### Scaling

If we scale this solution, we will need to purchase UniSat API calls:
- **Daily/monthly plans** for higher rate limits
- **Pay-as-you-go**: ~$80 per 200,000 calls (current sale: ~$40 per 200,000 calls)

### Why UniSat?

UniSat is the **cheapest inscription service with comprehensive API support** at the time of research.

### Removing UniSat Dependence

To not have a dependence on UniSat, we would need to implement our own inscriber/inscription service.

---

*Report generated from mainnet transaction data at 1 sat/vB fee rate.*
