# Tech Stack and Complex Components

This document describes the **general tech stack**, **specific complex components**, and the **tools/tech used by each component** for the AI agentic workflow.

---

## 1. General Tech Stack

### 1.1 Overview

| Layer | Technology | Purpose |
|-------|------------|--------|
| **Orchestration** | LangGraph, langchain-core | State graph workflow, conditional routing, node execution |
| **Runtime** | Python 3.x, asyncio, nest-asyncio | Async request handling and event loop |
| **Transport** | WebSockets (`websockets`), FastAPI, uvicorn | Real-time client connection; HTTP/API surface |
| **LLM Access** | OpenAI, Anthropic, AWS Bedrock, Perplexity, Portkey, llm_mixture | Multi-provider chat and model selection |
| **Embeddings** | sentence-transformers (all-MiniLM-L6-v2) | Local text embeddings (384-d) |
| **Vector DB** | Pinecone | User memory, support RAG, optional LLM-mixture index |
| **Cache** | Redis | LSH cache, response cache, rate-limit state |
| **Document DB** | MongoDB (pymongo, motor) | Conversation storage, rate-limit plans, PFM data |
| **Cloud** | AWS (Bedrock, ElastiCache, ECS, ALB, SQS, SSM) | Hosting, managed services, secrets |
| **Config & Env** | python-dotenv, pydantic-settings | Environment and validated config |
| **Observability** | Portkey, custom LLMUsageManager | LLM tracing, usage, and analytics |

### 1.2 Key Dependencies (from requirements.txt)

- **Agent workflow:** `langgraph`, `langgraph-checkpoint`, `langgraph-sdk`, `langchain-core`
- **LLM SDKs:** `openai`, `anthropic`, `boto3`/`botocore`, `portkey-ai`
- **Embeddings/ML:** `sentence-transformers`, `torch`, `transformers`, `scikit-learn`, `xgboost`, `joblib`
- **Data & vector:** `pinecone`, `pinecone-plugin-assistant`, `pinecone-plugin-interface`
- **Databases:** `redis`, `pymongo`, `motor`
- **Server:** `fastapi`, `uvicorn`, `websockets`
- **Utilities:** `pydantic`, `tenacity`, `httpx`, `aiohttp`

---

## 2. Specific Complex Components and Their Tech Stack

Each subsection below describes a **complex component**, what it does, and the **tools/tech** it uses.

---

### 2.1 LangGraph Engagement Graph (Orchestration)

**What it is:** The main agent workflow implemented as a LangGraph `StateGraph` with a single shared `AgentState`. Entry is through a cache check, then rate limit, then memory, then preprocessing; from there the graph routes by intent to engagement, context, assistance, specialized GPTs, or CashGPT, and eventually to validation and response.

**Responsibilities:**

- Run the pipeline: cache → rate_limit → memory → preProcess → engagement/context/assistance/specialized_gpt/cashGPT_moveMoney/end.
- Route after engagement/context to usecase, direct_response, support, static_response_generator, or end.
- Route from usecase to step_executor or finance_gpt; from step_executor to base_gpt, specialized_gpt, or cashGPT_moveMoney.
- Handle fallback (e.g. specialized GPT → base_gpt on failure) and conditional edges at each step.

**Tech / tools:**

| Item | Technology |
|------|------------|
| Graph definition | `langgraph.graph.StateGraph`, `langgraph.graph.Graph` |
| State type | `AgentState` (TypedDict) in `src/core/state.py` |
| Entry/exit | `set_entry_point("cache")`, `set_finish_point("end")` |
| Nodes | Functions in `src/graph/nodes/` (preProcess, engagement, context, usecase, step_executor, base_gpt, specialized_gpt, etc.) |
| Routing | Async conditional edge functions (e.g. `route_after_preprocess`, `route_after_engagement`) |
| Compilation | `workflow.compile()` in `src/graph/graph_builder.py` |
| Execution | `LangGraphProcessor` in `src/processors/langgraph_processor.py` invokes the compiled graph with `requestInfo` and state |

---

### 2.2 LSH Cache (Redis-Backed Semantic Cache)

**What it is:** A semantic response cache that uses LSH (Locality-Sensitive Hashing) to find similar past prompts and return cached responses when the current prompt is close enough. Backed by Redis for persistence and scalability.

