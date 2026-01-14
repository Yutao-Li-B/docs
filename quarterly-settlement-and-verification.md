# Quarterly Settlement + DID Verification (Merkle / On-chain / Lightning)

This document describes **how quarterly settlement works**, how **DID verification works**, and what we’ve observed in benchmarks as we scale from thousands to **1,000,000** DIDs (with projections to **10,000,000**).

## Where the code lives

- **Settlement API**: `src/settlement/settlement_router.py` (`POST /api/v1/settlement/run`)
- **Settlement job**: `src/settlement/quarterly_job.py` (`QuarterlySettlementJob`)
- **Merkle tree primitives**: `src/settlement/merkle_tree.py`
- **DID verify API**: `src/lightning/did_router.py` (`POST /api/v1/did/verify`)
- **Verify implementation**: `src/lightning/did_service.py` (`verify_did(... include_blockchain=..., skip_lightning_verification=...)`)
- **Bench harness**: `scripts/merkle_settlement_benchmark.py`

## Quarterly settlement: what happens (step-by-step)

At a high level the quarterly settlement process:

1. **Select DIDs to include**
   - In normal mode, selects “unsettled” DIDs (no `settlement_quarter`).
   - In benchmark/testing mode, uses `did_limit` to pick the **latest N** by `updatedAt/createdAt` and can include already-settled DIDs when `force_resettle=true`.

2. **Build the Merkle input list**
   - Read each DID’s `merkle_hash` (leaf).
   - Sort leaf hashes (`sorted(merkle_hashes)`) to make the Merkle tree deterministic.
   - Build a mapping `hash_to_index` so each DID gets the correct proof index in the sorted list.

3. **Build the Merkle tree and compute root**
   - `build_merkle_tree(sorted_hashes)`
   - `get_merkle_root(tree)`
   - Complexity: `O(N log N)`, but in practice **small vs proof storage** at large N.

4. **(Optional) sign root with KMS**
   - KMS signing is *optional* and only affects the root signature, not proof generation.

5. **Inscribe Merkle root on Bitcoin (via UniSat)**
   - Create inscription order → pay → later fetch `inscription_id`.
   - This is mostly a **constant-time** step relative to N (seconds).

6. **Store Merkle proofs on each DID**
   - For each DID, generate proof `generate_proof(tree, index)` and write fields onto the DID doc:
     - `merkle_proof` (proof path)
     - `merkle_root`
     - `settlement_quarter`
     - `settlement_txid` (when available)
     - `settled_at`
   - If `force_resettle=true`, legacy camelCase fields (`merkleProof`, `merkleRoot`, etc.) are unset to avoid ambiguity.
   - Complexity: **`O(N)`** DB updates and dominates total runtime for large N.

7. **Save settlement metadata**
   - Stored in the settlements collection as a record for the quarter (e.g. used for “quarter → inscription order” lookup during verification).
   - **Important scaling behavior**: MongoDB has a 16MB document limit, so for large N we must *not* store the full tree / did list in one doc.

### Important fixes we applied while scaling

- **Do not mutate `merkle_hash` after generating proofs**.
  - Earlier runs showed `merkle_valid=false` because `merkle_hash` was being recomputed/overwritten after proof storage, breaking the leaf/proof relationship.
  - Fix: removed post-update `merkle_hash` rewriting in `QuarterlySettlementJob.store_did_proofs()`.

- **Avoid MongoDB “document too large” for big settlements**.
  - At `did_count=131072`, saving a settlement record that includes `tree`/`did_hashes`/`sorted_hashes` exceeded 16MB and raised:
    - `pymongo.errors.DocumentTooLarge: 'update' command document too large`
  - Fix: `save_settlement()` now **drops** `tree`, `did_hashes`, `sorted_hashes` when `did_count > 32768`, keeping only metadata (root, quarter, order id, did_count, etc.).

## DID verification: what happens (step-by-step)

The verification endpoint (`POST /api/v1/did/verify`) returns:

- `signature_valid`
- `merkle_valid`
- `onchain_valid`
- `verified` (overall)

### Verification stages

1. **Fetch DID document**
   - If request includes `did_document`, uses it directly.
   - Otherwise fetches from MongoDB using `did_id`.
   - **Typical cost drivers**:
     - network/HTTP overhead
     - Mongo lookup latency (index + storage/cache state)

