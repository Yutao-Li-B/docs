# Beem Consumer Asset Tokenization

## What This Document Covers

This document explains how Beem turns user assets into spending power through **tokenization**. Users can pledge their assets (house, car, stocks, crypto, income, or cash) and receive instant spending power on a Beem card - without selling their assets.

**Supported Asset Types:**

| Asset | How It Works | Spending Power |
|-------|--------------|----------------|
| Real Estate | Partner with REIT, create ownership token | 50% of value |
| Vehicle | Register lien, create ownership token | 60% of value |
| Stocks | Pledge via broker, create receipt token | 50% of value |
| Crypto | Transfer to Beem vault | 40% of value |
| Wages/Income | Verify via bank data, create debt token | 70% of value |
| Cash | Transfer to Beem trust account | 95% of value |

---

## Part 1: How Tokenization Works (Simple Explanation)

### 1.1 What Is Asset Tokenization?

**Tokenization** means creating a digital record (a "token") that represents an asset. Think of it like:

- A **house deed** - a piece of paper that proves you own a house
- A **car title** - a document that proves you own a car
- A **stock certificate** - proof that you own shares in a company

Beem creates **digital versions** of these ownership records. These digital tokens are:
- Stored securely on a blockchain (a tamper-proof digital ledger)
- Used as proof of your collateral

### 1.2 Why Tokenize?

| Traditional Way | Beem Tokenization |
|-----------------|-------------------|
| Sell your stocks to get cash | Pledge stocks, keep ownership, get spending power |
| Take out a home equity loan (weeks of paperwork) | Tokenize home equity, get spending power in days |
| Payday loan with high fees | Income-based advance with transparent terms |

**Key benefit:** You don't sell your assets. You keep the upside if they grow in value, and you get them back when you repay.

### 1.3 The Simple Flow

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  1. USER     │    │  2. BEEM     │    │  3. USER     │    │  4. DONE     │
│  submits     │ ──▶│  verifies &  │ ──▶│  spends &    │ ──▶│  Asset       │
│  asset       │    │  issues CPT  │    │  repays      │    │  released    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### 1.4 How Beem Ensures Security

For tokenization to work, Beem needs to make sure users can't "double-spend" their collateral. Different asset types require different approaches:

| Asset Type | How Beem Secures It |
|------------|---------------------|
| Crypto | User transfers it to a Beem-controlled account |
| Cash | User transfers it to a Beem trust account |
| Stocks | Broker places a restriction - can't sell without Beem approval |
| Real Estate | Legal lien recorded with the county |
| Vehicle | Lien recorded with DMV - Beem is on the title |
| Wages | User signs a promissory note (legal IOU) |

### 1.5 Key Terms

| Term | What It Means |
|------|---------------|
| **Token** | A digital record representing ownership or an obligation |
| **LTV (Loan-to-Value)** | The percentage of your asset's value you can spend. Example: 50% LTV on a $100,000 house = $50,000 spending power |
| **Collateral** | The asset you pledge to secure your spending power |
| **Spending Power** | The amount you can spend via your Beem card |
| **Lien** | A legal claim on an asset (like when a bank is on your car title until you pay off the loan) |
| **Promissory Note** | A legal document where you promise to repay |

---

## Part 2: System Architecture

### 2.1 Overall Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER ASSETS                                     │
├─────────────┬─────────────┬─────────────┬─────────────┬─────────────────────┤
│   Crypto    │    Cash     │   Stocks    │ Real Estate │  Vehicle  │ Income  │
│   Wallet    │    Bank     │  Brokerage  │  Property   │           │         │
└──────┬──────┴──────┬──────┴──────┬──────┴──────┬──────┴─────┬─────┴────┬────┘
       │             │             │             │            │          │
       ▼             ▼             ▼             ▼            ▼          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              PARTNERS                                        │
│  (none)      (none)       Atomic       REIT        DMV/Title    Plaid      │
│                           Invest       Partner                              │
└──────┬──────────────┬──────────┬───────────┬───────────┬───────────┬────────┘
       │              │          │           │           │           │
       ▼              ▼          ▼           ▼           ▼           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BEEM PLATFORM                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Crypto Vault│  │  Ownership  │  │    Debt     │  │    Collateral      │ │
│  │  (holds     │  │   Tokens    │  │   Tokens    │  │     Registry       │ │
│  │   crypto)   │  │ (RWA claims)│  │  (IOUs)     │  │  (tracks all)      │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BITCOIN (Permanent Record)                                │
│                    Daily anchoring of all positions                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Asset Flow by Type

| Asset | Step 1 | Step 2 | Where It Ends Up |
|-------|--------|--------|------------------|
| **Crypto** | User transfers | Direct to vault | Crypto Vault Contract |
| **Cash** | User transfers | To trust account | Cash Trust Account |
| **Stocks** | Broker pledge | Create receipt token | Token Holding Contract |
| **Real Estate** | REIT enrollment | Create ownership token | Token Holding Contract |
| **Vehicle** | DMV lien | Create ownership token | Token Holding Contract |
| **Income** | Plaid verification | Create debt token | Token Holding Contract |

### 2.3 Component Descriptions

| Component | Purpose | Blockchain |
|-----------|---------|------------|
| **Crypto Vault Contract** | Holds deposited crypto (ETH, USDC, etc.) | Ethereum |
| **Ownership Token Contract** | Creates tokens for real-world assets | Ethereum |
| **Debt Token Contract** | Creates tokens for income-based loans | Ethereum |
| **Collateral Registry** | Central registry of all collateral positions | Ethereum |
| **BitID Identity** | User's Bitcoin-anchored identity | Bitcoin |
| **Merkle Anchoring** | Daily proof of all positions | Bitcoin |

### 2.4 Data Flow

**Deposit Flow:**
```
User ──▶ Beem App ──▶ Ethereum ──▶ Registry ──▶ User gets CPT
```

**Daily Anchoring:**
```
Beem Backend ──▶ Collect positions ──▶ Build Merkle tree ──▶ Anchor to Bitcoin
```

**Spending & Repayment:**
```
User spends ──▶ Beem processes ──▶ Update position
User repays ──▶ Release collateral ──▶ Assets returned
```

---

## Part 3: Crypto Assets

### 3.1 Overview

Crypto assets are the simplest case because:
- They're natively on-chain (no tokenization needed)
- Transfer = custody transfer
- Smart contracts can hold them directly
- Liquidation is instant (sell on DEX)

### 3.2 The Vault Design

**BeemCryptoVault** - holds user crypto deposits

| Data Stored | Description |
|-------------|-------------|
| `userDeposits` | How much each user has deposited |
| `userLocked` | Whether each user's funds are locked |
| `userLTV` | Each user's loan-to-value ratio |

| Function | What It Does |
|----------|--------------|
| `deposit()` | User adds crypto to vault |
| `withdraw()` | User takes crypto out (if unlocked) |
| `lock()` | Beem locks user's funds as collateral |
| `unlock()` | Beem releases lock after repayment |
| `liquidate()` | Beem sells user's crypto if they default |

### 3.3 Vault Functions

| Function | Who Can Call | What It Does |
|----------|--------------|--------------|
| `deposit(token, amount)` | User | Transfers tokens from user to vault |
| `withdraw(token, amount)` | User | Returns tokens to user (if not locked) |
| `lock(user)` | Beem only | Prevents user from withdrawing |
| `unlock(user)` | Beem only | Allows user to withdraw |
| `liquidate(user, buyer)` | Beem only | Transfers user's assets to buyer |
| `updateLTV(user, ltv)` | Beem only | Updates the loan-to-value ratio |
| `getPosition(user)` | Anyone | Returns user's deposit info |

### 3.4 Supported Cryptocurrencies

| Token | Type | LTV | Margin Call | Liquidation |
|-------|------|-----|-------------|-------------|
| ETH | Native | 40% | 60% LTV | 75% LTV |
| WETH | ERC-20 | 40% | 60% LTV | 75% LTV |
| USDC | Stablecoin | 85% | 90% LTV | 95% LTV |
| USDT | Stablecoin | 85% | 90% LTV | 95% LTV |
| DAI | Stablecoin | 80% | 88% LTV | 92% LTV |
| WBTC | Wrapped BTC | 45% | 65% LTV | 80% LTV |

Stablecoins have higher LTV because they're pegged to USD (low volatility).

### 3.5 Deposit Flow

```
User Wallet                    Beem Vault                    Beem Backend
     │                              │                              │
     │──── Approve & deposit ──────▶│                              │
     │      (e.g., 10,000 USDC)     │                              │
     │                              │──── Notify deposit ─────────▶│
     │                              │                              │
     │                              │                   Calculate LTV (85%)
     │                              │                   Lock collateral
     │                              │                              │
     │◀─────────────────── Issue 8,500 CPT spending power ─────────│
```

### 3.6 Withdrawal Flow (After Repayment)