**Responsibilities:**

- Embed prompts with a fixed embedding model (384-d).
- Maintain an LSH forest (multiple tables/trees) with random projections; store buckets and metadata in Redis.
- Optional filters: time relevancy (XGBoost classifier) and location relevancy (XGBoost classifier) so cache hits respect “when” and “where” context.
- Return cached agent response when similarity and (optional) time/location checks pass.

**Tech / tools:**

| Item | Technology |
|------|------------|
| LSH structure | Custom `LSHForest` in `src/cache/lsh_cache_redis.py`: multiple tables, random projections, configurable dimension (e.g. 384) |
| Embeddings | `sentence_transformers.SentenceTransformer` (e.g. all-MiniLM-L6-v2); same as rest of stack |
| Storage | Redis: keys for projection/tree matrices, version, and per-bucket cache entries (e.g. `DS_USER:lsh:...`, `DS_USER:cache:entry:...`) |
| Redis access | `src.utils.redis_utils.RedisManager` (async) |
| Time relevancy | `src.cache.time_relevancy_classifier.time_relevancy.predict_time_relevancy` — XGBoost model (joblib) + SentenceTransformer for query→embedding |
| Location relevancy | `src.cache.location_relevancy.locational_relevancy.predict_location_relevancy` — XGBoost (joblib) + SentenceTransformer |
| Cache API | `PromptCache` in `src/cache/lsh_cache_redis.py`; `CacheManagerRedis` in `src/cache/cache_manager_redis.py` exposes `get_cached_response` / `cache_response` |
| Numerics | `numpy` for vectors and projections |

---

### 2.3 Memory System (Pinecone + Async Updater)

**What it is:** User-scoped long-term memory: sections (e.g. financial_insights, job_career, location_benchmark) stored as vectors in Pinecone and updated asynchronously. Used to inject “user memory” into the graph (e.g. in the memory node) and to persist insights from conversations.

**Responsibilities:**

- Store/retrieve per-user, per-section memory in Pinecone (vector index).
- Generate embeddings for memory content with a local SentenceTransformer model.
- Run an async memory update path (Redis + MongoDB for conversation/context; Pinecone for vectorized sections).
- Support “6 core user sections” and optional engagement-outcome storage for learning.

**Tech / tools:**

| Item | Technology |
|------|------------|
| Vector DB | Pinecone (`pinecone` SDK); index e.g. `user-memory-manager` (384-d, cosine, serverless AWS) |
| Embeddings | `sentence_transformers.SentenceTransformer` — all-MiniLM-L6-v2; cached under `src/cache/model_cache/` |
| Memory manager | `PineconeMemoryManager` in `src/memory_manager/memory_initialization.py` (singleton); creates/uses index, embeds, upserts/queries by user and section |
| Async updater | `AsyncMemoryUpdater` in `src/memory_manager/` — coordinates Redis, MongoDB, and Pinecone for conversation context and section updates |
| Graph integration | `memory_node` in `src/graph/nodes/memory.py`: loads Pinecone manager, triggers memory update, attaches `user_memory` (or similar) to state |
| Persistence | Redis and MongoDB used by the updater for session/conversation data; Pinecone for vectorized memory only |

---

### 2.4 Rate Limiter (Plan-Based, MongoDB + Optional SQS)

**What it is:** A plan-based rate limiter that decides whether a user can proceed (e.g. with a new query) based on stored plan and usage in MongoDB. Can publish events (e.g. plan activation, expiry) to SQS for downstream alerts or billing.

**Responsibilities:**

- Parse and enforce plan limits (e.g. queries per day/week/month; expiry periods like 1d, 1w, 1m, 1y).
- Record usage and transactions in MongoDB (e.g. `QueryUsage`, `QueryTransactions`, plan metadata).
- Expose “can_proceed” (or similar) so the graph’s rate_limit node routes to memory or end.
- Optionally publish transaction/expiry events to an SQS queue (e.g. for 2MOE or alerts).

**Tech / tools:**