2. **Signature verification**
   - Validates the signature against the issuer pubkey.
   - If signature fails, the DID may be deactivated (current behavior).
   - **Cost**: fast (cryptographic verification is typically sub-millisecond); usually not the bottleneck.

3. **Lightning verification (optional)**
   - If `skip_lightning_verification=true`, this stage is skipped.
   - Otherwise it can:
     - check `lightningSettlement` fields
     - call LND / dual-node logic to verify payment/channel state and TLV
   - In benchmarks, this is typically the **dominant cost** in “full verification”.
   - **Why it dominates**:
     - it performs **remote network calls** to Lightning nodes / services
     - latency variance (timeouts, retries, TLS handshakes, node load) swamps local CPU work
   - **Rule of thumb from our full-verify runs**: ~3 seconds per DID end-to-end, mostly attributable to this stage and its related network calls.

4. **Merkle proof verification**
   - Uses:
     - leaf: `merkle_hash`
     - proof: `merkle_proof` (snake_case from quarterly settlement) or `merkleProof` (legacy)
     - root: from the DID doc, or from Bitcoin L1 if `include_blockchain=true`
   - Complexity: `O(log2(N))` proof steps.
     - 10k → ~14 steps
     - 100k → ~17 steps
     - 1M → ~21 steps
     - 10M → ~24 steps
   - **Important: we do NOT rebuild the whole tree during verification.**
     - We verify membership by hashing **only a single “branch”** (the proof path) from the leaf up to the root.
     - Operationally, verification does:
       - start with `current = leaf_hash` (the DID’s `merkle_hash`)
       - for each proof step `(sibling_hash, position)`:
         - if `position == "left"`: `current = H(sibling_hash || current)`
         - else: `current = H(current || sibling_hash)`
       - finally check `current == merkle_root`
     - In our implementation `H(x)` is the Bitcoin-style **double-SHA256** of the concatenated bytes.
   - **Compute cost**: ~20–25 hash-pair operations even at 10M DIDs — negligible versus Lightning calls.
   - **Storage cost per DID** (proof data):
     - proof length is `~log2(N)`
     - each step stores a sibling hash + a `"left"/"right"` marker
     - so proof storage grows slowly:
       - 1M → ~21 hashes
       - 10M → ~24 hashes
     - Compared to the DID document size, this is modest, but across millions of DIDs it becomes the primary database write volume.

5. **On-chain root verification (trustless)**
   - If `include_blockchain=true` and `settlement_quarter` exists:
     - lookup quarter → `inscription_order_id`
     - resolve order → `inscription_id`
     - fetch inscription content → anchored root
     - compare anchored root == DID’s `merkle_root` (this establishes the root is anchored on-chain)
   - Redis caching makes repeated calls for the same quarter fast (root fetch becomes mostly constant-time).
   - **What is (and isn’t) proven here**:
     - This stage proves the **root** is anchored for the quarter.
     - Membership of a specific DID still requires the Merkle proof stage above (the “branch hashing”).

6. **Overall `verified` decision**
   - Current behavior (see `DIDService.verify_did`): overall `verified` depends on `signature_valid` **AND** at least one of:
     - Lightning settlement / channel state, or
     - Merkle proof validity, or
     - on-chain validity

## Benchmarks: measured results

All runs below are:

- `force_resettle=true`
- settle **latest N** DIDs (testing mode)
- verify a random sample **synchronously**

### Settlement + verify (full verification) — key points

| N (DIDs settled) | Tree levels | Settlement total | Verify sample | Verify total | Avg verify ms/DID | Notes |
|---:|---:|---:|---:|---:|---:|---|
| 4,096 | 13 | 19.5s | 20 | 63.3s | 3167ms | Full verify; merkle/onchain ok |
| 16,384 | 15 | 144.1s | 20 | 64.0s | 3201ms | (Older run) merkle_valid was impacted by `merkle_hash` mutation |
| 32,768 | 16 | 350.3s | 20 | 61.1s | 3053ms | (Older run) merkle_valid impacted by `merkle_hash` mutation |
| 131,072 | 18 | 588.3s | 20 | 61.0s | 3049ms | After fix: merkle_valid true for sample |
| 1,000,000 | 21 | 4016.5s (~66.9m) | 20 | 59.9s | 2998ms | Unsigned batch used → signature_valid true only for a few docs |

### Settlement phase breakdown (server-side)

These are from `server_benchmark.log` “Performance Metrics”.

#### 131,072 DIDs

