# DID & Wallet External Integrations (APIs / Managed Services)

This document lists, for the **current DID + wallet services/routers**, what each one uses in terms of **external APIs** (e.g. UniSat) and **managed services** (e.g. AWS KMS).

Scope (code locations):
- **DID**: `src/lightning/did_router.py`, `src/lightning/did_service.py`, `src/lightning/voltage_node.py`
- **Settlement**: `src/settlement/settlement_router.py`, `src/settlement/quarterly_job.py`, `src/inscription/unisat_client.py`, `src/js/btc_payment.js`
- **Wallets**: `src/api/wallet_creation_router.py`, `src/api/wallet_router_v2.py`, `src/wallet/wallet_service.py`, `src/kms/wallet_encryption.py`
- **Treasury**: `src/api/treasury_router.py`
- **Spark (USDB)**: `src/api/spark_router.py`
- **Turnkey (legacy BRC-20 wallet)**: `src/api/turnkey_router.py`, `src/turnkey/wallet/create_wallet.py`, `src/turnkey/wallet/query_blockchain.py`
- **UniSat (BRC-20 mint/transfer router)**: `src/api/unisat_router.py`

---

## Quick map (router/service → external dependency)

### Lightning DID
- **`src/lightning/did_router.py` / `src/lightning/did_service.py`**
  - **LND REST API** (self-hosted node): via `src/lightning/voltage_node.py` (`requests` to LND REST endpoints)
  - **AWS KMS (optional)**: DID signing via `src/aws_kms/kms_signer.py`
  - **UniSat (optional / trustless verify)**: fetch settlement Merkle root from Bitcoin L1 via UniSat Open API (`src/inscription/unisat_client.py`)
  - **Redis (optional)**: cache UniSat trustless verification lookups (order→inscriptionId, inscriptionId→content)
  - **MongoDB**: DID storage + settlement lookup; plus optional PFM(Line) MongoDB for KYC lookup

### Settlement (Quarterly Merkle root on Bitcoin)
- **`src/settlement/settlement_router.py` / `src/settlement/quarterly_job.py`**
  - **UniSat Open API**: create inscription order for Merkle root JSON
  - **AWS KMS (optional)**: sign Merkle root (signature stored alongside settlement)
  - **mempool.space API**: fee estimates / UTXO + tx broadcast via Node script `src/js/btc_payment.js`
  - **AWS KMS (optional, JS path)**: `btc_payment.js` can sign BTC tx via KMS (HSM) when enabled
  - **MongoDB**: settlement records + proofs

### Wallets (BTC/ETH/CREATE2)
- **`src/api/wallet_creation_router.py`**
  - **AWS KMS**: encrypt/decrypt wallet secrets (`src/kms/wallet_encryption.py`), plus additional KMS pubkey/sign usage in some endpoints
  - **Ethereum JSON-RPC**: primary + fallback endpoints (Web3 HTTP provider)
  - **Bitcoin chain data**: uses **blockstream.info** and **mempool.space** in multiple endpoints (UTXO/tx/address info, explorer links)
  - **Spark SDK (via Node subprocess)**: derives Spark address (non-blocking helper)
  - **MongoDB**: persists wallet records and app state (e.g. conversion rates)

- **`src/api/wallet_router_v2.py` / `src/wallet/wallet_service.py`**
  - **AWS KMS**: encrypt/decrypt mnemonics/seeds (via `wallet_encryption_service`)
  - **Ethereum JSON-RPC**: used by ETH wallet/service operations
  - **Spark SDK (via Node subprocess)**: same helper pattern as above
  - **(Indirect) Treasury**: re-exports treasury endpoints/functions, which depend on mempool.space (+ signing flows)
  - **MongoDB**: wallet persistence via `src/database/wallet_repository.py`