| Item | Technology |
|------|------------|
| Persistence | MongoDB via `pymongo.MongoClient`; config from `src.config.database.db.MongoConfig` |
| Models | `src.models.rate_limit_model`: `Plans`, `QueryTransactions`, `QueryUsage`, `PlanCategory` |
| Period parsing | Custom `parse_expiry_period` (e.g. `1d`, `1w`, `1m`, `1y`, `1f`) in `src.core.rate_limiter` |
| Queue | `src.core.sqs_queue.sqs_queue` for publishing to SQS (e.g. `BLITZ_ALERTS_QUEUE`) |
| Graph integration | `rate_limit_node` in `src/graph/nodes/rate_limit.py`; sets `rate_limit_info` (e.g. `can_proceed`) on state |
| External API | Optional paywall/user-stats URL (e.g. `PAYWALL_USER_STATS_URL`) for fetching user limits |

---

### 2.5 LLM Client Layer (Multi-Provider + Portkey + Load Balancing)

**What it is:** A single abstraction that provides OpenAI-, Anthropic-, Bedrock-, Perplexity-, and (optionally) Hugging Face/Ollama clients with Portkey tracing and optional key rotation for OpenAI/Anthropic.

**Responsibilities:**

- Return correctly configured clients (OpenAI, Anthropic, Bedrock, etc.) with Portkey headers and base URL when Portkey is enabled.
- Round-robin over multiple API keys for OpenAI and Anthropic via `llm_calls_load_balancing`.
- Decorate LLM calls with usage tracking (`track_llm_usage`, `LLMUsageManager`) and optional node/tag metadata.
- Resolve API keys from env or AWS SSM (e.g. `/ds-gpt/prod/OPENAI_API_KEY`).

**Tech / tools:**

| Item | Technology |
|------|------------|
| Portkey | `portkey_ai`; `PortkeyService` in `src.services.portkey_service.py`; `create_portkey_headers` in `src.utils.portkey_helper` |
| Provider SDKs | `openai`, `anthropic`, `boto3` (bedrock-runtime) |
| Config | `PortkeyConfig`: api_key (SSM), base_url, tags, trace_id; API keys from `src.config.services.api_keys.APIKeys` |
| Load balancing | `src.utils.llm_calls_load_balancing`: `get_next_openai_key`, `get_next_anthropic_key` |
| Usage tracking | `src.utils.llm_usage_decorator.track_llm_usage`, `src.utils.llm_usage_manager.LLMUsageManager` |
| Secrets | `src.aws.ssm_param_store.ssm` for resolving key paths from env |

---

### 2.6 Base GPT and Specialized GPT Nodes

**What it is:** Two core “brain” nodes: **BaseGPT** handles general reasoning and step-wise task execution (e.g. from step_executor); **Specialized GPT** routes to internal microservices (Deals, Jobs, Budget, Price) or TaskGPT, with fallback to BaseGPT on failure.

**Responsibilities:**

- **Base GPT:** Call Anthropic (e.g. Claude) with step context, conversation history, and engagement context; output next_node (e.g. validation, response, step_executor, static/datetime/clarification generators).
- **Specialized GPT:** Map `pre_processed_info.original_node` (e.g. deals_gpt, jobs_gpt) to internal HTTP APIs or TaskGPT; build payloads with `build_gpt_payload`, call APIs with `call_gpt_api`/aiohttp; on error or “needs_fallback”, route to base_gpt (with fallback attempt limits).

**Tech / tools:**

| Item | Technology |
|------|------------|
| Base GPT LLM | Anthropic via `src.utils.llm_client.get_anthropic_client` (Portkey + metadata); model e.g. claude-sonnet-4-5-20250929 |
| Specialized APIs | Internal ALB URLs: `BUDGET_GPT_API_URL`, `DEALS_GPT_API_URL`, `JOBS_GPT_API_URL`, `PRICE_GPT_API_URL` (from `GPT_API` config) |
| HTTP client | `aiohttp` for async calls to specialized GPT services |
| Helpers | `src.utils.graph.gpt_utils`: `build_gpt_payload`, `call_gpt_api`, `prepare_response_state`, `VALID_GPT_TYPES` |
| Context | `SmartContextGenerator` for injecting smart context into prompts |
| Tracing | `TraceManager`; Portkey via the LLM client layer |

---

### 2.7 Engagement and Intent Analysis (Two-Step LLM)

**What it is:** Decides “should the AI engage?” (step 1) and “what is the intent and next node?” (step 2) using two LLM calls (e.g. one for engagement decision, one for intent categorization). Used in both group and one-on-one flows (engagement vs context node).