```
User requests withdrawal
         │
         ▼
┌─────────────────────┐
│  Loan repaid?       │
└─────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
   NO        YES
    │         │
    ▼         ▼
 DENIED    Unlock vault ──▶ Transfer crypto back to user
```

### 3.7 Margin Call Flow

```
Price drops ──▶ Beem recalculates LTV
                        │
                        ▼
              ┌─────────────────┐
              │   LTV Level?    │
              └─────────────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
     < 60%          60-75%          > 75%
      SAFE        MARGIN CALL     LIQUIDATION
        │              │              │
        ▼              ▼              ▼
    No action     Warn user:      Auto-sell:
                  "Add more       Sell crypto on DEX
                  collateral      Pay off loan
                  or repay"       Return remainder
```

### 3.8 Security

**Multi-Signature Control**
- Vault admin is a multi-sig wallet (3-of-5)
- Major actions require multiple signatures
- Prevents single point of failure

**Time Locks**
- Large withdrawals have 24-hour delay
- Gives time to detect suspicious activity

**Audit Trail**
- All actions emit events
- Full on-chain transparency
- Anchored to Bitcoin daily

**Insurance**
- Smart contract insurance (Nexus Mutual)
- Custodial insurance for large deposits

---

## Part 4: Real Estate

### 5.1 The Challenge with Real Estate

Real estate is fundamentally different from crypto:
- It's a **physical asset** (can't move it to a smart contract)
- Ownership is **legally registered** (county recorder, title company)
- Transfer requires **legal process** (deeds, notarization)
- Value is **illiquid** (takes months to sell)

Beem cannot "hold" a house in a smart contract. Instead, we create a **digital representation** (ownership token) that is legally binding.

### 4.2 The REIT Partnership Model

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│   USER   │     │   REIT   │     │  COUNTY  │     │   BEEM   │
│ (Owner)  │     │ Partner  │     │ Records  │     │          │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │                │
     │ 1. Enroll      │                │                │
     │───────────────▶│                │                │
     │                │ 2. Verify      │                │
     │                │───────────────▶│                │
     │                │ 3. Record lien │                │
     │                │───────────────▶│                │
     │                │ 4. Issue token │                │
     │                │───────────────────────────────▶│
     │                │                │                │
     │◀───────────────────── 5. Issue spending power ──│
     │                │                │                │

Result: User still owns property. Beem holds token as security.
```

### 4.3 How It Works

**Step 1: REIT Partnership**
- Beem partners with a licensed REIT (Real Estate Investment Trust)
- REIT has legal authority to manage properties
- REIT can issue digital tokens representing property interests

**Step 2: User Enrollment**
- User owns property free and clear (or has equity)
- User "enrolls" property with the REIT
- This doesn't transfer ownership - user still owns it
- REIT becomes the "manager" of record

**Step 3: Token Issuance**
- REIT creates a unique token: `Beem-123MainSt-RealEstate-001`
- Token represents the user's equity interest
- Token is minted on Ethereum

**Step 4: Collateral Pledge**
- Token is transferred to Beem's collateral contract
- Beem records a lien on the property (county level)
- User gets spending power based on property value

**Step 5: Legal Binding**
- Tripartite agreement: User + Beem + REIT
- Agreement states: whoever holds the token controls the lien
- If user defaults, Beem can instruct REIT to sell

### 4.4 The Ownership Token

| Property | Value |
|----------|-------|
| Token ID | `Beem-123MainSt-RealEstate-001` |
| Asset Type | Real Estate |
| Property Address | 123 Main St, City, State |
| Appraised Value | $500,000 |
| LTV | 50% |
| Spending Power | $250,000 |
| Beneficial Owner | User (BitID: xyz) |
| Token Holder | Beem Collateral Contract |
| REIT Partner | ABC REIT LLC |
| Lien Status | Recorded with County |

### 4.5 Why a REIT?

REITs provide several critical capabilities:

| Capability | Why Needed |
|------------|------------|
| Property management | Legal entity to hold interest |
| Title services | Verify ownership, record liens |
| Liquidation | Legally sell property if needed |
| Regulatory compliance | Proper securities structure |
| Tax efficiency | Pass-through taxation |

### 4.6 Liquidation Process

If the user defaults:

```
Step 1: Beem instructs REIT to foreclose (token holder right)
           │
           ▼
Step 2: REIT files notice of sale with county
           │
           ▼
Step 3: 90-day sale period
           │
           ▼
Step 4: Property sold (e.g., $480,000)
           │
           ├──▶ Beem gets: $250,000 (loan) + fees
           │
           └──▶ User gets: Remainder (~$230,000 minus fees)
           
Step 5: Token burned (no longer exists)
```

### 4.7 LTV and Risk

| Property Type | LTV | Reason |
|---------------|-----|--------|
| Single Family Home | 50% | Standard residential |
| Condo/Apartment | 45% | HOA risks, less control |
| Multi-family | 55% | Income-producing |
| Commercial | 40% | Higher volatility |
| Land | 30% | Most illiquid |

Lower LTV = more buffer for price decline + liquidation costs.

---

## Part 5: Vehicles and Equipment

### 5.1 Overview

Vehicles and equipment work similarly to real estate, but with key differences:
- **Faster depreciation** (cars lose value quickly)
- **More liquid** (easier to sell at auction)
- **Simpler lien process** (DMV vs county recorder)
- **User keeps possession** (unlike crypto in vault)

### 5.2 Vehicle Tokenization Flow

```
Step 1: User requests loan against vehicle
           │
           ▼
Step 2: Beem verifies ownership via DMV (VIN lookup)
           │
           ▼
Step 3: Beem appraises vehicle via KBB API (e.g., $35,000)
           │
           ▼
Step 4: Beem files UCC-1 lien with DMV
        ┌─────────────────────────────────────────┐
        │  Lienholder: Beem Inc.                  │
        │  Vehicle: 2022 Tesla Model 3            │
        │  VIN: 5YJ3E...                          │
        └─────────────────────────────────────────┘
           │
           ▼
Step 5: Ownership token created
           │
           ▼
Step 6: User gets $21,000 CPT (60% LTV)

        ★ User keeps driving the car!
```

### 6.3 The UCC-1 Filing

**What is UCC-1?**
- Uniform Commercial Code filing
- Legal document that registers a security interest
- Filed with state Secretary of State
- Gives Beem legal right to repossess if default

**What it enables:**
- Beem can send repo agent if user defaults
- Beem has first claim on sale proceeds
- User cannot sell vehicle without Beem's release
- Title shows encumbrance (warning to buyers)

### 5.4 Vehicle Token Structure

```json
{
  "tokenId": "Beem-Tesla-5YJ3E1EA5PF123456-001",
  "assetType": "Vehicle",
  "vin": "5YJ3E1EA5PF123456",
  "make": "Tesla",
  "model": "Model 3",
  "year": 2023,
  "appraisedValue": 35000,
  "appraisalSource": "KBB",
  "appraisalDate": "2024-01-15",
  "ltv": 0.60,
  "spendingPower": 21000,
  "beneficialOwner": "bitid:user123",
  "tokenHolder": "0xBeemCollateralContract",
  "lienStatus": {
    "filed": true,
    "filingNumber": "UCC-2024-123456",
    "filedWith": "CA Secretary of State",
    "filedDate": "2024-01-15"
  },
  "status": "LOCKED"
}
```

### 5.5 Equipment Tokenization

Same process applies to equipment (boats, RVs, machinery):

| Equipment Type | LTV | Registration |
|----------------|-----|--------------|
| Motorcycle | 55% | DMV |
| Boat | 50% | Coast Guard / DMV |
| RV/Camper | 50% | DMV |
| Heavy Equipment | 45% | UCC-1 only |
| Farm Equipment | 40% | UCC-1 only |

### 5.6 Depreciation Monitoring

Unlike real estate, vehicles depreciate rapidly. Beem must monitor:

**Monthly Check Process:**
1. Query KBB API for current vehicle value
2. Compare to outstanding loan balance
3. Take action based on ratio:

| Value vs Loan | Status | Action |
|---------------|--------|--------|
| > 140% | SAFE | No action |
| 120-140% | WATCH | Monitor closely |
| 100-120% | WARNING | Notify user |
| < 100% | MARGIN CALL | Request payment or additional collateral |

### 5.7 Repossession Process

If user defaults:

1. **Notice** - Beem sends formal default notice
2. **Cure Period** - User has 15-30 days to repay
3. **Repossession** - Beem hires licensed repo agent
4. **Auction** - Vehicle sold at licensed auction
5. **Settlement** - Proceeds distributed (Beem first, remainder to user)
6. **Token Burn** - Ownership token destroyed

---

## Part 6: Income/Wages (Debt Token Model)

### 6.1 Overview - A Different Approach

Income/wages tokenization is fundamentally different from other assets:
- **No physical asset to hold** (future income isn't tangible)
- **No employer involvement** (we don't contact their boss!)
- **Token represents DEBT** (not ownership of an asset)
- **Higher risk** (employment can end)

This is essentially a **personal loan** backed by verified income, but tokenized.

### 7.2 Why No Employer Contact?

Traditional wage advance/assignment requires employer involvement:
- Employer redirects paycheck to lender
- Employer must sign agreement
- Creates awkward situation for employee

**Problems with employer involvement:**
- Employee may feel embarrassed
- Employer may view negatively
- Employer may refuse to participate
- Takes time to set up
- Affects the employment relationship

**Beem's approach: No employer contact!**
- Verify income via Plaid (bank transaction analysis)
- User repays voluntarily
- Employer never knows
- Much simpler onboarding

### 6.3 The Debt Token Model

```
STEP 1: VERIFY INCOME (No employer contact!)
┌──────────┐     ┌──────────┐     ┌──────────┐
│   User   │────▶│  Plaid   │────▶│   Beem   │
│ connects │     │ analyzes │     │ confirms │
│   bank   │     │ deposits │     │  income  │
└──────────┘     └──────────┘     └──────────┘

