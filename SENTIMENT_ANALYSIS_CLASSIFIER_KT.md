# Knowledge Transfer: Sentiment & Topic Classification

**Production cron job (scheduled on this EC2 host)**

| Item | Value |
|------|--------|
| **When** | **10:45 IST** (India Standard Time) · **05:15 UTC** daily — crontab: `15 5 * * *` |
| **Wrapper** | `scripts/run_daily_cascaded_cron.sh` (sets `cd` to repo, pyenv `PATH`, `source .env` if present) |
| **Script actually run** | `python scripts/daily_sentiment_classifier_cascaded.py --push` |
| **Stdout/stderr log** | `results/daily_cascaded_cron.log` |

IST is **UTC+5:30** year-round (no daylight saving). `05:15 UTC` corresponds to **10:45** in India.

---

This document describes how **sentiment** and **topic (bucket / sub-topic)** classification work in this repo, with emphasis on the **cascaded LLM pipeline** used for daily production-style runs.

### Quick reference: MongoDB push and main entry points

There is **no separate “MongoDB push only” script** for the cascaded pipeline. **Writes happen inside** `scripts/daily_sentiment_classifier_cascaded.py` when you pass **`--push`**: it updates **PROD** (`app_review_ds_prod`) and then **`sync_to_dev`** copies the same classification fields to **DEV** (`app_review_ds_dev`). Without `--push`, the script still runs LLM classification and the CSV pipeline but **does not** write to MongoDB.

| How you run it | Command | MongoDB |
|----------------|---------|---------|
| **Cron (production)** | `scripts/run_daily_cascaded_cron.sh` → `python scripts/daily_sentiment_classifier_cascaded.py --push` | PROD + DEV sync |
| **Manual production run** | `python scripts/daily_sentiment_classifier_cascaded.py --push` | PROD + DEV sync |
| **Safe test (no DB writes)** | `python scripts/daily_sentiment_classifier_cascaded.py` or add `--limit N` | None |