- **Build settlement**: 3.24s  
- **Inscription**: 1.56s  
- **Save to DB**: 0.26s  
- **Store proofs**: 583.18s (**4.45ms/DID**)  
- **TOTAL**: 588.25s  

#### 1,000,000 DIDs

- **Build settlement**: 9.93s  
- **Inscription**: 1.72s  
- **Save to DB**: 0.01s  
- **Store proofs**: 4004.45s (**4.00ms/DID**)  
- **TOTAL**: 4016.11s  

### Verification cost (Merkle-only vs Full verification)

We observed two very different regimes:

- **Full verification** (signature + Lightning + Merkle + on-chain): ~**3.0s per DID**
  - Dominated by Lightning node / payment/channel lookups.
  - Merkle proof size changes with `log2(N)`, but this is negligible compared to the network cost.

- **Merkle-only-ish verification** (Lightning checks skipped): **~10–25ms per DID**
  - Example runs (verify 200 DIDs):
    - 4096 DIDs: ~12.6ms/DID
    - 8192 DIDs: ~20.8ms/DID
    - 16384 DIDs: ~17.7ms/DID
  - This regime is where `log2(N)` is actually visible, but still small.

## Scaling model (practical projections)

### Quarterly settlement scaling

Empirically, at large N the runtime is dominated by proof storage:

`T_settle(N) ≈ T_build(N) + T_inscribe + T_save + N * t_proof`

From measured runs:

- `t_proof` ≈ **4.0–4.5ms per DID** (on this hardware + MongoDB + indexes)
- `T_build(1M)` ≈ **10s**
- `T_inscribe` ≈ **~1–3s**
- `T_save` ≈ **~0–1s** (after large-settlement persistence fix)

**Rule of thumb**: **~4–5ms per DID** for settlement at scale.

## Worked example: “Quarterly settlement for 100,000,000 DIDs” (plain-English walkthrough + time)

This is a **concrete, plain-English** walkthrough of what the system would do if we ran quarterly settlement over **100,000,000 DIDs**, using the *same algorithm and storage model* we benchmarked at 1,000,000 DIDs.

### Assumptions for the estimate

- The dominant cost is **writing Merkle proofs back to MongoDB** for each DID.
- We use the observed proof-write rate from the 1,000,000 run as the baseline:
  - **~4.0–4.5 ms per DID** for “store proofs” (DB update + proof generation work inside the loop).
- “Build settlement” (sorting/building the tree) grows with N but was **single-digit seconds** at 1M; at 100M it will be larger and may require more memory / disk spill, but is still expected to be **much smaller than proof writes**.
- Bitcoin inscription / payment remains **roughly constant-time** (seconds) relative to N.
- Real-world runs at 100M likely get *worse* than linear due to storage/index growth and IO contention; treat these as best-case order-of-magnitude numbers.

### Step-by-step: what happens

1. **Pick the population (the 100,000,000 DIDs we’re settling)**
   - The system queries MongoDB to decide which DIDs belong to the quarter (in production: “unsettled”; in benchmark mode: “latest N”).
   - **Estimated time**: seconds to minutes, depending on query and index health.

2. **Read each DID’s `merkle_hash`**
   - For each DID, we read a single precomputed hash field (`merkle_hash`) used as the Merkle leaf.
   - **Estimated time**: can be significant at 100M if not streamed efficiently, but in our implementation this happens as part of building the settlement set and is typically not the largest term vs proof writes.

3. **Sort the 100,000,000 leaf hashes**
   - We sort to ensure deterministic tree construction (`sorted(merkle_hashes)`).
   - We also compute a mapping `hash_to_index` so every DID knows which leaf index its proof corresponds to.
   - **Estimated time**: “noticeable”, but expected to be far smaller than the proof-write phase.
     - Practically, this may become a memory/heap pressure point at 100M.

4. **Build the Merkle tree and compute the root**
   - We construct the Merkle tree levels and compute the final root hash.
   - Tree “height” at 100M is about `ceil(log2(100,000,000)) ≈ 27` levels.
   - **Estimated time**: likely minutes (depends heavily on implementation/memory), but still typically less than proof writes.

5. **(Optional) sign the root (KMS)**
   - One signature operation.
   - **Estimated time**: ~milliseconds to seconds, constant-time.

6. **Create & pay the Bitcoin inscription for the root**
   - UniSat order creation → payment submission.
   - **Estimated time**: a few seconds (constant-time).