STEP 2: CREATE DEBT TOKEN
┌─────────────────────────────────────────────┐
│  Beem mints token representing user's debt  │
│  User receives spending power               │
└─────────────────────────────────────────────┘

STEP 3: REPAYMENT
┌──────────┐     ┌──────────┐     ┌──────────┐
│  User    │────▶│  Token   │────▶│  When    │
│  pays    │     │ balance  │     │ paid:    │
│ monthly  │     │ updates  │     │ Token    │
│          │     │          │     │ to user  │
└──────────┘     └──────────┘     └──────────┘
```

### 6.4 Key Difference: Debt Token vs Ownership Token

| Aspect | Ownership Token | Debt Token |
|--------|-----------------|------------|
| **Represents** | Asset (car, house) | Obligation (what user owes) |
| **Held by** | Beem (as collateral) | Beem (as creditor) |
| **On repayment** | Token burned | Token sent to User |
| **Transferable** | To buyer on liquidation | To collections if sold |
| **Legal meaning** | "Holder has lien on asset" | "User owes holder $X" |

### 7.5 Debt Token Structure

```json
{
  "tokenId": "Beem-Debt-User123-001",
  "tokenType": "DEBT",
  "debtor": "bitid:user123",
  "creditor": "0xBeemContract",
  "originalPrincipal": 16800,
  "currentBalance": 11760,
  "interestRate": 0.05,
  "term": "3 months",
  "monthlyPayment": 5880,
  "incomeVerification": {
    "source": "Plaid",
    "employer": "TechCorp (detected)",
    "monthlyIncome": 8000,
    "verificationDate": "2024-01-15",
    "stabilityMonths": 14
  },
  "status": "ACTIVE",
  "transferable": true,
  "legalBinding": "Promissory note signed"
}
```

### 6.6 Income Verification via Plaid

```
User ──▶ Plaid ──▶ Bank
              │
              ▼
        ┌─────────────────────────────────┐
        │  Plaid analyzes 12 months of    │
        │  transaction history:           │
        │                                 │
        │  • Detect: "TechCorp Payroll"   │
        │  • Frequency: Bi-weekly         │
        │  • Amount: ~$4,000/deposit      │
        │  • Stability: 14 months         │
        └─────────────────────────────────┘
              │
              ▼
        Plaid ──▶ Beem: Income Report
              │
              ▼
        Beem calculates: $8,000/mo × 3 months × 70% = $16,800

        ★ EMPLOYER IS NEVER CONTACTED!
```

### 6.7 The Legal Contract

Since there's no collateral to seize, the legal contract is critical:

**Promissory Note Terms:**
- User acknowledges debt to token holder
- Debt is transferable (Beem can sell it)
- Default terms and consequences
- Jurisdiction and enforcement

**Key clause:**
> "The Debtor agrees to pay the Holder of token `Beem-Debt-User123-001` the sum of $17,640, payable in 3 monthly installments of $5,880. The Holder of this token is the legal creditor. If the token is transferred, the new Holder becomes the creditor."

### 6.8 Repayment Flow

```
Month 1:  User pays $5,880 ──▶ Balance: $11,760
Month 2:  User pays $5,880 ──▶ Balance: $5,880
Month 3:  User pays $5,880 ──▶ Balance: $0 ✓

         ┌─────────────────────────────────────┐
         │  PAID IN FULL!                      │
         │  Token transferred to user          │
         │  (Proof of completion)              │
         └─────────────────────────────────────┘
```

### 6.9 If User Defaults

```
User stops paying
       │
       ▼
30-day grace period notice
       │
       ├──────────────────┐
       ▼                  ▼
    User pays          User can't pay
       │                  │
       ▼                  ▼
   Resume normal      Options:
                      • Payment plan
                      • Use other collateral
                      • Sell to collections
                      • Standard legal process
                              │
                              ▼ (if sold)
                      Collections agency
                      pursues debt
```

### 6.10 Token Goes to User on Completion

**Why send the token to the user?**

Unlike ownership tokens (which are burned), debt tokens are sent to the user when paid off:

1. **Proof of completion** - User has on-chain proof they paid
2. **Badge of honor** - Shows creditworthiness
3. **Future trust** - May qualify for better terms next time
4. **Immutable record** - Can't be disputed

The token becomes a "paid in full" receipt permanently on blockchain.

### 6.11 Risk Management

| Risk | Mitigation |
|------|------------|
| Job loss | Lower LTV (70%), shorter terms |
| Fake income | Plaid verification, 12+ month history |
| Multiple advances | Cross-check across platforms |
| Non-payment | Collections, credit reporting |
| Fraud | Identity verification via BitID |

---

## Part 7: Stocks and Brokerage Assets

### 7.1 Overview

Stocks are interesting because:
- They're valuable and liquid
- But they're held by **brokerages** (not blockchain)
- Beem can't directly "hold" them
- Need brokerage partnership

### 7.2 The Broker Partnership Model

```
Step 1: User connects brokerage ──▶ Atomic Invest reads holdings
                                          │
Step 2: User selects shares to pledge     │
                                          ▼