**Responsibilities:**

- Step 1: Engagement model evaluates conversation and user message → should_engage, reason, has_user_mention, etc.
- Step 2: If engaging, intent model categorizes intent (e.g. weather, deals, support, chitchat, user-to-user) and returns intent_category, next_node (usecase, direct_response, support, static_response_generator), and conversation_context.
- Optionally store engagement outcomes in Pinecone for learning (e.g. in group chats).

**Tech / tools:**

| Item | Technology |
|------|------------|
| Engagement LLM | Anthropic (e.g. Claude) via `get_anthropic_client`; engagement-specific model/prompt |
| Intent LLM | Same or separate model (e.g. claude-sonnet-4-5-20250929); dedicated intent prompt and JSON schema |
| State | `engagement_info`: should_engage, next_node, intent_category, conversation_context, etc. |
| Learning store | Pinecone (engagement outcome vectors) when configured |
| Prompts | Built from conversation history, last intent, and optional SmartContext; see `_create_intent_prompt` and engagement prompt in `src/graph/nodes/engagement.py` |

---

### 2.8 Support RAG (Pinecone + Embeddings)

**What it is:** A support-oriented RAG path: user query is embedded, and the Pinecone index `support-rag` is queried for similar support content; the retrieved context is then used to generate a support response.

**Responsibilities:**

- Initialize Pinecone client and index `support-rag`.
- Reuse the same embedding model as the rest of the app (via `PineconeMemoryManager` or local SentenceTransformer).
- Query by vector and (optionally) metadata; format results for the support response.

**Tech / tools:**

| Item | Technology |
|------|------------|
| Vector DB | Pinecone index `support-rag` |
| Embeddings | Same as memory: SentenceTransformer (all-MiniLM-L6-v2), optionally via `PineconeMemoryManager` |
| Node | `src/graph/nodes/support.py`: Support handler initializes Pinecone, runs query, produces support response |

---

### 2.9 LLM Mixture (Model Selection and Validation)

**What it is:** A local SDK (`llm_mixture`) used in several “generator” nodes to choose among multiple LLMs and validate response quality (e.g. with similarity or quality scores). Used for direct_response, static_response_generator, datetime_response_generator, and clarification_generation.

**Responsibilities:**

- Run multiple models (or candidates) and validate (e.g. `run_models_and_validate_in_background`).
- Select best model for a query (e.g. `select_llm_model`) using stored outcomes and optional Pinecone index `llm-mixture`.
- Return final response via `get_response`.

**Tech / tools:**

| Item | Technology |
|------|------------|
| SDK | Local wheel `llm_mixture` (e.g. `local_sdk/llm_mixture-1.2.8-py3-none-any.whl`) |
| Embeddings / similarity | sentence-transformers (and optionally Pinecone) for query similarity and model-selection feedback |
| Config | `LLM_CHOICE_ENABLED`, `SIMILARITY_THRESHOLD`, `MIN_VALIDATION_SCORE`, `PINECONE_INDEX=llm-mixture`, etc. in env |
| Nodes | `direct_response`, `static_response_generator`, `datetime_response_generator`, `clarification_generation` in `src/graph/nodes/` |

---

### 2.10 State Persistence (Pinecone User State)

**What it is:** Persists a snapshot of the agent state (e.g. after a turn) into Pinecone for retrieval or debugging. Uses the same vector indexer as other Pinecone ops (user-scoped or session-scoped).

**Responsibilities:**

- Flatten or summarize `AgentState` (or relevant parts) into a form suitable for embedding and metadata.
- Upsert (and optionally query) state vectors in Pinecone via `PineconeOperations` so state can be retrieved by similarity or by user/session id.

**Tech / tools:**

| Item | Technology |
|------|------------|
| Indexer | `src.db.vector.indexer.PineconeOperations`: Pinecone + OpenAI-compatible or local embeddings (configurable); dimension and index name from config |
| Persistence API | `persist_user_state` in `src.utils.state_persistence.py` — builds metadata, calls `PineconeOperations.upsert_user_state` |
| Embeddings | Can use same SentenceTransformer or provider passed into `PineconeOperations` |

---

## 3. External and Internal APIs (Summary)