Legacy flat-topic daily job (different schema): `scripts/daily_sentiment_classifier.py` (see table below). For CLI flags (`--log`, `--reclassify-neutral-days`, etc.), see [CLI quick reference](#cli-quick-reference).

---

## What problem it solves

User reviews arrive from **App Store**, **Play Store**, **Trustpilot**, and **Reddit**. The system:

1. Infers **sentiment** toward the app: `positive`, `negative`, or `neutral`.
2. Assigns **topics** from a controlled taxonomy: **buckets** (high-level themes) and **sub-topics** (finer labels), constrained so topics align with sentiment rules.
3. Persists results on review documents in **MongoDB** (production first, optional sync to dev).

There are **two** classification approaches in the tree:

| Approach | Entry script | Model | Topic shape |
|----------|----------------|-------|-------------|
| **Legacy daily** | `scripts/daily_sentiment_classifier.py` | OpenAI `gpt-4.1-mini` | Flat lists: `NEGATIVE_TOPICS` / `POSITIVE_TOPICS` / `NEUTRAL_TOPICS` |
| **Cascaded (current taxonomy)** | `scripts/daily_sentiment_classifier_cascaded.py` | OpenAI `gpt-4.1-mini` (via `scripts/cascaded_classifier.py`) | JSON taxonomy: buckets → sub-topics, plus bucket–sentiment rules |

The **cascaded** path matches MongoDB’s **bucket + sub-topic** design and is the one to use for new work unless you explicitly need the legacy topic lists.

---

## Cascaded classifier: high-level behavior

For each review, the cascaded stack runs **three LLM calls** in order (`scripts/cascaded_classifier.py`):

1. **Sentiment** — single label: positive / negative / neutral (financial-app–focused prompt).
2. **Buckets** — only buckets allowed for that sentiment (from `bucket_sentiment_mapping_mongodb.json` and taxonomy keys). Neutral reviews are restricted to buckets marked **both** so positive/negative-only buckets are not used loosely.
3. **Sub-topics** — labels drawn only from the chosen buckets using `sub_topic_taxonomy_cascaded.json`.

For **every allowed sentiment value, bucket name, sub-topic label, and final `subtopic-bucket` string**, see the section [Taxonomy: buckets, sub-topics, and compound keys](#taxonomy-buckets-sub-topics-and-compound-keys) (generated from the same JSON).

Invalid or empty LLM outputs are guarded with fallbacks (e.g. `general` bucket/sub-topic where allowed).

**Model** is set in `cascaded_classifier.py` as `MODEL = "gpt-4.1-mini"`.

**Debug env vars** (optional): `LOG_SENTIMENT`, `SENTIMENT_EXPLAIN` — extra logging / explanation for sentiment calls.

---

## OpenAI API key (environment vs AWS SSM Parameter Store)

Implementation: `scripts/openai_api_key.py` (`resolve_openai_api_key()`). **All OpenAI usage in this repo** goes through this helper (classification, evaluation, taxonomy discovery, rating benchmarks, etc.) so **`OPENAI_API_KEY` vs SSM** behaves the same everywhere. Import it after ensuring the repo root is on `sys.path` (patterns already used in `scripts/*.py`, top-level `*.py` under the notebooks root, and `benchmarks/`).

**Resolution order**

1. **`OPENAI_API_KEY`** — If set to a non-empty value in the process environment, it is used (local override, CI, or `.env` loaded by `python-dotenv` / the shell).
2. **AWS Systems Manager Parameter Store** — If the variable is unset, the key is read with **`boto3`** via `ssm.get_parameter(Name=…, WithDecryption=True)`.

**SSM-related environment variables**

| Variable | Role |
|----------|------|
| `OPENAI_API_KEY_SSM_PATH` | Parameter **name** (default: `/ds-gpt/dev/OPENAI_API_KEY`) |
| `AWS_REGION` | Region for the SSM client (default: `us-east-1`) |

**IAM**: The runtime identity needs **`ssm:GetParameter`** on that parameter (and decrypt permission if it is a **SecureString**). On EC2, the **instance profile** is the usual source of credentials.

**Cron on this host**: `scripts/run_daily_cascaded_cron.sh` runs `daily_sentiment_classifier_cascaded.py --push`. It `cd`s to the repo, prepends pyenv shims to `PATH`, and **`source`s `.env`** when the file exists. If `OPENAI_API_KEY` is set in `.env`, that value **wins** over SSM. If it is absent or commented out, **`resolve_openai_api_key()` loads the key from SSM** at `OPENAI_API_KEY_SSM_PATH`.

**Optional stored value format**: If the parameter string contains `:`, the code may split on `:` and use the **last** segment as the secret (for `prefix:sk-…`-style values).

**Dependency**: `boto3` (see `requirements.txt`) is required only when resolving via SSM.

---

## End-to-end flow: `daily_sentiment_classifier_cascaded.py`

**Overall pipeline** (left → right):

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. FETCH (MongoDB PROD)                                                                   │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│  • Default: no sentiment  OR  classification_source = daily_cron                          │
│  • Optional (--reclassify-neutral-days): neutral + catch-all bucket text, last N days   │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 2. CASCADED LLM (in-process, per review)                                                  │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│   ┌───────────┐      ┌────────────────────────────┐      ┌─────────────────────────┐       │
│   │ Sentiment │  →   │ Buckets (sentiment-allowed)│  →   │ Sub-topics per bucket   │       │
│   └───────────┘      └────────────────────────────┘      └─────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 3. TEMP CSV                                                                               │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│  results/daily_cascaded_temp_classified.csv                                               │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 4. run_pipeline (subprocess chain)                                                        │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│   ┌────────────────────────────┐                                                          │
│   │ reclassify_subtopic_other  │  general/other → specific sub-topics                     │
│   └─────────────┬──────────────┘                                                          │
│                 ▼                                                                         │
│   ┌────────────────────────────┐                                                          │
│   │ filter_topics_by_sentiment │  mapping + overrides / exclusivity                      │
│   └─────────────┬──────────────┘                                                          │
│                 ▼                                                                         │
│   ┌────────────────────────────┐                                                          │
│   │ copy                       │  → classification_results_cascaded_final.csv            │
│   └─────────────┬──────────────┘                                                          │
│                 ▼                                                                         │
│   ┌────────────────────────────┐                                                          │
│   │ reduce_to_single_topic     │  → classification_results_cascaded_single_topic.csv     │
│   └─────────────┬──────────────┘                                                          │
│                 ▼                                                                         │
│   ┌────────────────────────────┐                                                          │
│   │ condense_subtopic_labels   │  short display labels (in-place on single-topic CSV)     │
│   └────────────────────────────┘                                                          │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 5. MONGODB (only with --push)                                                             │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│  PROD: sentiment, bucket_classification, classification_source, classified_at               │
│  DEV:  same four fields synced by sync_id_field                                           │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### Fetch rules

- **Default batch**: documents where `sentiment` is missing **or** `classification_source` is `"daily_cron"` (so older cron labels can be replaced by cascaded).
- **Optional re-run**: `--reclassify-neutral-days N` pulls **neutral** reviews whose `bucket_classification` matches broad “other neutral / general neutral” style patterns and `classified_at` within the last **N** days.

Per collection, **text** and **IDs** come from `COLLECTION_CONFIG` in `daily_sentiment_classifier_cascaded.py` (e.g. App/Play: `body`/`review_text`, `title`, `review_id`; Trustpilot: `review_text_description`, `review_title`, `_id`; Reddit: `body`, `title`, `post_id`).

### Post-classification pipeline (`run_pipeline`)

Defined in `scripts/cascaded_classifier.py`; `daily_sentiment_classifier_cascaded.py` always runs it after writing the temp CSV.

| Step | Script | Output artifact |
|------|--------|------------------|
| Reclassify vague sub-topics | `scripts/reclassify_subtopic_other.py` | `results/classification_results_cascaded_reclassified.csv` |
| Sentiment-consistent topics | `scripts/filter_topics_by_sentiment.py` | `results/classification_results_cascaded_sentiment_filtered.csv` |
| Final multi-topic copy | (file copy) | `results/classification_results_cascaded_final.csv` |
| Single primary topic | `scripts/reduce_to_single_topic_per_review.py` | `results/classification_results_cascaded_single_topic.csv` |
| Short labels | `scripts/condense_subtopic_labels_in_csv.py` `--in-place` | Same single-topic path (labels condensed in CSV) |

`filter_topics_by_sentiment.py` uses:

- `results/bucket_sentiment_mapping_mongodb.json`
- `results/sub_topic_sentiment_overrides.json` (optional)
- `results/sub_topic_sentiment_exclusivity.json` (optional)

### MongoDB writes (`--push`)

This is the **same** process the **Production cron job** table at the top of this document uses when it passes **`--push`** (see also [Quick reference: MongoDB push and main entry points](#quick-reference-mongodb-push-and-main-entry-points)).

- **Default**: **no writes** to MongoDB unless you pass **`--push`**.
- **Production DB**: `app_review_ds_prod` (URI from env `MONGODB_URI_PROD`; scripts may contain fallback literals—prefer env in production).
- **Dev DB**: `app_review_ds_dev` (`MONGODB_URI_DEV`) — updated via **`sync_to_dev`** after successful PROD updates (same four fields per collection).

Fields set on each updated review (`update_review_cascaded`):

| Field | Meaning |
|-------|--------|
| `sentiment` | `positive` / `negative` / `neutral` |
| `bucket_classification` | Value taken from the **first** topic in the single-topic row after pipeline (typically the condensed `sub_topic` string, or fallback to `bucket`) |
| `classification_source` | `"daily_cascaded"` |
| `classified_at` | UTC timestamp |

After a successful prod update, **`sync_to_dev`** copies those four fields to dev using each collection’s `sync_id_field`.

### CLI quick reference

Run from the **repo root** (`cd` to `notebooks/`). Use the same `python` as cron (e.g. pyenv shims) so imports resolve.

**Daily cascaded job (PROD fetch → classify → pipeline → optional MongoDB)**

```bash
# Dry-run: full pipeline, no MongoDB writes
python scripts/daily_sentiment_classifier_cascaded.py

# Limit reviews per collection (testing)
python scripts/daily_sentiment_classifier_cascaded.py --limit 10

# Production: write PROD + sync DEV (same as cron, minus wrapper env)
python scripts/daily_sentiment_classifier_cascaded.py --push

# Reclassify neutral “catch-all” reviews from last N days + push
python scripts/daily_sentiment_classifier_cascaded.py --reclassify-neutral-days 7 --push

# Append run log (cron uses results/daily_cascaded_cron.log via shell redirect)
python scripts/daily_sentiment_classifier_cascaded.py --log /path/to/run.log --push
```

**Batch / research (`cascaded_classifier.py` — dev samples or local JSON, CSV out)**

```bash
# Sample from MongoDB dev, write CSV (no daily PROD job)
python scripts/cascaded_classifier.py --mongo

# Same + post-pipeline (reclassify → filter → single-topic → condense)
python scripts/cascaded_classifier.py --mongo --pipeline

# Pipeline only on an existing classified CSV
python scripts/cascaded_classifier.py --pipeline-only results/some_classified.csv
```

**Legacy daily (flat topics, different MongoDB fields)**

```bash
python scripts/daily_sentiment_classifier.py --dry-run
python scripts/daily_sentiment_classifier.py --limit 5
# Default: LIVE MongoDB writes (no --dry-run). Use --dry-run for safe testing.
```

**Dependencies** (typical): `openai`, `pymongo`, `tqdm`, and **`boto3`** when using SSM for the API key (see [OpenAI API key (environment vs AWS SSM Parameter Store)](#openai-api-key-environment-vs-aws-ssm-parameter-store)).

---

## Standalone cascaded batch (no daily prod job)

`scripts/cascaded_classifier.py` can:

- Classify from **local JSON** (`--input`) or **sampled MongoDB dev** reviews (`--mongo`, `--mongo-all`).
- Optionally run the **same** `run_pipeline` via `--pipeline` or `--pipeline-only path/to.csv`.

Default paths:

- Taxonomy: `results/sub_topic_taxonomy_cascaded.json`
- Bucket–sentiment mapping: `results/bucket_sentiment_mapping_mongodb.json`

---

## Legacy daily classifier

`scripts/daily_sentiment_classifier.py`:

- Targets **App Store**, **Play Store**, **Trustpilot** only (no Reddit in `COLLECTIONS`).
- Uses **fixed topic enums** and a different MongoDB field layout than the cascaded job (see that file for `classification_source` and topic storage).
- Useful for historical comparison or if downstream still expects the old schema.

---

## Taxonomy: buckets, sub-topics, and compound keys

Source of truth: `results/sub_topic_taxonomy_cascaded.json`. **Buckets** are top-level keys. Each bucket has sub-topic objects with a **`label`** (used by the LLM and `validate_topics`).

After `filter_topics_by_sentiment.py`, each topic stores `sub_topic` as **`{label}-{bucket}`** (subtopic-bucket). That string is what usually flows into MongoDB `bucket_classification` (unless replaced by the condense step’s short display string).

### Step 1 — Sentiment (LLM output)

Exactly one of: **`positive`**, **`negative`**, **`neutral`**.

### Step 2 — Buckets (LLM picks from sentiment-allowed list)

Allowed buckets depend on `results/bucket_sentiment_mapping_mongodb.json` (positive / negative / both). **All bucket keys in the taxonomy:**

- `account_problems`
- `app_crashes_and_bugs`
- `bank_linking_problems`
- `budgeting_features`
- `cancellation_problems`
- `cash_advance_problems`
- `easy_signup_and_setup`
- `easy_to_use`
- `emergency_cash_access`
- `fair_pricing`
- `fast_cash_access`
- `fee_and_billing_complaints`
- `general_dissatisfaction`
- `general_positive_feedback`
- `great_customer_support`
- `identity_verification_issues`
- `misleading_advertising`
- `neutral`
- `other`
- `poor_customer_support`
- `reliable_service`
- `scam_and_fraud_concerns`
- `waitlist_delays`
- `withdrawal_issues`
- `would_recommend`

### Step 3 — Sub-topic `label` per bucket (LLM + validation)

Each line is **`label`** (stored with **`bucket`** in JSON until the sentiment filter emits the compound form).


**`account_problems`**
- `account_login_blocked`
- `signup_problems`
- `account_verification_failing`
- `account_linking_issues`
- `account_cancellation_issues`
- `payment_and_subscription_issues`
- `account_update_issues`
- `account_review_delays`
- `other`

**`app_crashes_and_bugs`**
- `update_loop`
- `app_freezes_during_login`
- `app_crashes`
- `verification_screen_stuck`
- `loading_problems`
- `withdrawal_bugs`
- `general_functionality_failure`
- `confusing_interface`
- `other`

**`bank_linking_problems`**
- `unsupported_banks`
- `repeated_verification_requests`
- `connection_errors`
- `bank_info_rejected_or_wrong`
- `support_unhelpful_bank_linking`
- `other`

**`budgeting_features`**
- `budget_tracking`
- `cash_advance`
- `savings_tools`
- `credit_management`
- `financial_advice`
- `connectivity_issues`
- `other`

**`cancellation_problems`**
- `subscription_cancellation_difficulty`
- `continued_charges_after_cancel`
- `account_deactivation_issues`
- `other`

**`cash_advance_problems`**
- `cash_advance_unavailability`
- `approval_issues`
- `technical_errors_advance_access`
- `support_unhelpful_advance_issues`
- `misleading_information`
- `high_cost`
- `loan_amount_limits`
- `repayment_terms_too_short`
- `income_requirements_too_high`
- `other`

**`easy_signup_and_setup`**
- `easy_sign_up`
- `quick_funds_transfer`
- `smooth_account_linking`
- `other`

**`easy_to_use`**
- `easy_navigation`
- `quick_cash_advance`
- `simple_setup`
- `user_friendly_interface`
- `helps_manage_finances`
- `simple_and_effective`
- `other`

**`emergency_cash_access`**
- `helpful_in_tough_times` *(listed twice in the JSON file; same label)*
- `financial_bridge_until_payday`
- `basic_needs_support`
- `customer_service_experience`
- `trust_and_security`
- `subscription_worth_it`
- `other`

**`fair_pricing`**
- `low_fees`
- `no_due_dates`
- `fair_subscription_cost`
- `convenient_service`
- `other`

**`fast_cash_access`**
- `instant_cash_availability`
- `loan_advance_feature`
- `financial_management_tools`
- `eligibility_and_requirements`
- `comparison_to_other_apps`
- `other`

**`fee_and_billing_complaints`**
- `unexpected_billing_charges`
- `subscription_cancellation_difficulty`
- `double_charging`
- `subscription_fee_confusion`
- `service_not_provided`
- `predatory_practices`
- `other`

**`general_dissatisfaction`**
- `app_useless`
- `generic_negative`
- `overall_bad_experience`
- `other`

**`general_positive_feedback`**
- `general_praise`
- `helpful`
- `overall_satisfaction`
- `there_when_needed`
- `customer_service_praise`
- `easy_and_affordable`
- `other`

**`great_customer_support`**
- `quick_resolution`
- `friendly_staff`
- `technical_issue_resolution`
- `patience_and_professionalism`
- `personalized_support`
- `other`

**`identity_verification_issues`**
- `code_not_received`
- `id_verification_failure`
- `app_crashes_during_verification`
- `verification_process_takes_too_long`
- `correct_info_rejected_verification`
- `selfie_id_mismatch`
- `device_compatibility`
- `other`

**`misleading_advertising`**
- `unavailable_services`
- `misleading_income_requirements`
- `false_credit_check_claims`
- `age_restriction_disclosure`
- `misleading_subscription_fees`
- `false_advertising_of_support`
- `wrong_product`
- `other`

**`neutral`**
- `account_connection_issues`
- `subscription_cost_concern`
- `technical_issues_general`
- `waitlist_or_availability_issues`
- `loan_amount_limits`
- `short_or_unclear`
- `other`

**`other`**
- `app_functionality_issues`
- `difficulty_using_app`
- `data_privacy_or_deletion_concern`
- `positive_feedback`
- `negative_feedback`
- `off_topic_or_wrong_app`
- `other`

**`poor_customer_support`**
- `unresponsive_support`
- `ineffective_support`
- `rude_support`
- `slow_response`
- `billing_issues`
- `account_management_issues`
- `vague_support_complaint`
- `other`

**`reliable_service`**
- `financial_support`
- `ease_of_use`
- `timely_service`
- `credit_building`
- `trustworthiness`
- `overall_reliability`
- `other`

**`scam_and_fraud_concerns`**
- `false_advertising`
- `charges_suggest_fraud_or_theft`
- `data_misuse_or_theft_concern`
- `data_exposure`
- `fake_reviews`
- `technical_issues`
- `scam_or_fraud`
- `other`

**`waitlist_delays`**
- `long_wait_times`
- `lack_of_transparency`
- `waitlist_progression_issues`
- `information_collection_concerns`
- `subscription_paid_then_waitlisted`
- `other`

**`withdrawal_issues`**
- `instant_withdrawals`
- `withdrawal_failures`
- `withdrawal_button_issues`
- `insufficient_funds_error`
- `slow_withdrawals`
- `other`

**`would_recommend`**
- `emergency_cash`
- `ease_of_use`
- `customer_support`
- `no_hidden_fees`
- `recommends_for_budgeting`
- `cash_back_feature`
- `general_positive_experience`
- `other`

**Total:** 25 buckets; **172** sub-topic rows in JSON (**171** unique `(bucket, label)` pairs; one duplicate `helpful_in_tough_times` under `emergency_cash_access`).

### End state — Compound keys `subtopic-bucket` (pipeline / MongoDB)

Unique strings of the form **`{label}-{bucket}`** (subtopic first, bucket second; same as `_to_subtopic_bucket` in `filter_topics_by_sentiment.py`):


- `account_cancellation_issues-account_problems`
- `account_connection_issues-neutral`
- `account_deactivation_issues-cancellation_problems`
- `account_linking_issues-account_problems`
- `account_login_blocked-account_problems`
- `account_management_issues-poor_customer_support`
- `account_review_delays-account_problems`
- `account_update_issues-account_problems`
- `account_verification_failing-account_problems`
- `age_restriction_disclosure-misleading_advertising`
- `app_crashes-app_crashes_and_bugs`
- `app_crashes_during_verification-identity_verification_issues`
- `app_freezes_during_login-app_crashes_and_bugs`
- `app_functionality_issues-other`
- `app_useless-general_dissatisfaction`
- `approval_issues-cash_advance_problems`
- `bank_info_rejected_or_wrong-bank_linking_problems`
- `basic_needs_support-emergency_cash_access`
- `billing_issues-poor_customer_support`
- `budget_tracking-budgeting_features`
- `cash_advance-budgeting_features`
- `cash_advance_unavailability-cash_advance_problems`
- `cash_back_feature-would_recommend`
- `charges_suggest_fraud_or_theft-scam_and_fraud_concerns`
- `code_not_received-identity_verification_issues`
- `comparison_to_other_apps-fast_cash_access`
- `confusing_interface-app_crashes_and_bugs`
- `connection_errors-bank_linking_problems`
- `connectivity_issues-budgeting_features`
- `continued_charges_after_cancel-cancellation_problems`
- `convenient_service-fair_pricing`
- `correct_info_rejected_verification-identity_verification_issues`
- `credit_building-reliable_service`
- `credit_management-budgeting_features`
- `customer_service_experience-emergency_cash_access`
- `customer_service_praise-general_positive_feedback`
- `customer_support-would_recommend`
- `data_exposure-scam_and_fraud_concerns`
- `data_misuse_or_theft_concern-scam_and_fraud_concerns`
- `data_privacy_or_deletion_concern-other`
- `device_compatibility-identity_verification_issues`
- `difficulty_using_app-other`
- `double_charging-fee_and_billing_complaints`
- `ease_of_use-reliable_service`
- `ease_of_use-would_recommend`
- `easy_and_affordable-general_positive_feedback`
- `easy_navigation-easy_to_use`
- `easy_sign_up-easy_signup_and_setup`
- `eligibility_and_requirements-fast_cash_access`
- `emergency_cash-would_recommend`
- `fair_subscription_cost-fair_pricing`
- `fake_reviews-scam_and_fraud_concerns`
- `false_advertising-scam_and_fraud_concerns`
- `false_advertising_of_support-misleading_advertising`
- `false_credit_check_claims-misleading_advertising`
- `financial_advice-budgeting_features`
- `financial_bridge_until_payday-emergency_cash_access`
- `financial_management_tools-fast_cash_access`
- `financial_support-reliable_service`
- `friendly_staff-great_customer_support`
- `general_functionality_failure-app_crashes_and_bugs`
- `general_positive_experience-would_recommend`
- `general_praise-general_positive_feedback`
- `generic_negative-general_dissatisfaction`
- `helpful-general_positive_feedback`
- `helpful_in_tough_times-emergency_cash_access`
- `helps_manage_finances-easy_to_use`
- `high_cost-cash_advance_problems`
- `id_verification_failure-identity_verification_issues`
- `income_requirements_too_high-cash_advance_problems`
- `ineffective_support-poor_customer_support`
- `information_collection_concerns-waitlist_delays`
- `instant_cash_availability-fast_cash_access`
- `instant_withdrawals-withdrawal_issues`
- `insufficient_funds_error-withdrawal_issues`
- `lack_of_transparency-waitlist_delays`
- `loading_problems-app_crashes_and_bugs`
- `loan_advance_feature-fast_cash_access`
- `loan_amount_limits-cash_advance_problems`
- `loan_amount_limits-neutral`
- `long_wait_times-waitlist_delays`
- `low_fees-fair_pricing`
- `misleading_income_requirements-misleading_advertising`
- `misleading_information-cash_advance_problems`
- `misleading_subscription_fees-misleading_advertising`
- `negative_feedback-other`
- `no_due_dates-fair_pricing`
- `no_hidden_fees-would_recommend`
- `off_topic_or_wrong_app-other`
- `other-account_problems`
- `other-app_crashes_and_bugs`
- `other-bank_linking_problems`
- `other-budgeting_features`
- `other-cancellation_problems`
- `other-cash_advance_problems`
- `other-easy_signup_and_setup`
- `other-easy_to_use`
- `other-emergency_cash_access`
- `other-fair_pricing`
- `other-fast_cash_access`
- `other-fee_and_billing_complaints`
- `other-general_dissatisfaction`
- `other-general_positive_feedback`
- `other-great_customer_support`
- `other-identity_verification_issues`
- `other-misleading_advertising`
- `other-neutral`
- `other-other`
- `other-poor_customer_support`
- `other-reliable_service`
- `other-scam_and_fraud_concerns`
- `other-waitlist_delays`
- `other-withdrawal_issues`
- `other-would_recommend`
- `overall_bad_experience-general_dissatisfaction`
- `overall_reliability-reliable_service`
- `overall_satisfaction-general_positive_feedback`
- `patience_and_professionalism-great_customer_support`
- `payment_and_subscription_issues-account_problems`
- `personalized_support-great_customer_support`
- `positive_feedback-other`
- `predatory_practices-fee_and_billing_complaints`
- `quick_cash_advance-easy_to_use`
- `quick_funds_transfer-easy_signup_and_setup`
- `quick_resolution-great_customer_support`
- `recommends_for_budgeting-would_recommend`
- `repayment_terms_too_short-cash_advance_problems`
- `repeated_verification_requests-bank_linking_problems`
- `rude_support-poor_customer_support`
- `savings_tools-budgeting_features`
- `scam_or_fraud-scam_and_fraud_concerns`
- `selfie_id_mismatch-identity_verification_issues`
- `service_not_provided-fee_and_billing_complaints`
- `short_or_unclear-neutral`
- `signup_problems-account_problems`
- `simple_and_effective-easy_to_use`
- `simple_setup-easy_to_use`
- `slow_response-poor_customer_support`
- `slow_withdrawals-withdrawal_issues`
- `smooth_account_linking-easy_signup_and_setup`
- `subscription_cancellation_difficulty-cancellation_problems`
- `subscription_cancellation_difficulty-fee_and_billing_complaints`
- `subscription_cost_concern-neutral`
- `subscription_fee_confusion-fee_and_billing_complaints`
- `subscription_paid_then_waitlisted-waitlist_delays`
- `subscription_worth_it-emergency_cash_access`
- `support_unhelpful_advance_issues-cash_advance_problems`
- `support_unhelpful_bank_linking-bank_linking_problems`
- `technical_errors_advance_access-cash_advance_problems`
- `technical_issue_resolution-great_customer_support`
- `technical_issues-scam_and_fraud_concerns`
- `technical_issues_general-neutral`
- `there_when_needed-general_positive_feedback`
- `timely_service-reliable_service`
- `trust_and_security-emergency_cash_access`
- `trustworthiness-reliable_service`
- `unavailable_services-misleading_advertising`
- `unexpected_billing_charges-fee_and_billing_complaints`
- `unresponsive_support-poor_customer_support`
- `unsupported_banks-bank_linking_problems`
- `update_loop-app_crashes_and_bugs`
- `user_friendly_interface-easy_to_use`
- `vague_support_complaint-poor_customer_support`
- `verification_process_takes_too_long-identity_verification_issues`
- `verification_screen_stuck-app_crashes_and_bugs`
- `waitlist_or_availability_issues-neutral`
- `waitlist_progression_issues-waitlist_delays`
- `withdrawal_bugs-app_crashes_and_bugs`
- `withdrawal_button_issues-withdrawal_issues`
- `withdrawal_failures-withdrawal_issues`
- `wrong_product-misleading_advertising`

**Count:** 171 unique compound keys.

### After `condense_subtopic_labels_in_csv.py`

`bucket_classification` may instead hold a **short display phrase** (see `SUBTOPIC_BUCKET_TO_SHORT` in `scripts/condense_subtopic_labels_in_csv.py`) when the compound key is in that map; otherwise a **Title Case** fallback is derived from the subtopic part of the key.


---

## Related scripts and artifacts (by purpose)

| Purpose | Location |
|---------|----------|
| OpenAI key: env or SSM | `scripts/openai_api_key.py` |
| Cron wrapper (pyenv, `source .env`, `--push`) | `scripts/run_daily_cascaded_cron.sh` |
| Core cascaded LLM + `run_pipeline` | `scripts/cascaded_classifier.py` |
| Prod daily cascaded job | `scripts/daily_sentiment_classifier_cascaded.py` |
| Legacy prod daily job | `scripts/daily_sentiment_classifier.py` |
| Pipeline steps | `reclassify_subtopic_other.py`, `filter_topics_by_sentiment.py`, `reduce_to_single_topic_per_review.py`, `condense_subtopic_labels_in_csv.py` |
| Taxonomy / rules (data) | `results/sub_topic_taxonomy_cascaded.json`, `results/bucket_sentiment_mapping_mongodb.json`, overrides/exclusivity JSONs under `results/` |
| Pilot / eval | `scripts/baseline_classifier_pilot.py`, `scripts/optimized_classifier_pilot.py`, `scripts/evaluate_label_match.py` |
| **Non-LLM** sentiment (Hugging Face BERTweet) — separate exploratory pipeline | `scripts/sentiment_classification.py`, `mongo_sentiment_pipeline.py`, `scripts/cluster_by_sentiment.py` |
| Reporting / validation samples | `SENTIMENT_ANALYSIS_REPORT.md`, `sentiment_accuracy_validation.py`, various `results/*sentiment*.csv` |
| Taxonomy / label discovery (historical; see below) | `scripts/sub_topic_discovery.py`, `scripts/taxonomy_discovery_fresh.py`, `scripts/generate_descriptive_labels.py` |
| Clustering exploration (non-production path) | `scripts/sentiment_classification.py` → `scripts/cluster_by_sentiment.py` (UMAP + HDBSCAN) |

---

## How taxonomy and labels were developed (historical)

The production taxonomy file **`results/sub_topic_taxonomy_cascaded.json`** and the **bucket / sub-topic** labels did not come from a single automated step. They reflect an **iterative process** that mixed **embedding-based clustering**, **LLM discovery**, and **manual curation** to get stable labels suitable for MongoDB and reporting.

1. **Sentiment + clustering (exploratory)**  
   - `scripts/sentiment_classification.py` runs a **Hugging Face** sentiment model (BERTweet-family) over reviews and saves grouped results (e.g. `results/sentiment_results.pkl`).  
   - `scripts/cluster_by_sentiment.py` then applies **UMAP** dimensionality reduction and **HDBSCAN** **separately** within sentiment groups (e.g. positive vs negative) to surface **natural clusters** in embedding space—useful for understanding themes before naming them.

2. **LLM-based taxonomy discovery**  
   - `scripts/sub_topic_discovery.py` **samples** reviews per bucket and uses an LLM to **propose sub-topics** (discover / classify modes).  
   - `scripts/taxonomy_discovery_fresh.py` pulls **fresh samples** from MongoDB and uses an LLM to discover **buckets and sub-topics** without depending on prior classification (`discover-buckets`, `discover-subtopics`, `full`).

3. **Consolidation into the cascaded taxonomy**  
   The **`sub_topic_taxonomy_cascaded.json`** structure (buckets → objects with `label`, `description`, `examples`, `display_label`) was built by **merging and refining** those signals so labels align with **product language**, **coverage**, and **sentiment rules** (`bucket_sentiment_mapping_mongodb.json`, overrides, exclusivity JSON).

4. **Display labels**  
   - `scripts/generate_descriptive_labels.py` builds **human-readable** display strings from the taxonomy.  
   - At inference time, `scripts/condense_subtopic_labels_in_csv.py` maps **`subtopic-bucket`** keys to **short phrases** for dashboards and MongoDB-facing fields where applicable.

5. **Iteration**  
   Pilots (`baseline_classifier_pilot.py`, `optimized_classifier_pilot.py`), reclassification passes (`reclassify_subtopic_other.py`, etc.), and evaluation scripts refined prompts and taxonomy entries over time.

**Takeaway:** New labels should be added by **updating the JSON + mappings** and re-validating—not by expecting clustering alone to match production taxonomy without review.

---

## Operational notes

1. **Cost & rate**: Each review triggers **multiple** OpenAI calls (3+ in cascaded; more if reclassify step hits “general/other” rows). There is a small `time.sleep(0.05)` between reviews in the daily cascaded script to reduce burstiness.
2. **Secrets**: Prefer **SSM** for the OpenAI key in production (see [OpenAI API key (environment vs AWS SSM Parameter Store)](#openai-api-key-environment-vs-aws-ssm-parameter-store)); use **`OPENAI_API_KEY`** only as a deliberate override. Set **`MONGODB_URI_PROD` / `MONGODB_URI_DEV`** in the environment; avoid relying on hardcoded URI fallbacks in shared code long term.
3. **Single source of truth for topics**: Editing classification behavior usually means updating **taxonomy JSON**, **bucket–sentiment mapping**, and/or prompts in `cascaded_classifier.py`, then re-running pilots before changing production cron args.

---

*Document generated for KT; aligns with scripts under `scripts/` as of the repo snapshot used to author it.*