Step 3: Atomic places restriction on shares (can't sell without Beem OK)
                                          │
Step 4: Atomic issues receipt token ─────▶ Beem
                                          │
Step 5: Beem registers collateral & issues spending power to User

        ★ Shares stay at brokerage!
        ★ User keeps dividends & voting rights!
```

### 7.3 How It Works

**Step 1: Connect Brokerage**
- User connects brokerage via Atomic Invest API
- Beem can see holdings (read access)
- User selects shares to pledge

**Step 2: Pledge Agreement**
- User signs pledge authorization
- Brokerage places restriction on shares
- Shares cannot be sold/transferred without Beem approval
- User retains dividend rights, voting rights

**Step 3: Receipt Token**
- Beem mints receipt token representing pledged shares
- Token held in Beem's collateral contract
- Token = proof that shares are pledged

**Step 4: Spending Power**
- User gets CPT based on share value
- Shares stay at brokerage (not moved)
- User still sees shares in their account

### 7.4 Receipt Token Structure

```json
{
  "tokenId": "Beem-AAPL-500-User123-001",
  "assetType": "Stock",
  "ticker": "AAPL",
  "shares": 500,
  "priceAtPledge": 170.00,
  "valueAtPledge": 85000,
  "currentPrice": 175.00,
  "currentValue": 87500,
  "ltv": 0.50,
  "spendingPower": 42500,
  "brokerage": "Atomic Invest",
  "brokerageAccountId": "AI-12345",
  "pledgeStatus": "RESTRICTED",
  "beneficialOwner": "bitid:user123",
  "tokenHolder": "0xBeemCollateralContract",
  "dividendRights": "Retained by user",
  "votingRights": "Retained by user",
  "status": "LOCKED"
}
```

### 7.5 Real-Time Price Monitoring

Stocks are volatile - Beem must monitor continuously:

**Price Monitoring (Every 1 minute):**

| LTV Range | Status | Action |
|-----------|--------|--------|
| < 60% | SAFE | No action needed |
| 60-75% | WATCH | Monitor closely |
| 75-85% | WARNING | Notify user |
| > 85% | MARGIN CALL | User has 48 hours to act |
| > 90% | LIQUIDATION | Auto-sell shares |

### 7.6 Margin Call Process

**Example: AAPL drops to $100**

```
500 shares × $100 = $50,000 value
Loan outstanding: $42,500
LTV: 85% ──▶ MARGIN CALL!
       │
       ▼
┌─────────────────────────────────────────────────┐
│  User notified: 48 hours to respond             │
│                                                 │
│  Options:                                       │
│  1. Add more collateral                         │
│  2. Repay portion of loan                       │
│  3. Deposit cash                                │
└─────────────────────────────────────────────────┘
       │
       ├──────────────────────┐
       ▼                      ▼
   User responds          No response in 48 hrs
   (deposits $10K)              │
       │                        ▼
       ▼                   AUTO-LIQUIDATION:
   New LTV: 65%            • Sell 500 AAPL: $50,000
   Status: SAFE            • Beem gets: $42,500
                           • User gets: $7,500
```

### 8.7 Liquidation via Brokerage

Unlike crypto (DEX sell), stock liquidation goes through the brokerage:

1. Beem sends liquidation instruction to Atomic Invest
2. Atomic executes market sell order
3. Settlement: T+1 (next business day)
4. Proceeds distributed: Beem first, remainder to user
5. Receipt token burned

### 7.8 Supported Securities

| Security Type | LTV | Reason |
|---------------|-----|--------|
| Large Cap Stocks (AAPL, MSFT) | 50% | Lower volatility |
| Mid Cap Stocks | 45% | Medium volatility |
| Small Cap Stocks | 40% | Higher volatility |
| ETFs (SPY, QQQ) | 55% | Diversified |
| Bonds/Bond ETFs | 70% | Low volatility |
| REITs | 45% | Real estate exposure |
| Crypto ETFs | 35% | High volatility |

### 7.9 User Retains Rights

Important: while pledged, user keeps:
- **Dividend rights** - Dividends paid to user's account
- **Voting rights** - User can vote on shareholder matters
- **Corporate actions** - Splits, spin-offs handled normally

User loses:
- **Ability to sell** - Shares are restricted
- **Ability to transfer** - Cannot move to another brokerage
- **Ability to use as margin elsewhere** - Already pledged

---

## Part 8: Technology Architecture

### 8.1 Why Blockchain Technology?

Beem uses blockchain technology (specifically Ethereum and Bitcoin) because they provide:

| Benefit | What It Means for You |
|---------|----------------------|
| **Transparency** | Your collateral records are on a public ledger - verifiable by anyone |
| **Tamper-proof** | Once recorded, data cannot be altered or deleted |
| **No single point of failure** | Even if Beem's servers go down, your records exist on the blockchain |
| **Programmable rules** | Smart contracts automatically enforce the terms of your agreement |

**You don't need to understand blockchain** to use Beem - it works behind the scenes like the internet does for your phone.

### 8.2 The Architecture

| Layer | Components | Purpose |
|-------|------------|---------|
| **Ethereum** | Crypto Vault, Ownership Tokens, Debt Tokens, Collateral Registry | Day-to-day operations, smart contract logic |
| **Bitcoin** | BitID Identity, Merkle Root Proofs, Ordinals | Permanent record-keeping, tamper-proof anchoring |

**Daily Anchoring Process:**

```
ETHEREUM                           BITCOIN
(operations)                       (permanent record)
     │                                  │
     ▼                                  │
┌─────────────────┐                     │
│ Collateral      │                     │
│ Registry        │                     │
│ (all positions) │                     │
└────────┬────────┘                     │
         │                              │
         ▼                              │
   Beem collects                        │
   all positions                        │
         │                              │
         ▼                              │
   Build Merkle tree                    │
   (hash of all data)                   │
         │                              │
         └──────────────────────────────▶ Inscribe root to Bitcoin
                                        │
                                        ▼
                                   Permanent,
                                   tamper-proof
                                   record
```

### 8.3 Ethereum for Operations

**Ethereum provides:**

1. **Smart Contract Programmability**
   - Complex logic (lock, unlock, liquidate)
   - Automated enforcement
   - Composability with DeFi

2. **Token Standards**
   - ERC-20 for fungible tokens (CPT)
   - ERC-721 for unique tokens (ownership tokens)
   - Well-understood, auditable

3. **Fast Finality**
   - Transactions confirm in ~15 seconds
   - Suitable for real-time operations

4. **Rich Ecosystem**
   - Mature tooling (Ethers.js, Hardhat)
   - Oracles (Chainlink for prices)
   - DEXs for liquidation

**Why not Bitcoin for operations?**
- Bitcoin script is limited (no complex logic)
- No native token standard
- Slower (10-minute blocks)
- Not designed for programmability

### 8.4 Bitcoin for Permanent Records

**Bitcoin provides:**

1. **Maximum Security**
   - Most secure blockchain
   - Highest hash rate
   - Never been hacked

2. **Immutability**
   - Cannot be changed or reversed
   - Perfect for permanent proofs

3. **Trust**
   - Most recognized cryptocurrency
   - Institutional trust
   - Regulatory clarity

4. **Longevity**
   - Will exist forever
   - Proofs remain valid indefinitely

**Why not Ethereum for anchoring?**
- Less battle-tested (only since 2015)
- Proof of Stake less proven than Proof of Work
- State changes possible (TheDAO fork precedent)

### 8.5 BitID Integration

Every user has a BitID - a Bitcoin-anchored identity:

| BitID Component | What It Is |
|-----------------|------------|
| **DID Document** | Your decentralized identifier (like a digital passport) |
| **Smart Wallet Address** | Your Ethereum wallet for transactions |
| **Bitcoin Inscription** | Permanent proof of your identity on Bitcoin |

**BitID contains:**
- User's DID (Decentralized Identifier)
- Smart wallet address (Ethereum)
- Public keys for signing
- Anchored to Bitcoin via Ordinals

**Why anchor identity to Bitcoin?**
- Identity is permanent (doesn't change often)
- Maximum security for the "root of trust"
- Can verify identity without Beem's servers

### 8.6 Daily Record Anchoring

Every day at 00:00 UTC, Beem anchors the state of all positions to Bitcoin:

```
Step 1: Query all positions from Ethereum contracts
        • User A: 10 ETH
        • User B: House token
        • User C: AAPL shares
        • ...

Step 2: Build Merkle tree (hash all positions together)

Step 3: Inscribe root hash to Bitcoin via Ordinals

Step 4: Store mapping: Date ──▶ Inscription ID ──▶ Root Hash
```

### 8.7 Record Structure

```
                    [Merkle Root]
                   /              \
           [Hash AB]              [Hash CD]
          /        \              /        \
    [Hash A]    [Hash B]    [Hash C]    [Hash D]
        |           |           |           |
    Position A  Position B  Position C  Position D
    (10 ETH)    (House)     (AAPL)      (Car)
```

**Each position hash contains:**
- User BitID
- Asset type and identifier
- Current value
- Loan amount
- Status
- Timestamp

### 8.8 Verification Without Beem

Anyone can verify a position existed at a point in time:

**Example: Verify User A had 10 ETH on Jan 15**

```
Step 1: Get inscription from Bitcoin for Jan 15
        ──▶ Returns Merkle root: 0xabc123...

Step 2: Get Merkle proof from Beem (or reconstruct from public data)
        ──▶ Returns proof: [hash1, hash2, hash3]

Step 3: Verify proof against root
        ──▶ VERIFIED! (or not)
```

This works even if Beem's servers are down - the proof is on Bitcoin.

**Use cases for verification:**
- Regulatory audits
- Dispute resolution
- Insurance claims
- Estate settlement

### 8.9 Why This Architecture Works

| Concern | Ethereum Role | Bitcoin Role |
|---------|---------------|--------------|
| **Operations** | Execute smart contracts | N/A |
| **Speed** | Fast transactions | N/A |
| **Flexibility** | Complex logic | N/A |
| **Trust** | N/A | Anchor proofs |
| **Permanence** | N/A | Immutable record |
| **Identity** | Wallet addresses | BitID anchoring |
| **Audit trail** | Events/logs | Daily Merkle roots |

### 8.10 Cost Considerations

| Operation | Blockchain | Frequency | Cost |
|-----------|------------|-----------|------|
| Deposit | Ethereum | Per transaction | ~$5-50 |
| Lock/Unlock | Ethereum | Per transaction | ~$2-10 |
| Token mint | Ethereum | Per asset | ~$5-20 |
| Daily anchor | Bitcoin | Once per day | ~$5-15 |
| BitID creation | Bitcoin | Once per user | ~$5-15 |

Bitcoin anchoring is cost-effective because it's **batched** - one inscription per day covers all positions.

### 8.11 Failure Modes and Recovery

**If Ethereum goes down:**
- Operations pause temporarily
- Bitcoin proofs still exist
- Resume when Ethereum recovers

**If Bitcoin goes down:**
- Operations continue on Ethereum
- Anchoring pauses
- Historical proofs still valid

**If Beem goes down:**
- Smart contracts continue holding assets
- Users can still access via contract directly
- Bitcoin proofs verify historical state
- Recovery plan with multi-sig

---

## Part 9: Token Contracts - Deep Dive

This section provides comprehensive explanation of token contracts with detailed end-to-end examples for every asset type.

### 9.1 What Exactly Is a Token?

**Definition:** A token is a digital record on a blockchain that represents something in the real world.

| What It IS | What It Is NOT |
|------------|----------------|
| A digital receipt of your collateral pledge | NOT a currency to spend |
| Proof that your asset is held as security | NOT a collectible NFT |
| A legally enforceable claim | NOT a tradeable security |

**Purpose:** Bridge physical/legal ownership (like a house deed or car title) to a digital representation that can be managed by automated systems.

### 9.2 The Parties Involved

| Party | Role | What They Provide | What They Receive |
|-------|------|-------------------|-------------------|
| **User** | Asset owner, borrower | Asset (car, property, stocks) | USD loan, CPT spending power |
| **Beem** | Lender, contract operator | USD advance, platform | Collateral security, fees |
| **Third Party** | Asset custodian/validator | Validation, legal recognition | Partnership fees |
| **Smart Contract** | Automated enforcement | Trustless execution | N/A |

**Third Party examples:**
- Real Estate: REIT or Title Company
- Vehicle: DMV or Title Service
- Stocks: Brokerage (Atomic Invest)
- Wages: **None** (Plaid for verification only - no employer contact)
- Cash: Bank (via Plaid/Flinks)

### 10.3 What the User Brings In (Inputs)

| Asset Type | What User Provides | Documentation Required |
|------------|-------------------|----------------------|
| Real Estate | Property deed/title | Notarized agreement, REIT enrollment |
| Vehicle | Car title (pink slip) | UCC-1 lien filing, title transfer |
| Stocks | Brokerage account access | Broker authorization, pledge agreement |
| Wages | Bank account (Plaid) | Promissory note (NO employer consent needed) |
| Cash | Bank account access | Account verification, ACH authorization |
| Crypto | Wallet with assets | Transaction signature |

### 9.4 What Happens (Process)

```
PHASE 1: INITIATION
┌──────────┐     ┌──────────┐     ┌──────────────┐
│   User   │────▶│   Beem   │────▶│ Third Party  │
│ requests │     │ verifies │     │  confirms    │
│  pledge  │     │          │     │  ownership   │
└──────────┘     └──────────┘     └──────────────┘

PHASE 2: TOKEN MINTING
┌─────────────────────────────────────────────────┐
│  Create unique token                            │
│  • Status: LOCKED                               │
│  • Beneficiary: User                            │
│  • Controller: Beem                             │
└─────────────────────────────────────────────────┘

PHASE 3: REGISTRATION
  Beem ──▶ Registry: Record collateral and lien

PHASE 4: LEGAL BINDING
  Beem ──▶ Third Party: Record lien in legal system

PHASE 5: LOAN DISBURSEMENT
  Beem ──▶ User: Issue USD loan + CPT spending power
```

### 9.5 Token States and Transitions

```
Token Lifecycle:

  ┌─────────┐
  │ MINTED  │ ◀── Asset verified, token created
  └────┬────┘
       │ Collateral registered
       ▼
  ┌─────────┐
  │ LOCKED  │
  └────┬────┘
       │ Loan disbursed
       ▼
  ┌─────────┐
  │ ACTIVE  │
  └────┬────┘
       │
       ├────────────────────────┐
       │ Loan repaid            │ Default/margin call
       ▼                        ▼
  ┌──────────────┐        ┌─────────────┐
  │ PENDING      │        │ LIQUIDATING │
  │ RELEASE      │        └──────┬──────┘
  └──────┬───────┘               │
         │ Lien removed          ├──────────────┐
         ▼                       │              │
  ┌─────────┐              Asset sold    Partial sale
  │ RELEASED│              to buyer      covers debt
  └────┬────┘                   │              │
       │                        ▼              ▼
       │                   ┌────────────┐ ┌─────────┐
       │                   │ TRANSFERRED│ │ RELEASED│
       │                   └─────┬──────┘ └────┬────┘
       │                         │             │
       └───────────┬─────────────┴─────────────┘
                   ▼
              ┌─────────┐
              │ BURNED  │ ◀── Token destroyed
              └─────────┘
```

### 10.6 What the Contract Actually Does (Functions)

| Function | Who Can Call | What It Does | When Used |
|----------|--------------|--------------|-----------|
| `mint()` | Beem only | Creates new token for verified asset | Asset onboarding |
| `lock()` | Beem only | Marks token as pledged collateral | Loan initiation |
| `updateValue()` | Beem only | Updates appraised value | Revaluation |
| `release()` | Beem only | Removes collateral status | Loan repaid |
| `liquidate()` | Beem only | Initiates forced sale process | Default |
| `transfer()` | Beem only | Moves token to buyer after sale | Liquidation complete |
| `burn()` | Beem only | Destroys token permanently | Asset released or sold |
| `getOwner()` | Anyone | Returns beneficial owner | Query |
| `getStatus()` | Anyone | Returns token state | Query |
| `getTerms()` | Anyone | Returns loan terms | Query |

### 9.7 What Comes Out (Outputs)

**For the User:**
- USD loan deposited to bank account
- CPT spending power in Beem app
- Beneficial ownership retained (they still "own" the asset)
- Obligation to repay

**For Beem:**
- Ownership token held as collateral
- Right to liquidate on default
- Legal lien on the asset
- Interest/fee income

**For Third Party:**
- Partnership fees
- Updated records (lien notation)
- Liquidation execution rights (if applicable)

---

### 10.8 End-to-End Examples

---

#### Example 1: Real Estate

```
USER HAS: House worth $500,000

STEP 1: INITIATION
├─ User signs up with Beem
├─ Beem partners with ABC REIT
├─ User enrolls property in REIT structure
└─ REIT verifies ownership via title search

STEP 2: TOKEN CREATION
├─ Contract: mint("123-Main-St", userId, $500000, {ltv: 50%})
├─ Token created: Beem-123MainSt-RealEstate-001
├─ Token status: LOCKED
├─ Token held by: Beem Collateral Contract
└─ Beneficial owner: User

STEP 3: LEGAL RECORDING
├─ REIT records Beem's lien on property
├─ County recorder notified
└─ Title updated with encumbrance

STEP 4: LOAN DISBURSEMENT
├─ Spending power: $500,000 × 50% = $250,000
├─ CPT minted to user's BitID wallet
└─ User can now spend via Beem card

STEP 5A: HAPPY PATH (User Repays)
├─ User repays $250,000 + fees over time
├─ Contract: release(tokenId)
├─ REIT removes lien
├─ Contract: burn(tokenId)
└─ User has unencumbered property again

STEP 5B: DEFAULT PATH (User Doesn't Repay)
├─ Contract: liquidate(tokenId)
├─ REIT lists property for sale
├─ Property sells for $480,000
├─ Beem receives: $250,000 (principal) + fees
├─ User receives: $480,000 - $250,000 - fees = ~$220,000
├─ Contract: transfer(tokenId, buyerAddress)
└─ Contract: burn(tokenId)
```

---

#### Example 2: Wages / Future Income (DEBT TOKEN MODEL)

```
USER HAS: $8,000/month salary at TechCorp

STEP 1: INCOME VERIFICATION (NO EMPLOYER CONTACT!)
├─ User connects bank via Plaid
├─ Plaid detects recurring deposits: "TechCorp Payroll - $8,000"
├─ Beem confirms: Income = $8,000/month, stable for 12+ months
├─ User requests 3-month advance
├─ Beem calculates: 3 × $8,000 × 70% LTV = $16,800 advance
└─ EMPLOYER IS NEVER CONTACTED - only bank data used

STEP 2: DEBT TOKEN CREATION (Different from other assets!)
├─ Contract: mintDebt("Debt-User123", userId, $16800, {term: 3mo, rate: 5%})
├─ Token created: Beem-Debt-User123-001
├─ Token represents: "User123 OWES holder $16,800 + fees"
├─ Token status: ACTIVE
├─ Token held by: Beem
└─ KEY DIFFERENCE: This is a DEBT token, not an asset token!

STEP 3: LEGAL CONTRACT
├─ User signs promissory note agreement
├─ Legal binding: "Whoever holds Beem-Debt-User123-001 is owed"
│   ├─ Principal: $16,800
│   ├─ Fees: 5% over term = $840
│   └─ Total: $17,640
├─ Token is TRANSFERABLE (Beem can sell it)
└─ No employer signature needed

STEP 4: LOAN DISBURSEMENT
├─ Spending power: $16,800 CPT minted to user's wallet
├─ User can spend via Beem card
└─ User's paycheck continues to their normal bank (unchanged!)

STEP 5A: HAPPY PATH (User Repays Voluntarily)
├─ Month 1: User pays $5,880 via Beem app
│   └─ Contract: updateBalance(tokenId, $11,760 remaining)
├─ Month 2: User pays $5,880
│   └─ Contract: updateBalance(tokenId, $5,880 remaining)
├─ Month 3: User pays $5,880 (final)
│   ├─ Contract: clearDebt(tokenId)
│   ├─ Contract: transfer(tokenId, userAddress)
│   └─ User NOW HOLDS the token (proof of completion!)
└─ User's "paid off" token is a badge of completion

STEP 5B: BEEM SELLS THE DEBT (If needed)
├─ User stops paying after Month 1
├─ Remaining debt: $11,760
├─ Beem can sell token to Collections Agency
│   ├─ Contract: transfer(tokenId, collectionsAgency)
│   ├─ Collections Agency now holds token
│   └─ Per legal contract: User owes Collections Agency
├─ This is how traditional debt sales work, but on-chain
└─ User's legal obligation follows the token holder

STEP 5C: EMPLOYMENT ENDS (Risk Scenario)
├─ User loses job
├─ Beem has NO payroll redirect (never did)
├─ User still legally owes the debt
├─ Beem options:
│   ├─ a) Work out payment plan
│   ├─ b) Use other collateral if user has any
│   ├─ c) Sell debt token to collections
│   └─ d) Standard collections process
└─ Higher risk for Beem (no hard collateral)