| API / Service | Purpose |
|---------------|--------|
| **OpenAI API** | Chat completions (direct or via Portkey) |
| **Anthropic API** | Chat completions (direct or via Portkey) |
| **AWS Bedrock** | Claude models (when configured) |
| **Portkey** | Optional proxy + tracing for OpenAI/Anthropic/etc. |
| **BUDGET_GPT_API_URL, DEALS_GPT_API_URL, JOBS_GPT_API_URL, PRICE_GPT_API_URL** | Internal ALB endpoints for specialized agents |
| **PAYWALL_USER_STATS_URL** | User rate-limit / plan stats |
| **TOKENIZATION_URL** | External (e.g. VGS) webhook |
| **SQS (BLITZ_ALERTS_QUEUE)** | Alerts / event publishing (e.g. plan expiry) |
| **Pinecone** | Vector indexes: user-memory-manager, support-rag, llm-mixture |
| **Redis** | Cache, LSH structures, rate-limit state |
| **MongoDB** | Conversations, rate-limit plans/usage, PFM data |

---

## 4. Context Tracking and User Memory

This section describes how **current conversation context**, **past conversations**, and the **user memory system** are tracked and used across the pipeline.

**GPT context and memory — three sources:** For GPT calls we provide (1) **our own memory system** — long-term user preferences and facts stored in Pinecone and retrieved each turn; (2) **5 turns of previous conversations** from the same chat, stored and passed by the backend; and (3) a **`conversation_context`** field — an LLM-generated **summary** of the conversation thus far, produced during engagement/intent analysis, stored in Redis/Mongo, and passed downstream so prompts can use a concise summary as well as raw turns.

### 4.1 Current Conversation Context (In-Request)

**Where it lives:** The client sends each request with the current turn and recent messages. The server does **not** fetch conversation history from a database for the main flow; it relies on the payload.

| Field | Source | Purpose |
|-------|--------|--------|
| **`latestMessage`** | Request payload (client) | The current user message for this turn. Contains `inputInfo.content.text` (and optional `content.data` for structured content like centerBtn). |
| **`lastConversations`** | Stored and passed by backend | **5 turns** of previous messages from the same chat, used for conversational context (engagement, intent, Base GPT, specialized nodes, memory update). Newest first. |
| **`requestInfo`** | Populated in `LangGraphProcessor.initialize_state()` | Typed structure holding `sessionId`, `userId`, `userDetails`, `latestMessage`, `lastConversations`, and (after memory node) `user_memory`. |
| **`AgentState`** | LangGraph state | Carries `requestInfo` through the graph so every node can read `lastConversations` and `latestMessage`. |

**Flow:**

1. Backend stores and passes **5 turns** of previous conversations from the same chat (e.g. via state persistence and/or request payload) so each turn has recent conversational context.
2. `LangGraphProcessor.initialize_state()` builds `RequestInfo` and assigns it to `state["requestInfo"]`, including `latestMessage` and `lastConversations`. It may fix empty message entries and backfill `userId` from `primaryUser`.
3. Nodes read context from `state["requestInfo"]`:
   - **preProcess:** Uses `lastConversations` (e.g. centerBtn handling, duplicate detection).
   - **engagement / context:** Use `lastConversations` (formatted) for engagement and intent prompts.
   - **base_gpt:** Uses **5 turns** from `lastConversations` plus `user_memory` for the analysis prompt.
   - **specialized_gpt, direct_response, support, static_response_generator, cashGPT_moveMoney:** Use `lastConversations` and `request_info.get("user_memory")` where relevant.
4. **Ordering:** `lastConversations` is **newest first**; nodes that need chronological order slice and reverse as needed.

**Tech / code:**

- State type: `RequestInfo` and `AgentState` in `src/core/state.py`.
- Initialization: `src/processors/langgraph_processor.py` (`initialize_state`, `fix_empty_message_entries`, `get_primary_user_id`).
- Usage: `request_info.get("lastConversations", [])`, `request_info.get("latestMessage", {})` across `src/graph/nodes/` (engagement, context, base_gpt, usecase, support, direct_response, static_response_generator, cashGPT_moveMoney, etc.).

### 4.2 Past Conversations (5 Turns — Stored and Passed by Backend)

**Conversational context:** The backend **stores and passes 5 turns** of previous messages from the same chat so GPT and other nodes have recent conversational context. This is separate from long-term user memory (Section 4.3).