### Treasury (Taproot multisig)
- **`src/api/treasury_router.py`**
  - **mempool.space API**: UTXO listing + fee recommendations (and explorer links)
  - **AWS KMS**: decrypt admin/user secrets where applicable (via `wallet_encryption_service`)
  - **(Indirect) tx building/broadcast**: repo JS spend scripts also use mempool.space; some flows sign via AWS KMS depending on environment

### Spark (USDB)
- **`src/api/spark_router.py`**
  - **Spark network (via Spark SDK)**: executed through `scripts/spark/spark_operations.ts` via subprocess (`node --loader ts-node/esm`)
  - **AWS KMS**: decrypt stored mnemonic(s) used to derive Spark keys
  - **MongoDB**: wallet lookup / persistence

### Turnkey (legacy BRC-20 wallet management)
- **`src/api/turnkey_router.py` / `src/turnkey/wallet/create_wallet.py`**
  - **Turnkey API**: `https://api.turnkey.com/public/v1/submit/create_sub_organization` (X-Stamp auth)
  - **UniSat Open API**: router initializes `UniSatClient` for inscription-related operations
  - **MongoDB**: stores wallet docs

- **`src/turnkey/wallet/query_blockchain.py`**
  - **BlockCypher API**: `https://api.blockcypher.com/v1/btc/main` or `/btc/test3` (address info, txs, utxos)
  - **CoinGecko API**: `https://api.coingecko.com/api/v3/simple/price` (BTC/USD price for display)

### UniSat Router (BRC-20 minting/transfers)
- **`src/api/unisat_router.py`**
  - **UniSat Open API**:
    - hardcoded endpoint: `https://open-api.unisat.io/v2/inscribe/order/create/brc20-mint`
    - also uses `UniSatClient` (`src/inscription/unisat_client.py`) for generic order + inscription fetches
  - **mempool.space API**: `https://mempool.space/api/v1/fees/recommended` (fee estimation)
  - **JS subprocess scripts**: `src/js/btc_payment.js` and `src/js/save_payment_to_queue.js` (payment automation)
  - **MongoDB**: persists KYC inscription metadata and related records

---

## External APIs: endpoints + where they’re called

### UniSat Open API (Bitcoin inscriptions / BRC-20)
Primary call sites:
- `src/inscription/unisat_client.py` (Python client via `requests`)
  - `POST /v2/inscribe/order/create`
  - `GET  /v2/inscribe/order/{orderId}`
  - `GET  /v1/indexer/inscription/info/{inscriptionId}`
  - `GET  /v1/indexer/inscription/content/{inscriptionId}`
- `src/api/unisat_router.py`
  - `POST /v2/inscribe/order/create/brc20-mint` (direct constant)
- Used by:
  - DID trustless verification path (`DIDService.get_merkle_root_from_bitcoin_l1`)
  - Settlement inscription creation (`QuarterlySettlementJob`)
  - BRC-20 mint/transfer endpoints

Key config:
- **`UNISAT_API_KEY`** (env) is supported by `create_unisat_client()`, but there are also hardcoded defaults in code.
- **`UNISAT_BASE_URL`** is set in `src/turnkey/config.py` and used by routers that instantiate `UniSatClient` directly.

### AWS KMS (encryption + signing)
Primary call sites:
- **Wallet encryption/decryption (Python)**: `src/kms/wallet_encryption.py`
  - `boto3.client("kms").encrypt(...)`
  - `boto3.client("kms").decrypt(...)`
- **DID signing (Python, optional)**: `src/aws_kms/kms_signer.py`
  - `kms.get_public_key(...)`
  - `kms.sign(MessageType="DIGEST", SigningAlgorithm="ECDSA_SHA_256")`
- **BTC tx signing (Node, optional)**: `src/js/btc_payment.js`
  - enables KMS signing path when `USE_BTC_KMS_SIGNING=true` (requires `KMS_BTC_SIGNING_KEY_ID`)

Key config:
- **`AWS_REGION`** (defaults to `us-east-1`)
- **DID signing**:
  - **`KMS_SIGNING_ENABLED=true|false`** (enables KMS signing in DID service and settlement root signing)
  - **`KMS_DID_SIGNING_KEY_ID`** (key id/arn for DID signatures)