KEY DIFFERENCES FROM OTHER ASSETS:
- TOKEN = DEBT, not ownership of asset
- NO employer involvement (just Plaid verification)
- User's paycheck is UNCHANGED (goes to their bank)
- No asset to liquidate (unsecured loan)
- Token goes TO USER when paid off (proof of completion)
- Token is TRANSFERABLE (debt can be sold)
- Higher risk → lower advance amounts
- Similar to traditional personal loan, but tokenized
```

---

#### Example 3: Vehicle (Car)

```
USER HAS: 2023 Tesla Model 3 worth $35,000

STEP 1: INITIATION
├─ User provides VIN: 5YJ3E1EA5PF123456
├─ Beem verifies ownership via DMV database
├─ Beem confirms: Clean title, no existing liens
└─ Vehicle appraised at $35,000 (via Kelley Blue Book API)

STEP 2: LIEN REGISTRATION
├─ Beem prepares UCC-1 Financing Statement
├─ Filed with state Secretary of State
├─ DMV title updated: Lienholder = Beem Inc.
└─ User keeps possession of vehicle (can still drive it)

STEP 3: TOKEN CREATION
├─ Contract: mint("VIN-5YJ3E1EA5PF123456", userId, $35000, {ltv: 60%})
├─ Token created: Beem-Tesla-5YJ3E1EA5PF123456-001
├─ Token represents: Lien on vehicle
├─ Token status: LOCKED
└─ Physical asset: User still has the car