**Server-side persistence:**

| Mechanism | Where | Purpose |
|-----------|--------|--------|
| **MongoDB** | Collection from `AIGENT_CONVERSATION_COLLECTION` (e.g. `prompt_storing`) | `persist_user_state()` in `src/utils/state_persistence.py` upserts a state snapshot by `userId`, including conversation-related data. Used so the backend can store and pass conversation context (e.g. 5 turns) for the next request. |
| **Redis** | User-scoped key, TTL (e.g. 3600s) | `redis_client.update_user_state()` stores the same state snapshot for fast access; supports storing and passing recent turns. |
| **Pinecone** | Index used by `PineconeOperations` (e.g. `pine-index`) | State persistence can also upsert a vector representation of state for similarity search; see `state_persistence.py` and `PineconeOperations.upsert_user_state`. |

So: **conversational context** = 5 turns of previous conversations from the same chat, **stored and passed by the backend**. Long-term **user memory** (preferences, facts) is handled by the separate Pinecone memory system (Section 4.3).

#### 4.2.1 conversation_context (summary of conversation thus far)

We also create a **`conversation_context`** field that **summarizes** the conversation thus far. It is a string produced by the engagement/intent LLM. That LLM sees **up to 20 turns** of the conversation (`lastConversations[:20]`, newest first, formatted) in its prompt and is asked to apply “Conversation Transcript Rules” when creating the summary. After each engagement/intent run, we call **`update_conversation_context(session_id, analysis)`**, which writes to Redis and MongoDB: **`conversationContext`** (the summary), plus `lastIntent`, `lastNode`, etc. Downstream, **`engagement_info["conversation_context"]`** is passed to usecase and the router; the router appends it to the user prompt as **`Conversation Context: {conversation_context}`**, so GPT sees both the raw 5 turns (where used) and this summarized field. See `docs/MEMORY_SYSTEM.md` for more detail.

### 4.3 User Memory System (Long-Term, Pinecone)

**What it is:** Per-user, long-term **preference and factual memory** stored in Pinecone as structured **sections**. It is separate from raw conversation history: it’s a distilled, queryable store that gets updated from conversations and injected into prompts.

**Sections (6 main + optional):**

- **Main sections** (used for LLM-updatable memory and retrieval): `personal_interests`, `health_wellness`, `budget_finances`, `job_career`, `shopping_products`, `location_schedule`.
- **Other sections** (e.g. financial/location insights, not exposed to the generic LLM updater): e.g. `financial_insights`, `location_benchmark`, `job_career` (also used in memory manager).  
- Defined in `MEMORY_SECTIONS` in `src/memory_manager/memory_initialization.py`.

**Retrieval (how context is “kept” for the current turn):**

1. **When:** In the **memory** node, right after rate_limit and before preProcess.
2. **Inputs:** Current query (`latestMessage` → text), plus a short **conversation_history** string built from the last 2–3 entries in `lastConversations` (memory node builds this from `requestInfo`).
3. **Method:** `PineconeMemoryManager.get_user_memory_relevant_sections()`:
   - **Hybrid similarity:** Combines embedding similarity to the **current query** and to a **context query** (current query + conversation_history) with configurable weights (e.g. 70% current, 30% context).
   - Queries Pinecone (user-scoped vectors, section metadata), scores sections by a hybrid score, applies a similarity threshold, and returns the top-k sections (optionally filtered by `allowed_sections`).
4. **Two outputs written to state:**
   - **`requestInfo["user_memory"]`:** Formatted string of **all** relevant sections (for downstream nodes). Format: `"{userName}({userId}) learned preferences: {section1}\n{section2}..."`. Used by base_gpt, static_response_generator, and others as “user memory” in prompts.
   - **`requestInfo["retrieved_memory"]`:** Same idea but restricted to **allowed_sections** (the 6 main sections) for the LLM memory updater, so only updatable sections are passed to the update logic.

**Update (how past conversations change memory):**

