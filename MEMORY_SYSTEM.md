# Memory System

Short overview of how we handle context and memory for the AI agent.

---

## Three Parts

1. **Long-term user memory** — Our own memory system: per-user preferences and facts stored in Pinecone, retrieved each turn and injected into prompts.
2. **Conversational context** — Previous messages from the same chat, stored and passed by the backend. We receive more than 5 turns in `lastConversations`; in cost-sensitive paths (e.g. base_gpt) we **limit to 5 turns** when building prompts to optimize for cost.
3. **`conversation_context` field** — An LLM-generated **summary** of the conversation thus far. Produced during engagement/intent analysis, stored in Redis/Mongo, and passed downstream so prompts can use a concise summary instead of raw turns only.

---

## Long-Term User Memory (Pinecone)

**What it is:** Per-user memory stored as **sections** in Pinecone (vector DB). It’s updated from conversations and used to personalize responses.

**Sections (6 main):** `personal_interests`, `health_wellness`, `budget_finances`, `job_career`, `shopping_products`, `location_schedule`. Optional: `financial_insights`, `location_benchmark`. Defined in `MEMORY_SECTIONS` in `src/memory_manager/memory_initialization.py`.

**Retrieval:** In the **memory** node (after rate_limit, before preProcess), `PineconeMemoryManager.get_user_memory_relevant_sections()` runs. It uses **hybrid similarity** (current query + short conversation snippet, configurable weights) to pick relevant sections, then writes:
- **`requestInfo["user_memory"]`** — formatted string for all downstream nodes (e.g. base_gpt, static_response_generator).
- **`requestInfo["retrieved_memory"]`** — filtered to the 6 main sections for the updater.

**Update:** An async task (`AsyncMemoryUpdater`) runs after retrieval. It uses the last few messages from `lastConversations` and LLM logic to decide what to add or change, then writes back to Pinecone. In group chats, LLM memory updates can be skipped; financial-insight updates may still run.

**Embeddings:** SentenceTransformer (`all-MiniLM-L6-v2`), 384-d, same as the rest of the app. Index: `user-memory-manager` (cosine).

---

## Conversational Context (lastConversations)

**What it is:** The backend **stores and passes previous messages** from the same chat in `lastConversations` — often more than 5 turns. We use this for recent conversational context. In cost-sensitive paths (e.g. base_gpt) we **limit to 5 turns** when building prompts to optimize for cost; other nodes (e.g. engagement, context) may use more (e.g. 15–20) where needed.

**Flow:** Stored via `persist_user_state()` (MongoDB, Redis, optionally Pinecone). The backend sends `lastConversations` in the request; we put it in `requestInfo["lastConversations"]`. Order is newest first; nodes slice (e.g. [:5], [:20]) and reverse as needed.

---

## conversation_context (Summary of Conversation Thus Far)

**What it is:** A **summary** of the conversation so far, produced as a string by the engagement/intent LLM. We store it and pass it as the **`conversation_context`** field so downstream nodes (e.g. router, usecase) can inject a concise description of the dialogue instead of only raw message history.

**How it’s created:** During engagement (or context) analysis, the intent analyzer is asked to return JSON that includes `conversation_context`. It sees **up to 20 turns** of the conversation (`lastConversations[:20]`, newest first, formatted) in the prompt. The prompt tells it to apply “Conversation Transcript Rules” (chronological order, resolving “it”/“that” from recent messages, etc.) when writing this field — so the LLM summarizes or describes the conversation thus far.

**Storage:** After each engagement/intent run, we call `update_conversation_context(session_id, analysis)`. That writes to Redis and MongoDB: `conversationContext` (the summary string), plus `lastIntent`, `lastNode`, etc. So the backend stores the latest summary per session.

**Where it’s used:** Passed in `engagement_info["conversation_context"]` → usecase puts it in `engagement_context` → e.g. the router appends it to the user prompt as `Conversation Context: {conversation_context}`. So GPT sees both the raw turns we pass (e.g. 5 in base_gpt for cost) and this summarized **conversation_context** field.

---

## Where Memory Is Used

| Context | Where |
|--------|--------|
| User memory string | `request_info.get("user_memory")` in base_gpt, static_response_generator, datetime_response_generator, clarification_generation, direct_response, support, cashGPT_moveMoney |
| lastConversations (we limit to 5 in cost-sensitive paths) | `request_info.get("lastConversations", [])` in engagement, context, base_gpt, usecase, support, direct_response, static_response_generator, cashGPT_moveMoney; base_gpt uses last 5 for cost optimization |
| conversation_context (summary) | `engagement_info["conversation_context"]` → usecase → router; stored in Redis/Mongo as `conversationContext`; appended to prompts as “Conversation Context: …” |

---

## Key Code

| Piece | Location |
|-------|----------|
| Manager & sections | `PineconeMemoryManager`, `MEMORY_SECTIONS` — `src/memory_manager/memory_initialization.py` |
| Memory node | `src/graph/nodes/memory.py` |
| Updater | `AsyncMemoryUpdater` — `src/memory_manager/` |
| State persistence (conversation) | `persist_user_state` — `src/utils/state_persistence.py` |
| conversation_context (create/store) | Engagement/intent analysis and `update_conversation_context()` — `src/graph/nodes/engagement.py`, `src/graph/nodes/context.py`; consumption in `src/graph/nodes/usecase.py`, `src/graph/routers.py` |

For full pipeline and tech stack details, see `docs/TECH_STACK_AND_COMPONENTS.md` (Section 4).