- **Wallet encryption**:
  - **`WALLET_KMS_KEY_ID`** (or `settings.wallet_kms_key_id`) for encrypt/decrypt of wallet secrets
- **BTC signing (JS)**:
  - **`USE_BTC_KMS_SIGNING=true|false`**
  - **`KMS_BTC_SIGNING_KEY_ID`**

### LND REST API (Lightning node)
Primary call sites:
- `src/lightning/voltage_node.py` (`requests` with macaroon auth to LND REST)
- Used by `src/lightning/did_service.py` (channel state, signing fallback, keysend/settlement flows via node manager)

Key config (from `src/lightning/voltage_node.py`):
- **`VOLTAGE_API_ENDPOINT`**, **`VOLTAGE_REST_PORT`**
- **`VOLTAGE_MACAROON_PATH`**, **`VOLTAGE_TLS_CERT_PATH`**
- **`VOLTAGE_VERIFY_SSL`** (or config file)

### mempool.space API (Bitcoin data + fee estimates + broadcast in JS)
Primary call sites:
- `src/api/unisat_router.py` (fee recommendations)
- `src/api/treasury_router.py` (UTXOs + fees)
- `src/js/btc_payment.js` (fees, UTXOs, tx hex, broadcast)
- Other helpers also generate explorer links (not necessarily API calls)

Base URL:
- `https://mempool.space/api`

### blockstream.info API (Bitcoin data)
Primary call sites:
- `src/api/wallet_creation_router.py` (address/utxo calls; see grep hits)
- `src/api/wallet_router_v2.py` (address info via `https://blockstream.info/api/address/{address}`)

Base URL:
- `https://blockstream.info/api` (mainnet)
- `https://blockstream.info/testnet/api` (testnet)

### BlockCypher API (Bitcoin data, Turnkey legacy)
Primary call sites:
- `src/turnkey/wallet/query_blockchain.py`

Base URL:
- Mainnet: `https://api.blockcypher.com/v1/btc/main`
- Testnet: `https://api.blockcypher.com/v1/btc/test3`

### CoinGecko API (pricing)
Primary call sites:
- `src/turnkey/wallet/query_blockchain.py` (`/simple/price?ids=bitcoin&vs_currencies=usd`)

### Ethereum JSON-RPC (Web3)
Primary call sites:
- `src/api/wallet_creation_router.py` (fallback list includes Ankr/PublicNode/Cloudflare/etc.)
- `src/api/wallet_router_v2.py` (same pattern)
- `src/wallet/wallet_service.py` (defaults to `ETH_RPC_URL` or `settings.eth_rpc_url`)

Key config:
- **`ETH_RPC_URL`**, **`ETH_RPC_URL_FALLBACK`** (preferred)
- Default fallback examples appear in code (e.g. `https://rpc.ankr.com/eth`, `https://ethereum-rpc.publicnode.com`)

### Spark SDK (Bitcoin L2) via Node subprocess
Primary call sites:
- `src/api/spark_router.py`
- `src/api/wallet_creation_router.py` and `src/api/wallet_router_v2.py` (helper to derive Spark address)

Key config:
- **`SPARK_NETWORK`** (e.g. `MAINNET` / `REGTEST`)
- **`SPARK_USDB_TOKEN_ID`**

### Turnkey API
Primary call sites:
- `src/turnkey/wallet/create_wallet.py`
  - `POST https://api.turnkey.com/public/v1/submit/create_sub_organization` (stamped request)

---

## Notes / follow-ups

- **Hardcoded keys/endpoints**: several modules include hardcoded UniSat API keys and some signing fallbacks (not ideal for prod). This doc does not change behavior, it only inventories usage.
- **“External API” vs “external service”**: MongoDB and Redis are not “APIs” per se, but are still external dependencies and are listed where they affect these flows.