1. **When:** The memory node starts an async task that calls `AsyncMemoryUpdater.prepare_memory_update()` (e.g. `_safe_memory_update`).
2. **Inputs:** Full state (including `requestInfo["latestMessage"]`, `requestInfo["lastConversations"]`, `requestInfo["retrieved_memory"]`). Conversation history for the updater is taken from **`lastConversations`** in the payload (not from MongoDB/Redis).
3. **Process:** `AsyncMemoryUpdater` (in `src/memory_manager/`):
   - Formats the last 4–5 messages from `lastConversations` for context.
   - Loads all 6 core sections from Pinecone (to avoid overwriting sections not relevant to this turn).
   - Runs LLM-based logic (and optional financial-insight logic) to decide what to add or change; then **writes back** to Pinecone via `PineconeMemoryManager.store_user_section_memory()` (or update/delete helpers).
4. **Group chats:** When `otherUsers` is present, LLM-driven memory updates can be skipped (to avoid attributing preferences to the wrong user), but financial-insight updates may still run.

**Where user memory appears in prompts:**

- **base_gpt:** `request_info.get("user_memory", "")` is injected as “User Memory & Preferences” in the analysis prompt.
- **static_response_generator / datetime_response_generator / clarification_generation:** `context.get("user_memory")` (sourced from `requestInfo["user_memory"]`) is included in system/user prompts.
- **direct_response, support, cashGPT_moveMoney:** Use `lastConversations` (and sometimes `user_memory`) as needed for context.

**Tech / code:**

| Item | Location / technology |
|------|------------------------|
| Sections & storage | `PineconeMemoryManager`, `MEMORY_SECTIONS`, `src/memory_manager/memory_initialization.py`; Pinecone index `user-memory-manager` (384-d, cosine). |
| Embeddings | `SentenceTransformer` (all-MiniLM-L6-v2), same as rest of app. |
| Retrieval | `get_user_memory_relevant_sections()`, `get_relevant_sections()`, `_calculate_hybrid_similarity()`, `_get_simple_relevant_sections()`. |
| Memory node | `src/graph/nodes/memory.py`: builds conversation snippet, calls retrieval, sets `requestInfo["user_memory"]` and `requestInfo["retrieved_memory"]`, kicks off `_safe_memory_update`. |
| Updater | `AsyncMemoryUpdater` in `src/memory_manager/`; uses `lastConversations` from state, Pinecone for read/write, optional Redis/MongoDB for supporting data. |

### 4.4 Smart Context (Theme, Scenario, Group Chat)

**What it is:** Extra context injected into prompts so responses respect **theme/subTheme**, **scenario** (e.g. friends vs public), and **group chat** rules. It does not replace conversation or user memory; it adds instructions and participant info.

**Sources:**

- **Theme / subTheme:** From `requestInfo` (e.g. `theme`, `subTheme`), often set at greeting/assistance.
- **Scenario:** `requestInfo.scenario` (e.g. `"friends"`, `"family"`, `"all"`).
- **Group vs single:** Derived from `userDetails.otherUsers`; if present, group-chat instructions and privacy tiers apply.

**Usage:** `SmartContextGenerator.generate_smart_context(request_info, conversation_context)` returns a dict with `full_context` (and related fields) that is appended to prompts in base_gpt, engagement, and other nodes. See `docs/SMART_CONTEXT_SYSTEM.md` for details.

**Summary:**

- **GPT context and memory** = (1) **Our own memory system** — Pinecone-backed user sections, retrieved each turn and exposed as `requestInfo["user_memory"]`; (2) **5 turns of previous conversations** from the same chat, stored and passed by the backend (`lastConversations` in `state.requestInfo`); (3) **`conversation_context`** — LLM-generated summary of the conversation thus far, stored in Redis/Mongo as `conversationContext`, passed via `engagement_info` to usecase and router, and appended to prompts as “Conversation Context: …”.
- **Current turn** = `latestMessage`; **conversational context** = 5 turns + the summary field; used by base_gpt, engagement, router, and other nodes.
- **User memory** = long-term preferences/facts in Pinecone; updated asynchronously from conversation by `AsyncMemoryUpdater`.

---

## 5. Document History

- **Created:** From codebase review (graph, cache, memory, rate limiter, LLM client, base/specialized GPT, engagement, support RAG, llm_mixture, state persistence).
- **Updated:** Added Section 4 (Context Tracking and User Memory). Removed Bedrock Greeting Service (not in use). Added `conversation_context` field (Section 4.2.1) and summary.
- **Location:** `docs/TECH_STACK_AND_COMPONENTS.md`