7. **Generate a Merkle proof for each DID and write it back to MongoDB**
   - For each of the **100,000,000** DIDs:
     - generate the proof path (about ~27 sibling hashes and left/right markers)
     - update that DID document with:
       - `merkle_proof`, `merkle_root`, `settlement_quarter`, `settlement_txid`, `settled_at`
   - **This is the dominant step.**
   - **Estimated time (best-case linear extrapolation from 1M run)**:
     - At **4.0 ms/DID**: `100,000,000 * 0.004s = 400,000s` ≈ **111.1 hours** ≈ **4.6 days**
     - At **4.5 ms/DID**: `100,000,000 * 0.0045s = 450,000s` ≈ **125.0 hours** ≈ **5.2 days**

8. **Save the settlement metadata**
   - Store quarter → merkle_root → inscription order id, did_count, timestamps, etc.
   - For large N we do **not** store the full tree or full did list in a single MongoDB doc (16MB limit).
   - **Estimated time**: < 1 second (constant-time).

### Total time estimate (dominated by proof writes)

If you combine “constant-time” phases (seconds/minutes) plus proof writes:

- **Best-case total**: roughly **4.6–5.2 days** wall-clock for 100,000,000 DIDs
- **More realistic**: could be **longer** if:
  - the DB cannot sustain the write throughput continuously
  - indexes grow and updates slow down
  - disks saturate, cache misses increase, or other workloads compete

### 100M settlement: time budget (quick table)

This table summarizes the same steps above in a “time budget” format.

| Step | What happens | Estimated time @ 100M | What it depends on |
|---|---|---:|---|
| 1 | Select the 100M DIDs (query + sort/limit rules) | minutes (varies) | index on `updatedAt/createdAt`, query shape, cache |
| 2 | Read `merkle_hash` values | minutes (varies) | read throughput, projection, cursor streaming |
| 3 | Sort 100M leaf hashes + build `hash_to_index` | minutes–hours (varies) | RAM / spill-to-disk, Python overhead, implementation |
| 4 | Build Merkle tree levels + compute root | minutes–hours (varies) | RAM/CPU; tree level count (~27) |
| 5 | Sign root (optional) | < 1s | KMS latency (one call) |
| 6 | Inscribe root on Bitcoin (UniSat) | ~1–10s | external API + payment |
| 7 | **Generate + write proofs to 100M DID docs** | **~4.6–5.2 days** | dominant; DB write throughput + per-update cost |
| 8 | Save settlement metadata | < 1s | constant-time (after large-N persistence fix) |

If you need one “single-number” estimate, Step 7 is the one to focus on: **hundreds of thousands of seconds**.

## Worked example: “Verify one DID from a 100,000,000-DID on-chain settlement”

This is the plain-English flow for verifying a single DID that was included in a quarter whose Merkle root was anchored on-chain, where the quarter batch size is **100,000,000**.

### Key point before the steps

Verification **does not** rebuild the full Merkle tree for 100M DIDs. It uses:

- the DID’s stored **leaf**: `merkle_hash`
- the DID’s stored **proof path**: `merkle_proof` (about ~27 sibling hashes for 100M)
- the quarter’s **root**:
  - either from the DID doc (`merkle_root`)
  - and/or fetched from Bitcoin L1 (inscription content) when `include_blockchain=true`

### Step-by-step (with estimated per-step time)

Assume we call `POST /api/v1/did/verify` with:

- `did_id = <the DID>`
- `include_blockchain = true`
- `skip_lightning_verification = true` (so we isolate Merkle/on-chain behavior)

1. **Fetch DID doc from MongoDB**
   - Read fields like `merkle_hash`, `merkle_proof`, `merkle_root`, `settlement_quarter`, `signature` (if present).
   - **Estimated time**: ~5–50ms typical, but depends on DB/cache.

2. **(Optional) signature verification**
   - If the DID has a signature, verify it against issuer pubkey.
   - **Estimated time**: usually sub-millisecond.
   - Note: if you seeded without signing, `signature_valid=false` and overall `verified` can become false, but `merkle_valid`/`onchain_valid` can still be true.

3. **Fetch on-chain root for the quarter (trustless root fetch)**
   - Use `settlement_quarter` to find the quarter’s `inscription_order_id`.
   - Resolve the order to an `inscription_id`.
   - Fetch inscription content and parse the anchored Merkle root.
   - **Estimated time**:
     - **Cold cache (first verify for a quarter)**: ~0.5–3.0s (external calls dominate)
     - **Warm cache (same quarter already verified recently)**: ~5–50ms (Redis hit; mostly local)