STEP 4: LOAN DISBURSEMENT
├─ Spending power: $35,000 × 60% = $21,000
├─ CPT minted to user's BitID wallet
└─ User can spend while still driving their car

STEP 5A: HAPPY PATH (User Repays)
├─ User repays $21,000 + fees over 12 months
├─ Contract: release(tokenId)
├─ Beem files UCC-3 Termination Statement
├─ DMV removes lien from title
├─ Contract: burn(tokenId)
└─ User has clean title again

STEP 5B: DEFAULT PATH (User Doesn't Repay)
├─ Contract: liquidate(tokenId)
├─ Beem sends repo agent (legal with lien)
├─ Vehicle recovered and sold at auction: $30,000
├─ Beem receives: $21,000 (principal) + fees
├─ User receives: $30,000 - $21,000 - fees = ~$7,500
├─ Contract: transfer(tokenId, auctionBuyer)
└─ Contract: burn(tokenId)

KEY DIFFERENCES FROM REAL ESTATE:
- Faster to liquidate (repo + auction)
- Lower LTV (vehicles depreciate faster)
- User keeps possession during loan
- State DMV + UCC filing required
- More liquid market for resale
```

---

#### Example 4: Stocks / Brokerage Assets

```
USER HAS: 500 shares of AAPL worth $85,000 (at $170/share)

STEP 1: INITIATION
├─ User connects brokerage via Atomic Invest
├─ Beem sees: 500 AAPL shares @ $170 = $85,000
├─ User selects AAPL shares as collateral
└─ Atomic Invest confirms shares are available (not already pledged)

STEP 2: BROKER PLEDGE AGREEMENT
├─ User signs pledge authorization
├─ Atomic Invest places restriction on shares
├─ Shares cannot be sold/transferred without Beem approval
└─ User retains: Dividend rights, voting rights

STEP 3: TOKEN CREATION
├─ Contract: mint("AAPL-500-User123", userId, $85000, {ltv: 50%})
├─ Token created: Beem-AAPL-500-User123-001
├─ Token represents: Receipt for pledged shares
├─ Token status: LOCKED
└─ Shares remain at Atomic Invest (not moved)

STEP 4: LOAN DISBURSEMENT
├─ Spending power: $85,000 × 50% = $42,500
├─ CPT minted to user's BitID wallet
├─ User can spend while still owning shares
└─ User receives dividends directly from AAPL

STEP 5A: HAPPY PATH (User Repays)
├─ User repays $42,500 + fees
├─ Contract: release(tokenId)
├─ Atomic Invest removes restriction on shares
├─ Contract: burn(tokenId)
└─ User can trade shares freely again

STEP 5B: MARGIN CALL (Stock Price Drops)
├─ AAPL drops from $170 to $100
├─ 500 shares now worth: $50,000
├─ Current LTV: $42,500 / $50,000 = 85% (DANGER)
├─ Contract: status = MARGIN_CALL
├─ User has 48 hours to:
│   ├─ a) Add more collateral
│   └─ b) Repay portion of loan
└─ If no action → proceed to liquidation

STEP 5C: LIQUIDATION (User Doesn't Act)
├─ Contract: liquidate(tokenId)
├─ Beem instructs Atomic Invest: SELL 500 AAPL
├─ Shares sold at market: $100 × 500 = $50,000
├─ Beem receives: $42,500 (principal) + fees
├─ User receives: $50,000 - $42,500 - fees = ~$6,500
├─ Contract: burn(tokenId)
└─ Shares are gone, user has remaining cash

KEY DIFFERENCES FROM OTHER ASSETS:
- Real-time price feeds (can monitor instantly)
- Faster margin calls (stocks volatile)
- Lower LTV (50%) due to volatility
- User keeps dividend/voting rights
- Liquidation = market sell (instant)
- No physical repo needed
```

---

#### Example 5: Crypto (ETH, USDC, DOGE)

```
USER HAS: 10 ETH worth $25,000 (at $2,500/ETH)

STEP 1: INITIATION
├─ User connects MetaMask wallet
├─ Beem sees: 10 ETH @ $2,500 = $25,000
├─ User selects ETH as collateral
└─ Beem generates deposit address: Beem Vault Contract

STEP 2: ASSET TRANSFER (THE KEY DIFFERENCE!)
├─ User signs transaction: Transfer 10 ETH → Beem Vault
├─ Transaction confirmed on Ethereum blockchain
├─ ETH is NOW in Beem's control (not user's wallet)
└─ This is what gives Beem enforcement power

STEP 3: VAULT REGISTRATION
├─ Vault Contract: deposit(userId, 10 ETH, $25000)
├─ Internal accounting updated
├─ NO separate token needed (ETH itself is the collateral)
├─ Vault status for user: LOCKED
└─ Collateral Registry updated with ETH position

STEP 4: LOAN DISBURSEMENT
├─ Spending power: $25,000 × 40% = $10,000 (low LTV for crypto)
├─ CPT minted to user's BitID wallet
├─ User can spend via Beem card
└─ User's ETH sits in Beem Vault (not their wallet)

STEP 5A: HAPPY PATH (User Repays)
├─ User repays $10,000 + fees
├─ Vault Contract: unlock(userId)
├─ Vault Contract: withdraw(userId, 10 ETH)
├─ 10 ETH sent back to user's MetaMask
└─ User has their crypto back

STEP 5B: ETH PRICE DROPS (Margin Call)
├─ ETH drops from $2,500 to $1,500
├─ 10 ETH now worth: $15,000
├─ Current LTV: $10,000 / $15,000 = 67% (WARNING)
├─ Vault Contract: status = MARGIN_CALL
├─ User has 24 hours to:
│   ├─ a) Deposit more ETH
│   └─ b) Repay portion of loan
└─ If no action → liquidation

STEP 5C: LIQUIDATION
├─ Vault Contract: liquidate(userId)
├─ Beem sells 10 ETH on exchange: $15,000
├─ Beem receives: $10,000 (principal) + fees
├─ User receives: $15,000 - $10,000 - fees = ~$4,500 USDC
└─ User's ETH is gone, converted to USDC remainder

KEY DIFFERENCES FROM OTHER ASSETS:
- ACTUAL TRANSFER to Beem (not just a lien)
- No ownership token needed (crypto IS the collateral)
- Beem has REAL control (can sell without user)
- 24/7 price monitoring
- Lowest LTV (40%) due to extreme volatility
- Fastest liquidation (instant DEX sell)
- No legal filing needed (blockchain IS the registry)
```

---

#### Example 6: Cash / Bank Deposit

```
USER HAS: $50,000 in Chase savings account

STEP 1: INITIATION
├─ User connects bank via Plaid
├─ Beem sees: Chase Savings = $50,000
├─ User requests to use cash as collateral
└─ Beem explains: Must transfer to Beem-controlled account

STEP 2: CASH TRANSFER
├─ User initiates ACH transfer: $50,000 → Beem Trust Account
├─ Beem Trust Account = FDIC-insured bank partner
├─ Transfer completes in 1-3 business days
├─ Cash is NOW in Beem's custody
└─ User receives interest (passed through from Beem)

STEP 3: COLLATERAL REGISTRATION
├─ No token needed for cash (it's fungible)
├─ Beem internal ledger: User123 deposited $50,000
├─ Collateral Registry updated
├─ Status: LOCKED
└─ Cash earns ~4% APY while locked

STEP 4: LOAN DISBURSEMENT
├─ Spending power: $50,000 × 95% = $47,500 (highest LTV!)
├─ CPT minted to user's BitID wallet
├─ User can spend via Beem card
└─ Their cash is earning interest in background

STEP 5A: HAPPY PATH (User Repays)
├─ User repays $47,500 + fees
├─ Beem unlocks cash: $50,000 + accrued interest
├─ User can withdraw to Chase or spend
└─ Total returned: $50,000 + ~$500 interest

STEP 5B: USER WANTS TO SPEND MORE
├─ User already spent $30,000
├─ Remaining: $47,500 - $30,000 = $17,500 available
├─ User can add more cash to increase limit
└─ Or pay down to unlock and withdraw cash

STEP 5C: DEFAULT (User Doesn't Repay)
├─ Beem already has the cash (simplest case!)
├─ Beem deducts: $47,500 (principal) + fees from deposit
├─ User receives: $50,000 - $47,500 - fees = ~$2,000
├─ Remaining cash returned to user
└─ No liquidation auction needed

KEY DIFFERENCES FROM OTHER ASSETS:
- HIGHEST LTV (95%) - cash is stable, no volatility
- Easiest to liquidate (Beem already has it)
- User earns interest on collateral
- No ownership token needed
- No legal filing needed (it's just cash)
- Fastest repayment resolution
- Essentially "cash-secured loan"

WHY WOULD ANYONE DO THIS?
├─ User wants to keep cash earning interest
├─ User doesn't want to trigger taxable event (selling investments)
├─ User wants quick access to spending power
└─ Better than credit card (no interest if repaid on time)
```

---

### 9.9 Comparison Table: All Asset Types

| Aspect | Real Estate | Wages | Car | Stocks | Crypto | Cash |
|--------|-------------|-------|-----|--------|--------|------|
| **LTV** | 50% | 70% | 60% | 50% | 40% | 95% |
| **Control Method** | REIT + Lien | Debt Token | UCC Lien | Broker Pledge | Transfer to Vault | Transfer to Trust |
| **Third Party** | REIT | None (Plaid only) | DMV | Brokerage | None | Bank |
| **Token Type** | Ownership (asset) | Debt (obligation) | Ownership | Ownership | No token needed | No token needed |
| **Token Created?** | Yes | Yes (debt token) | Yes | Yes | No (use native) | No (use ledger) |
| **Token Destination** | Burned | Sent to User | Burned | Burned | N/A | N/A |
| **User Keeps Asset?** | Yes (lives there) | Yes (paycheck) | Yes (drives it) | Yes (dividends) | No (in vault) | No (in trust) |
| **Liquidation** | Sell property | Sell debt / collect | Repo + auction | Market sell | DEX sell | Already held |
| **Price Volatility** | Low | None | Medium | High | Very High | None |
| **Legal Filing** | County + REIT | Promissory Note | UCC-1 + DMV | Broker Agreement | Smart Contract | Trust Agreement |
| **Margin Call Risk** | Very Low | None | Low | High | Very High | None |
| **Employer Contact?** | N/A | **NO** | N/A | N/A | N/A | N/A |

---

## Part 10: Technical Implementation

Beem's tokenization system is built on four smart contracts that work together:

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   BeemVault     │  │ OwnershipToken  │  │   DebtToken     │
│   (crypto)      │  │   (RWA)         │  │   (income)      │
├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│ • deposit       │  │ • mint          │  │ • mintDebt      │
│ • withdraw      │  │ • lock          │  │ • updateBalance │
│ • lock          │  │ • release       │  │ • clearDebt     │
│ • unlock        │  │ • liquidate     │  │ • transfer      │
│ • liquidate     │  │ • transfer      │  │                 │
│                 │  │ • burn          │  │                 │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
                 ┌─────────────────────────┐
                 │   Collateral Registry   │
                 ├─────────────────────────┤
                 │ • register              │
                 │ • update                │
                 │ • remove                │
                 │ • getPosition           │
                 └─────────────────────────┘
```

| Contract | Purpose | Used For |
|----------|---------|----------|
| **BeemVault** | Holds cryptocurrency deposits | ETH, USDC, etc. |
| **OwnershipToken** | Creates tokens for real assets | Houses, cars, stocks |
| **DebtToken** | Creates tokens for income advances | Wage-based lending |
| **CollateralRegistry** | Tracks all collateral positions | All asset types |

---

## Part 11: Legal Framework

### 11.1 Overview

The legal framework bridges digital tokens to real-world ownership and enforceability.

| On-Chain (Digital) | Legal World (Real) | Connection |
|--------------------|-------------------|------------|
| Token on Ethereum | Recorded Lien | Token represents the lien |
| Token Holder (Beem) | Legal Agreement | Holder has rights per agreement |
| Smart Contract | Legal Agreement | Contract enforces the agreement |

**Key point:** The token is only valuable because it's backed by real legal documents (liens, agreements, filings).

### 11.2 Legal Agreements by Asset Type

| Asset Type | Primary Agreement | Secondary Documents |
|------------|------------------|---------------------|
| **Real Estate** | REIT Operating Agreement | Property deed, Lien recording |
| **Vehicle** | Security Agreement | UCC-1 Filing, Title lien |
| **Stocks** | Pledge Agreement | Broker authorization |
| **Wages (Debt)** | Promissory Note | Income verification |
| **Cash** | Trust Agreement | Bank authorization |
| **Crypto** | Smart Contract Terms | Wallet signature |

### 11.3 Real Estate Legal Structure

```
TRIPARTITE AGREEMENT: USER + BEEM + REIT

Parties:
1. User (Property Owner) - "Pledgor"
2. Beem Inc. - "Lender"  
3. ABC REIT LLC - "Manager"

Key Terms:
├─ User enrolls property in REIT structure
├─ REIT issues ownership token to Beem
├─ Beem holds token as collateral
├─ REIT records lien with county recorder
├─ User retains beneficial ownership (lives there, pays taxes)
├─ User agrees to repay loan per terms
├─ On default: Beem may instruct REIT to sell
├─ On repayment: Beem returns token, REIT removes lien
└─ Governing law: State where property located
```

### 12.4 Vehicle Legal Structure

```
SECURITY AGREEMENT: USER + BEEM

Parties:
1. User (Vehicle Owner) - "Debtor"
2. Beem Inc. - "Secured Party"

Key Terms:
├─ User grants security interest in vehicle
├─ Beem files UCC-1 with Secretary of State
├─ DMV records Beem as lienholder on title
├─ User retains possession (can drive it)
├─ User agrees to maintain insurance
├─ User agrees not to sell without Beem consent
├─ On default: Beem may repossess via licensed agent
├─ On repayment: Beem files UCC-3 termination
└─ Governing law: State where vehicle titled
```

### 11.5 Wage/Debt Legal Structure

```
PROMISSORY NOTE: USER (DEBTOR)

Key Terms:
├─ User promises to pay holder the principal + interest
├─ Payment schedule: Monthly installments
├─ Token represents the debt obligation
├─ Whoever holds token is the creditor
├─ Token is freely transferable
├─ On default: Holder may pursue collections
├─ On payment completion: Token transferred to user
└─ Governing law: User's state of residence

IMPORTANT CLAUSE:
"The Debtor acknowledges that the debt represented by token
[Beem-Debt-User123-001] is owed to the current Holder of
said token. Upon transfer of the token, the new Holder
becomes the creditor with all rights thereunder."
```

### 11.6 Stock Pledge Legal Structure

```
PLEDGE AGREEMENT: USER + BEEM + BROKERAGE

Parties:
1. User (Account Holder) - "Pledgor"
2. Beem Inc. - "Pledgee"
3. Atomic Invest LLC - "Custodian"

Key Terms:
├─ User pledges securities as collateral
├─ Custodian places trading restriction on shares
├─ User retains dividend and voting rights
├─ Beem receives receipt token for pledged shares
├─ On margin call: User must add collateral or repay
├─ On default: Beem instructs Custodian to liquidate
├─ On repayment: Custodian removes restriction
└─ Governed by: SEC regulations, FINRA rules
```

### 11.7 Enforcement Mechanisms

| Asset Type | How Beem Enforces | Legal Basis |
|------------|-------------------|-------------|
| **Crypto** | Smart contract control | Contract terms |
| **Real Estate** | Instruct REIT to foreclose | State foreclosure law |
| **Vehicle** | Hire repo agent | UCC Article 9 |
| **Stocks** | Instruct broker to sell | Pledge agreement |
| **Wages (Debt)** | Collections / lawsuit | Promissory note |
| **Cash** | Deduct from deposit | Trust agreement |

### 11.8 Regulatory Considerations

| Concern | How Addressed |
|---------|---------------|
| **Securities law** | Tokens are not securities (not tradeable, no profit expectation) |
| **Lending license** | Beem must be licensed in each state |
| **Consumer protection** | Clear disclosures, fair terms |
| **Privacy** | Comply with state privacy laws |
| **AML/KYC** | BitID identity verification |
| **UDAP** | No unfair or deceptive practices |

---

## Part 12: User Flow Examples

### 12.1 Summary Flow Diagrams

#### Crypto Deposit Flow

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│ Connect │──▶│ Select  │──▶│  Sign   │──▶│  Beem   │──▶│  Spend  │
│ wallet  │   │ amount  │   │   tx    │   │issues   │   │via card │
│         │   │         │   │         │   │ CPT     │   │         │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
```

#### Real Estate Flow

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│ Enroll  │──▶│  REIT   │──▶│ Record  │──▶│  Beem   │──▶│  Spend  │
│  with   │   │verifies │   │  lien   │   │issues   │   │via card │
│  REIT   │   │& mints  │   │         │   │ CPT     │   │         │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
                                                              │
                                          ┌───────────────────┴───────────────────┐
                                          ▼                                       ▼
                                     User repays                            User defaults
                                          │                                       │
                                          ▼                                       ▼
                                    Lien removed                            REIT sells
                                    Token burned                            property
```

#### Wage Advance Flow

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│ Connect │──▶│  Plaid  │──▶│  Sign   │──▶│  Beem   │──▶│  Spend  │
│  bank   │   │verifies │   │promiss. │   │issues   │   │via card │
│         │   │ income  │   │ note    │   │ CPT     │   │         │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
                                                              │
                                                              ▼
                                                        Monthly payments
                                                              │
                                          ┌───────────────────┴───────────────────┐
                                          ▼                                       ▼
                                     Fully repaid                              Default
                                          │                                       │
                                          ▼                                       ▼
                                    Token sent to                           Collections
                                    user (proof)                             process
```

### 12.2 Mobile App UI Mockups (Conceptual)

#### Home Screen
```
┌────────────────────────────────────┐
│  BEEM                         ⚙️   │
├────────────────────────────────────┤
│                                    │
│  Available Balance                 │
│  ┌────────────────────────────┐   │
│  │      $47,500.00            │   │
│  │      ────────────          │   │
│  │      CPT Spending Power    │   │
│  └────────────────────────────┘   │
│                                    │
│  [Add Collateral]  [Make Payment]  │
│                                    │
├────────────────────────────────────┤
│  Your Collateral                   │
│  ┌────────────────────────────┐   │
│  │ 🏠 123 Main St              │   │
│  │    $500,000 → $250,000 CPT │   │
│  │    Status: Active ✓        │   │
│  └────────────────────────────┘   │
│  ┌────────────────────────────┐   │
│  │ 🚗 Tesla Model 3           │   │
│  │    $35,000 → $21,000 CPT   │   │
│  │    Status: Active ✓        │   │
│  └────────────────────────────┘   │
│                                    │
├────────────────────────────────────┤
│  Recent Transactions               │
│  • Grocery Store      -$127.50    │
│  • Gas Station        -$65.00     │
│  • Restaurant         -$84.25     │
│                                    │
└────────────────────────────────────┘
```

#### Add Collateral Screen
```
┌────────────────────────────────────┐
│  ← Add Collateral                  │
├────────────────────────────────────┤
│                                    │
│  What would you like to pledge?    │
│                                    │
│  ┌────────────────────────────┐   │
│  │ 💰 Crypto                   │   │
│  │    ETH, USDC, BTC          │   │
│  │    LTV: 40-85%             │   │
│  └────────────────────────────┘   │
│                                    │
│  ┌────────────────────────────┐   │
│  │ 📈 Stocks                   │   │
│  │    Connect your brokerage  │   │
│  │    LTV: 50%                │   │
│  └────────────────────────────┘   │
│                                    │
│  ┌────────────────────────────┐   │
│  │ 🏠 Real Estate             │   │
│  │    Home equity             │   │
│  │    LTV: 50%                │   │
│  └────────────────────────────┘   │
│                                    │
│  ┌────────────────────────────┐   │
│  │ 🚗 Vehicle                  │   │
│  │    Car title loan          │   │
│  │    LTV: 60%                │   │
│  └────────────────────────────┘   │
│                                    │
│  ┌────────────────────────────┐   │
│  │ 💵 Income                   │   │
│  │    Wage advance            │   │
│  │    LTV: 70%                │   │
│  └────────────────────────────┘   │
│                                    │
└────────────────────────────────────┘
```

#### Collateral Detail Screen
```
┌────────────────────────────────────┐
│  ← 🏠 123 Main St                  │
├────────────────────────────────────┤
│                                    │
│  Property Value                    │
│  $500,000                          │
│                                    │
│  ┌─────────────────────────────┐  │
│  │ Spending Power   $250,000   │  │
│  │ Outstanding      $127,500   │  │
│  │ Available        $122,500   │  │
│  └─────────────────────────────┘  │
│                                    │
│  ────────────────────────────────  │
│  Token Details                     │
│  ────────────────────────────────  │
│  Token ID: Beem-123MainSt-RE-001  │
│  Status: Active ✓                  │
│  LTV: 50%                          │
│  Current LTV: 25.5%                │
│  REIT Partner: ABC REIT            │
│  Lien Status: Recorded             │
│                                    │
│  ────────────────────────────────  │
│  Bitcoin Proof                     │
│  ────────────────────────────────  │
│  Last anchored: 2024-01-15         │
│  Inscription: abc123...            │
│  [View on Bitcoin Explorer]        │
│                                    │
│  [Make Payment]  [Release Asset]   │
│                                    │
└────────────────────────────────────┘
```

### 12.3 API Endpoint Mapping

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/collateral/crypto/deposit` | POST | Initiate crypto deposit |
| `/v1/collateral/crypto/withdraw` | POST | Request withdrawal |
| `/v1/collateral/realestate/enroll` | POST | Enroll property |
| `/v1/collateral/vehicle/register` | POST | Register vehicle |
| `/v1/collateral/stocks/pledge` | POST | Pledge stocks |
| `/v1/collateral/income/advance` | POST | Request wage advance |
| `/v1/collateral/{id}` | GET | Get collateral details |
| `/v1/collateral/{id}/value` | GET | Get current value |
| `/v1/collateral/{id}/release` | POST | Request release |
| `/v1/payments/repay` | POST | Make repayment |
| `/v1/spending/authorize` | POST | Authorize card spend |
| `/v1/proofs/{id}/merkle` | GET | Get Merkle proof |
| `/v1/proofs/{id}/bitcoin` | GET | Get Bitcoin inscription |

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| **BitID** | Bitcoin-anchored decentralized identity |
| **CPT** | Collateral Position Token - spending power token |
| **LTV** | Loan-to-Value ratio |
| **Merkle Root** | Cryptographic hash summarizing all positions |
| **Ordinals** | Bitcoin inscription protocol |
| **REIT** | Real Estate Investment Trust |
| **UCC-1** | Uniform Commercial Code financing statement |
| **Vault** | Smart contract holding crypto assets |

---

## Appendix B: Related Documents

- [BEEM_ASSET_TOKENIZATION_IMPLEMENTATION.md](BEEM_ASSET_TOKENIZATION_IMPLEMENTATION.md) - Product/UX focused implementation
- [BLACKROCK_TOKENIZATION_EXPLAINED.md](BLACKROCK_TOKENIZATION_EXPLAINED.md) - Research on institutional tokenization

---

## Appendix C: Key Technical Concepts Summary

| Asset Type | How Beem Gains Control | Token Type | Legal Binding | On Repayment |
|------------|------------------------|------------|---------------|--------------|
| Crypto (ETH, USDC) | User transfers to Beem Vault Contract | None (native crypto) | Smart contract terms | Return crypto |
| Cash | User transfers to Beem Trust Account | None (ledger entry) | Trust agreement | Return cash |
| Real Estate | REIT creates ownership tokens, held in Beem contract | Ownership: Beem-{property} | REIT operating agreement | Burn token |
| Stocks | Brokerage creates receipt tokens | Ownership: Beem-{ticker} | Brokerage agreement | Burn token |
| Vehicle | Title lien registered, token created | Ownership: Beem-{VIN} | UCC-1 filing | Burn token |
| Wages | Income verified via Plaid, debt token created | **Debt**: Beem-Debt-{user} | Promissory note | Token to User |