4. **Compare on-chain root to the DID’s stored `merkle_root`**
   - This sets `onchain_valid = (onchain_root == did.merkle_root)`.
   - **Estimated time**: ~microseconds (string compare).

5. **Merkle proof verification (“branch hashing”)**
   - Take `current = merkle_hash` (leaf).
   - For each proof step `(sibling_hash, left/right)` (about ~27 steps for 100M):
     - compute `current = H(left || right)` where `H` is double-SHA256.
   - At the end check `current == root`.
   - **Estimated time**:
     - hashing **~27 steps** is typically **well under 1ms** on modern CPUs
     - the cost is effectively constant as N grows (it increases with `log2(N)`, very slowly)

### What changes between 1M and 100M verification?

- **Merkle compute**: goes from ~21 steps (1M) to ~27 steps (100M) → negligible difference.
- **On-chain root fetch**: unchanged; still “one quarter root per quarter” and cacheable.
- **Lightning verification** (if enabled): unchanged and usually dominates (seconds/DID).

### Practical “total verify time” expectations at 100M

Two common modes:

- **Merkle+on-chain (skip Lightning)**:
  - **Warm cache**: typically **tens of ms** per DID
  - **Cold cache**: typically **~1–3s** for the *first* DID in a quarter, then **tens of ms** for subsequent DIDs

- **Full verification (includes Lightning network checks)**:
  - typically **~3s per DID** regardless of whether the quarter had 1M or 100M DIDs
  - the dependency on N is effectively invisible because network calls dominate


#### Projected settlement times (using 4.0ms/DID and 4.5ms/DID)

| N | Proof-store @ 4.0ms/DID | Proof-store @ 4.5ms/DID | Expected total (rough) |
|---:|---:|---:|---|
| 10,000 | 40s | 45s | ~45–60s |
| 100,000 | 6m 40s | 7m 30s | ~7–9m |
| 1,000,000 | 1h 6m 40s | 1h 15m | ~1.1–1.3h (measured: ~66.9m) |
| 10,000,000 | 11h 6m | 12h 30m | **~11–13h** (likely higher due to IO/index growth) |

**Why 10M may be slower than linear**:
- DB cache misses and disk IO become dominant
- indexes grow and update cost increases
- longer GC / compaction / checkpoint intervals
- operational contention (other workloads, replication, etc.)

### Verification scaling

#### Merkle proof complexity

Merkle proof length is `log2(N)`. Going from:
- 10k → ~14 proof steps  
- 100k → ~17  
- 1M → ~21  
- 10M → ~24  

That’s only **+10 steps** between 10k and 10M, so the pure Merkle part grows very slowly.

#### Full verification (includes Lightning)

From observed results, full verification is dominated by Lightning and stays around:
- **~3.0s per DID** (with some variance from network conditions and cache hits)
- Only weakly dependent on N (Merkle proof length is negligible here)

#### Merkle-only verification (skip Lightning)

Observed:
- **~10–25ms per DID** for N in the 4k–16k range

Projected:
- For 10M, expect **tens of ms**, not seconds, assuming:
  - the server can fetch the quarter’s on-chain root once and cache it (Redis)
  - verification avoids Lightning calls

## Operational notes for large-N benchmarking

- **Avoid pulling 1,000,000 did_ids over HTTP**.
  - `scripts/merkle_settlement_benchmark.py` supports `--sample-from-mongo` to:
    - settle latest N via the settlement API (`did_limit=N`)
    - sample the verify set (20 IDs) directly from MongoDB

- **Unsigned synthetic DIDs**:
  - If you seed without signatures, expect:
    - `signature_valid=false`
    - `merkle_valid=true` and `onchain_valid=true` after settlement
    - overall `verified` may be false because it depends on signature validity

- **Large-settlement persistence**:
  - Settlement metadata must avoid storing per-DID lists/tree for large N (Mongo 16MB limit).
  - Per-DID proof data lives on the DID documents themselves.

## Appendix: benchmark artifacts referenced

- `results/merkle_benchmark_n4096_1768319805.json`
- `results/merkle_benchmark_n16384_1768320518.json`
- `results/merkle_benchmark_n32768_1768365241.json`
- `results/merkle_benchmark_n131072_1768367184.json`
- `results/merkle_benchmark_n1000000_1768369599.json`
- Server log: `/home/ec2-user/yutao/ds-crypto-wallets/server_benchmark.log`

