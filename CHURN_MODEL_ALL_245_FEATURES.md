# Churn Model — All 245 Features: Definitions & Churn Relevance

**Document purpose:** This document provides a comprehensive reference for every feature used in the Beem subscription churn prediction model. It is intended for data science, product, and engineering teams to understand *what* each feature measures, *why* it is predictive of churn, and *how* it supports retention interventions.

**Dataset:** `150_complete_churn_feature_data_with_mixpanel.parquet`  
**Total features:** 245 (157 base/enriched + 88 Mixpanel)  
**Excluded from model:** userId, month, createdAt, type, startDate, status, endDate, month_snapshot, is_churned

---

## Executive Summary

The churn model uses **245 numeric features** drawn from five data sources: **MongoDB** (subscriptions, transactions, logins, behavioural scores), **MoEngage** (email and push engagement), **AppsFlyer** (attribution and in-app events), **Mixpanel** (app engagement, churn-intent flows, friction events), and **derived composites** that combine signals across sources.

**Why feature documentation matters:** Each feature was selected because it captures a dimension of user behavior, financial health, or engagement that research and model validation have shown to predict churn. Understanding these features enables (1) **model interpretability** — explaining why a user was flagged as at-risk; (2) **retention strategy** — knowing which levers to pull (e.g., re-engagement campaigns for users with low email opens); and (3) **data quality** — identifying gaps or drift in upstream data sources.

**Model performance context:** The model with Mixpanel features achieves ~96% accuracy on test and backtest, with improved precision (fewer false alarms) and recall (fewer missed churners) compared to the baseline without Mixpanel. Feature importance analysis consistently ranks engagement recency, loan activity, and communication engagement among the top predictors.

---

## Table of Contents

- [1. Base 11 (MongoDB — Core Product & Risk)](#1-base-11-mongodb--core-product--risk)
- [2. Subscription & Billing](#2-subscription--billing-mongodb--linesubscriptionchangelogs-linelatefees-latefeewaiverrequests)
- [3. Logins & Sessions](#3-logins--sessions-mongodb--logins-userloginsessions)
- [4. Payment & Funds](#4-payment--funds-mongodb--linefunds-stripepaymentmethods)
- [5. Recurring Transactions](#5-recurring-transactions-mongodb--linerecurringtransactions)
- [6. MoEngage (Email, Push, Campaigns)](#6-moengage-email-push-campaigns)
- [7. Behavioural Score Metrics](#7-behavioural-score-metrics-mongodb--behaviouralscoremetrics)
- [8. Qualification Metrics](#8-qualification-metrics-mongodb--linequalificationmetrics)
- [9. Everdraft & Loans](#9-everdraft--loans-mongodb--everdraftitems-lineloanitems)
- [10. Onboarding & User Profile](#10-onboarding--user-profile-mongodb--onboardingtrackers-lineaccounts-lineusers)
- [11. AppsFlyer (Attribution & Engagement)](#11-appsflyer-attribution--engagement)
- [12. Derived / Composite Features](#12-derived--composite-features)
- [13. Mixpanel — Engagement](#13-mixpanel--engagement-30d--90d-counts)
- [14. Mixpanel — Churn Intent & Flows](#14-mixpanel--churn-intent--flows-90d)
- [15. Mixpanel — Friction & Errors](#15-mixpanel--friction--errors-90d)
- [16. Mixpanel — Recency](#16-mixpanel--recency-days-since)
- [17. Mixpanel — Derived](#17-mixpanel--derived)
- [Summary by Group](#summary-by-group)

---

## 1. Base 11 (MongoDB — Core Product & Risk)

These 11 features form the original model foundation. They capture core product usage, payment health, and risk flags — the most direct signals of whether a user is receiving value and can afford to stay.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `loans_count_in_month` | Number of loans/advances taken in the snapshot month (lineloanitems) | **Strongest product-usage signal.** Everdraft/cash advance is the primary reason users subscribe. Churners typically have 0 or near-zero loan activity (94%+ of churners have 0); active users average ~0.8+ loans/month. This feature consistently ranks in the top 5 by SHAP importance. Users who stop taking advances have effectively decided the product is no longer valuable. |
| `loan_paid_45` | Proportion of loans repaid within 45 days | **Repayment discipline.** Users who repay on time are financially stable and engaged. Churners rarely repay (98%+ have 0); non-churners average ~0.29. Low repayment indicates either financial distress (can't afford to stay) or disengagement (don't care to repay). Both paths lead to churn. |
| `nsf_count` | Count of NSF (non-sufficient funds) events in lookback (linefunds) | **Payment friction and financial stress.** Repeated NSF events indicate the user is struggling to maintain sufficient balance. This correlates with involuntary churn (subscription payment fails) and voluntary churn (frustration with the product). Model shows churners have higher NSF counts on average. |
| `nsf_check` | 1 if NSF was resolved, 0 otherwise | **Recovery signal.** Resolved NSF means the user fixed the issue — a positive sign. Unresolved NSF suggests ongoing payment problems that may culminate in churn. Helps distinguish "had a blip but recovered" from "chronically unable to pay." |
| `card_status` | Card verification status (stripepaymentmethods) | **Payment method health.** Unverified or problematic cards cause subscription payment failures at renewal. Users with bad card status face involuntary churn the moment the next billing cycle hits. Proactive card-update prompts can prevent this. |
| `card_exp` | 1 if card is expired | **Direct payment failure risk.** Expired cards cannot be charged. This is a leading indicator of imminent involuntary churn. Retention teams can target these users for card-update campaigns before the next billing attempt. |
| `income_check` | Income verification flag (linequalificationmetrics/lineusers) | **Financial stability and qualification.** Verified income indicates the user has completed onboarding and qualifies for advances. Unverified may indicate stuck onboarding, qualification issues, or disengagement — all associated with higher churn. |
| `negative_velocity` | 1 if user hit negative velocity limit (velocitychecks) | **Transfer/usage restriction.** Users who hit velocity limits have been flagged for risk (e.g., suspicious activity). This restriction can frustrate users and block core use cases, driving both voluntary and involuntary churn. |
| `volatility_label` | Income volatility indicator | **Financial stability.** High income volatility means unpredictable cash flow. These users may churn when they cannot afford the subscription during a low-income month, or when they feel the product no longer fits their situation. |
| `ticket_raised` | 1 if user has support ticket (jira/usertickets) | **Friction indicator.** Support tickets often reflect unresolved issues — billing problems, feature confusion, or technical bugs. Users who file tickets and do not receive satisfactory resolution are at elevated churn risk. Proactive outreach to ticket-filers can improve retention. |
| `has_disliked` | 1 if user gave negative app feedback | **Direct dissatisfaction.** This is an explicit signal of poor experience. Users who leave negative feedback are already expressing frustration; without intervention, many will churn. High-priority segment for retention outreach. |

---

## 2. Subscription & Billing (MongoDB — linesubscriptionchangelogs, linelatefees, latefeewaiverrequests)

Subscription lifecycle features capture commitment, prior churn history, and payment health. These are among the most interpretable predictors: a user who has churned before, downgraded recently, or accumulated late fees is at elevated risk.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `renewal_count` | Number of renewals in 12-month lookback | **Commitment signal.** Each renewal is a vote of confidence. Users with 6+ renewals have demonstrated willingness to stay; churners typically have 0–2. Renewal count is a strong negative correlate of churn — the more renewals, the lower the risk. |
| `prior_churn_count` | Times user churned before (resubscribers) | **Re-churn risk.** Prior churners have already left once; they know the exit path and are more likely to churn again. Resubscribers often have different expectations and may churn faster if those expectations are not met. |
| `subscription_tenure_months` | Months since first subscription | **Tenure effect.** Early-lifecycle users (0–3 months) churn at higher rates than mature users (12+ months). Tenure moderates risk: the same behavior (e.g., low logins) is more concerning for a 2-month user than a 24-month user. |
| `downgrade_count` | Downgrades in 12 months | **Intent signal.** Downgrades often precede full churn — the user reduces spend before exiting. Even one downgrade in the past year is a meaningful risk indicator. |
| `upgrade_count` | Upgrades in 12 months | **Commitment signal.** Upgraders show intent to invest more in the product. They rarely churn in the short term; upgrades are a protective factor. |
| `late_fee_waiver_request_count` | Waiver requests in 90d | **Financial stress.** Frequent waiver requests indicate the user is struggling with cash flow. They are asking for relief — a sign of strain that may lead to churn when they can no longer afford the subscription. |
| `has_waiver_request` | 1 if any waiver request | **Binary waiver flag.** Flags any waiver request in the lookback window. Simpler than count for model use; users with has_waiver_request=1 are at elevated financial stress risk regardless of how many requests they made. |
| `late_fee_last_90d` | Late fee count in 90d | **Payment delinquency.** Late fees mean the user failed to pay on time. Each late fee is a near-miss; repeated late fees indicate financial stress and predict involuntary churn when the next payment fails. |
| `late_fee_total_amount_90d` | Sum of late fee amounts in 90d | **Severity of delinquency.** Total amount captures both frequency and magnitude. High amounts may also indicate frustration with fees, driving voluntary churn. |
| `waiver_approved_90d` | 1 if waiver approved in last 90d | **Recovery signal.** An approved waiver may reduce frustration and keep the user. However, users who need waivers are still at elevated risk; this feature helps distinguish "got relief and stayed" from "got relief but churned anyway." |

---

## 3. Logins & Sessions (MongoDB — logins, userloginsessions)

Login and session features measure *engagement recency* and *access friction*. Users who stop logging in have effectively disengaged; those who struggle to log in may churn out of frustration. These features consistently rank among the top predictors in SHAP analysis.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `days_since_last_login` | Days since last login | **One of the strongest churn predictors.** Long gaps (e.g., 30+ days) indicate disengagement. Churners average 45+ days since last login; non-churners average ~7. This feature captures "going dark" — the user has stopped using the app before they cancel. |
| `logins_last_30d` | Login count in last 30 days | **Engagement frequency.** Zero logins in 30 days is a red flag; 5+ suggests active use. Churners typically have 0–1; non-churners average 4–6. Complements days_since for both level and recency. |
| `logins_prior_30d` | Login count in prior 30–60d window | **Trend baseline.** Used with logins_last_30d to compute login trend. Declining logins (current < prior) is a leading indicator — the user is pulling back before they churn. |
| `logins_last_7d` | Login count in last 7 days | **Very recent signal.** Captures immediate engagement. Users active in the last 7 days are lower risk; zero in 7d with prior activity suggests sudden drop-off. |
| `login_failure_ratio` | 1 − (successful logins / total attempts) | **Access friction.** High ratio means the user is trying to log in but failing — password issues, 2FA problems, or technical bugs. Unresolved login friction drives churn; these users may give up and cancel. |
| `is_ios` | 1 if majority of sessions are iOS | **Platform segmentation.** iOS vs Android users may have different churn profiles (e.g., different payment behaviors, app store dynamics). Helps the model calibrate risk by platform. |
| `device_switch_count` | (Distinct devices) − 1 | **Device churn.** Frequent device changes may indicate account sharing (risk) or user instability (new phone, lost device). Very high counts can signal fraud or multi-user accounts. |
| `userloginsessions_device_count` | Distinct device count | **Device diversity.** Number of distinct devices used. High count may indicate account sharing or user instability; single device = more typical. Complements device_switch_count (which is count − 1). |
| `userloginsessions_count_7d` | Sessions in last 7 days | **Session frequency.** Sessions can exceed logins (multiple sessions per login). High session count = heavy engagement; zero = disengaged. |

---

## 4. Payment & Funds (MongoDB — linefunds, stripepaymentmethods)

Payment and funds features capture the *ability to pay* — the leading cause of involuntary churn. Users with expired cards, no payment method, or repeated Stripe failures will churn when the next billing attempt fails.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `card_expired_count` | Count of card-expiry failures | **Payment friction.** Each card-expiry failure is a missed payment. Users with multiple failures are on the path to involuntary churn; proactive card-update prompts can prevent this. |
| `no_payment_method_count` | Count of no-payment-method failures | **Critical.** No payment method means the user cannot be charged. This is a leading indicator of imminent involuntary churn — the next billing attempt will fail. High-priority segment for payment-method setup campaigns. |
| `has_velocity_failure_limit` | 1 if velocity limitType contains "failure" | **Usage restriction.** Users who hit velocity limits have been flagged for risk. The restriction can block core features (transfers, withdrawals), frustrating users and driving both voluntary and involuntary churn. |
| `stripe_failed_charge_count` | Failed Stripe charges in 90d | **Payment failure.** Each failed charge is a near-miss. Repeated failures indicate the user's payment method is unreliable; they will eventually churn when a critical charge (e.g., subscription) fails. |
| `stripe_payment_failure_rate` | failed / (failed + success) | **Payment health.** High rate (e.g., >50%) means most payment attempts fail. The subscription is at risk; dunning and retry campaigns may be insufficient. |
| `days_since_last_sub_payment` | Days since last subscription payment | **Payment recency.** Stale (e.g., 60+ days) suggests the user has not paid recently — they may have churned, paused, or be in grace/dunning. Complements subscription status for identifying at-risk users. |

---

## 5. Recurring Transactions (MongoDB — linerecurringtransactions)

Recurring transaction features measure *financial planning engagement* — users who track recurring expenses (rent, utilities, subscriptions) are more embedded in the product and use it for budgeting. Lower engagement with this feature may indicate superficial use.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `recurring_txn_count` | Recurring transactions in 90d | **Financial planning engagement.** Users who track recurring expenses are using the product for budgeting and cash flow management. Higher counts indicate deeper integration; zero may indicate the user only uses the app for advances and is less embedded. |
| `fixed_recurring_count` | Count where fixedRecurrFlag=true | **Fixed vs variable.** Fixed recurring (e.g., rent) vs variable (e.g., utilities) may capture different engagement depths. Complements recurring_txn_count for a fuller picture of financial planning usage. |

---

## 6. MoEngage (Email, Push, Campaigns)

MoEngage features capture *communication engagement* — whether users open emails, receive push, and respond to retention campaigns. Users who disengage from communications are often on the path to churn; those who engage with retention outreach are more likely to stay.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `email_open_count_30d` | Email opens in last 30d | **Communication engagement.** No opens = the user is not reading our emails. Churners average 0–1 opens; non-churners average 3–5. Email opens are a leading indicator — users who stop opening emails often churn within 1–2 months. |
| `email_bounce_count_30d` | Email soft bounces in 30d | **Delivery issues.** Bounces indicate invalid email, full inbox, or deliverability problems. Users we cannot reach cannot be retained via email; high bounces predict churn. |
| `email_click_count_30d` | Email clicks in 30d | **Deeper engagement.** Clicks require more intent than opens. Users who click are actively engaging; zero clicks with opens may indicate passive engagement. Clicks often have stronger predictive value than opens alone. |
| `push_received_count_30d` | Push notifications received in 30d | **Push engagement.** Push is a key retention channel. Users who receive and (implicitly) do not opt out are reachable; declining push received may indicate delivery issues or opt-out. |
| `days_since_last_email_open` | Days since last email open | **Recency.** Long gap (e.g., 30+ days) = disengaged. Complements open count — a user with 5 opens 60 days ago is different from 5 opens in the last 7 days. |
| `push_opt_out_flag` | 1 if user opted out of push | **Strong churn signal.** Push opt-out is a declared disengagement — the user explicitly chose to stop receiving communications. Often precedes or accompanies churn. |
| `email_open_count_prior_30d` | Email opens in prior 30–60d | **Trend baseline.** Used to compute email_open_trend_30d. |
| `email_open_trend_30d` | Current opens − prior opens | **Engagement trend.** Declining opens (negative trend) = the user is pulling back. One of the leading churn indicators; often precedes cancellation by 1–2 months. |
| `push_received_count_prior_30d` | Push received in prior 30–60d | **Trend baseline for push.** Used to compute push_received_trend_30d. Compare current vs prior to detect whether push engagement is improving or declining. |
| `push_received_trend_30d` | Push trend (current − prior) | **Push engagement trend.** Negative trend = user is receiving or engaging with fewer push notifications. Declining push engagement often precedes churn; complements email_open_trend for a full picture of communication pullback. |
| `days_since_push_opt_out` | Days since last opt-out event | **Recency of disengagement.** Recent opt-out = fresh disengagement; older opt-out may have been followed by re-opt-in. |
| `email_click_open_ratio_30d` | Clicks / max(opens, 1) | **Engagement depth.** High ratio = user opens and acts; low ratio = opens but does not click. Deeper engagement correlates with lower churn. |
| `email_open_transactional_30d` | Opens where content type = Transactional | **Transactional vs promotional.** Transactional (e.g., receipts, alerts) may indicate the user is still using the product; promotional opens may indicate interest in offers. Mix of both is healthy. |
| `email_open_promotional_30d` | Opens where content type = Promotional | **Promotional content engagement.** Users who open promotional emails (offers, tips, upsells) show interest beyond transactional needs. Zero promotional opens with transactional opens may indicate narrow engagement; both types contribute to stickiness. |
| `no_comms_engagement_30d` | 1 if no email opens and no push received | **Complete comms disengagement.** No email opens and no push = user is unreachable via our primary channels. Strong churn signal; these users are "going dark" across all touchpoints. |
| `days_since_last_push_received` | Days since last push | **Push recency.** Long gap = user has not received (or engaged with) push in a while. May indicate delivery issues, opt-out, or app not opened. Complements days_since_last_email_open for cross-channel recency. |
| `days_since_last_email_click` | Days since last email click | **Recency of deep engagement.** Clicks are stronger than opens; long gap since last click = shallow or no engagement. |
| `received_retention_campaign_30d` | 1 if retention/winback/dues campaign | **Retention outreach.** Flags users who received a retention campaign. Helps measure campaign reach; combined with engagement, indicates whether campaigns are working. |
| `push_re_opt_in_30d` | 1 if opted out then opted back in | **Re-engagement. Positive signal.** User opted back in after opting out — a sign of renewed interest. Protective factor. |
| `email_click_after_gap_30d` | 1 if click >30d after last open | **Re-engagement after gap.** User had a long gap then clicked — possible re-engagement. Can indicate winback success. |
| `email_open_transactional_share_30d` | Share of opens that are transactional | **Engagement mix.** High transactional share = user engages with product-related emails; low = may only open promotional. |
| `moe_retention_campaign_30d_count` | Count of retention campaign events | **Retention touchpoints.** More touchpoints = more opportunities to re-engage. Combined with engagement, indicates campaign effectiveness. |
| `moe_days_since_last_retention_event` | Days since last retention event | **Recency of retention touch.** Stale = user has not been reached recently; may need another retention push. |
| `moe_retention_engaged_30d` | 1 if retention + (email open or click) | **Engaged with retention campaign.** User received retention outreach and responded. Strong positive signal — retention campaigns that get engagement are associated with lower churn. |
| `moe_days_since_any_open` | min(days_since_email_open, days_since_email_click) | **Best recency across channels.** Uses the most recent engagement (open or click) to avoid penalizing users who prefer one channel. |
| `email_click_to_open_rate_30d` | Clicks / max(opens, 1) | **Engagement depth (click-to-open ratio).** Measures how often users act (click) when they open. High rate = engaged readers who take action; low rate = passive opens. Duplicate of email_click_open_ratio_30d; both capture the same signal. |
| `email_open_promotional_share_30d` | Promotional opens / total opens | **Content preference.** Share of opens that are promotional vs transactional. Users who only open transactional may be narrowly engaged; balanced mix suggests broader interest. |
| `email_open_ratio_vs_prior` | Current opens / (prior + 1) | **Engagement trend.** Ratio > 1 = improving; < 1 = declining. Declining ratio is a leading churn indicator. |

---

## 7. Behavioural Score Metrics (MongoDB — behaviouralscoremetrics)

Behavioural Score Metrics (BSM) capture *financial health* from bank account data — income, cash flow, overdraft, and NSF propensity. Users in financial distress are more likely to churn (voluntary: "can't afford it"; involuntary: payment fails). These features help the model distinguish "engaged but struggling" from "disengaged."

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `qualification` | 1 if qualified for advances | **Product access.** Unqualified users cannot use the core product (Everdraft). They may churn because they get no value, or because qualification issues reflect broader financial instability. |
| `qualification_amount` | Qualified advance amount | **Capacity.** Higher qualified amount = more product value. Users with $0 or low qualification may churn when they feel the product does not meet their needs. |
| `bsm_income30d` | Income in last 30d | **Financial health.** Low or zero income = user may not afford the subscription. Income volatility (from trend) also predicts churn. |
| `bsm_ncf90d` | Net cash flow 90d | **Cash flow health.** Negative NCF = user is spending more than they earn. Strong predictor of involuntary churn — they cannot pay when funds are low. |
| `bsm_debit30d` | Debit volume 30d | **Spending/outflow.** High debit = active account; low = possible disengagement or account dormancy. |
| `bsm_debit30d_count` | Debit transaction count 30d | **Activity level.** Transaction count complements volume; zero transactions = inactive account, higher churn risk. |
| `bsm_overdraft90davg` | Average overdraft 90d | **Financial stress.** High overdraft = user is frequently overdrawing. Indicates cash flow problems; correlates with NSF and payment failures. |
| `bsm_borrowed30d` | Borrowed amount 30d | **Reliance on credit.** High borrowed amount = user depends on credit/advances. May indicate financial strain; also indicates product usage (if from Beem). |
| `bsm_insufficient_funds_ratio` | NSF ratio (60–90d) | **Payment failure propensity.** High ratio = user frequently hits NSF. Direct predictor of involuntary churn; also indicates frustration with fees. |
| `bsm_days_since_outflow` | Days since last outflow transaction | **Activity recency.** Long gap = inactive account. User may have moved financial activity elsewhere — a leading churn indicator. |
| `bsm_max_balance90d` | Max balance in 90d | **Financial capacity.** Higher max balance = more cushion; low = living paycheck-to-paycheck, higher risk of payment failure. |
| `bsm_service30d` | Service spend 30d | **Subscription/usage.** Captures spend on subscription-like services. Complements qualification and product usage. |

---

## 8. Qualification Metrics (MongoDB — linequalificationmetrics)

Qualification Metrics (LQM) provide a *lending-risk view* — scores, liquidity, income gaps, and fee history. These complement BSM with qualification-specific signals. Users with poor LQM scores may face product restrictions or financial stress that drives churn.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `qualification_score` | Qualification score | **Lending risk / capacity.** Low score = user may be restricted from advances or face lower limits. Restriction can frustrate users; low capacity may also indicate financial stress. |
| `qualified_loan_amount` | Qualified loan amount | **Product value.** Zero or low amount = user gets little value from the core product. May churn when they feel the product does not meet their needs. |
| `lqm_nsf_fee_count` | NSF fee count | **Payment friction.** Each NSF fee is a negative experience. High count = repeated payment problems; predicts involuntary churn and user frustration. |
| `lqm_liquidity` | Liquidity metric | **Financial health.** Low liquidity = user has little cushion. More likely to hit NSF or fail subscription payment during a low-cash month. |
| `lqm_monthly_income` | Monthly income | **Capacity.** Income drives ability to pay. Low or declining income predicts churn. |
| `lqm_income_gap_months` | Income gap months | **Financial stress.** Months with no or low income. High gap count = unstable income; user may churn when they cannot afford the subscription. |
| `lqm_account_history_months` | Account history length | **Stability.** Longer history = more data for qualification; may also indicate user tenure with their bank. Shorter history = newer user, potentially higher risk. |
| `lqm_monthly_coverage` | Monthly coverage | **Income vs expenses.** Coverage < 1 = expenses exceed income. User is under financial pressure; higher churn risk. |
| `lqm_overdraft_fee_count` | Overdraft fee count | **Financial stress.** Overdraft fees indicate the user is overdrawing. Correlates with NSF and payment failures; repeated fees can also drive frustration and voluntary churn. |

---

## 9. Everdraft & Loans (MongoDB — everdraftitems, lineloanitems)

Everdraft and loan features measure *core product usage* and *repayment health*. Everdraft/cash advance is the primary reason users subscribe; users who stop using it or fail to repay are at high churn risk. These features consistently rank among the top predictors.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `days_since_last_everdraft` | Days since last everdraft use | **Core product recency.** Long gap (e.g., 60+ days) = user has stopped using the flagship feature. Churners typically have 45+ days; non-churners average ~14. One of the strongest product-usage signals. |
| `everdraft_usage_count_30d` | Everdraft count in 30d | **Core product usage.** Zero in 30d = no Everdraft use. Complements loans_count_in_month; captures Everdraft-specific events. |
| `everdraft_usage_count_90d` | Everdraft count in 90d | **Core product usage (90d window).** Longer window captures users with sporadic usage (e.g., monthly advances). Zero in 90d = no Everdraft use for a full quarter; strong disengagement signal. Complements 30d count for users who use the product less frequently. |
| `everdraft_paid_ratio` | Share of advances paid/closed | **Repayment health.** Low ratio = user is not repaying advances. Indicates either financial distress (can't pay) or disengagement (won't pay). Both paths lead to churn; writeoffs often follow. |
| `loan_writeoff_flag` | 1 if any loan writeoff | **Severe delinquency. Strong churn signal.** Writeoff means the loan was deemed uncollectible. User has likely churned or will churn; writeoff often coincides with account closure. |
| `current_outstanding_loan_amount` | Sum of active loan outstanding | **Financial commitment.** Users with outstanding loans have "skin in the game" — they may stay to repay. However, very high outstanding with low repayment = distress, higher churn risk. |
| `loan_activity_30d` | 1 if loans or loan_paid > 0 | **Binary loan engagement.** Any loan activity in 30d = engaged. Zero = no loans taken or repaid; high churn risk. |

---

## 10. Onboarding & User Profile (MongoDB — onboardingtrackers, lineaccounts, lineusers)

Onboarding and profile features capture *activation*, *bank integration*, and *tenure*. Incomplete onboarding and lost bank connections block core features and drive early churn; feature exploration and tenure are protective factors.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `onboarding_steps_completed` | Count of completed onboarding steps | **Activation.** Incomplete onboarding = user never fully activated. They may churn early because they never experienced the core value. Higher completion = lower early churn. |
| `days_to_onboarding_completion` | Days to complete onboarding | **Activation speed.** Long time to complete = user is stuck or disengaged. Fast completion = strong intent. |
| `has_verified_bank_account` | 1 if any account verified | **Prerequisite for subscription.** Unverified = user cannot use Everdraft or full wallet features. Critical for product value; unverified users churn at higher rates. |
| `days_since_balance_update` | Days since lastBalanceUpdate | **Data freshness / engagement.** Stale balance = user has not used the app recently or bank connection is broken. Long gap = disengagement. |
| `features_explored_count` | Count of features explored | **Product adoption.** More exploration = user has discovered value across the product. Stickier; lower churn. Zero exploration = superficial use. |
| `free_trial_used` | 1 if free trial availed | **Trial users have different churn profile.** Trial users may churn when trial ends if they did not convert; or they may be more price-sensitive. Helps segment risk. |
| `days_since_last_late_fee_waived` | Days since last waiver | **Waiver recency.** Recent waiver = user needed relief recently; may still be at risk. |
| `plaid_revoked` | 1 if Plaid/MX token revoked | **Bank connection lost.** Revoked token = user disconnected their bank. May block Everdraft, balance checks, and other features. Drives churn when user cannot use core functionality. |
| `plaid_accounts_linked` | Count of linked accounts | **Bank integration depth.** More linked accounts = deeper integration. Single account = higher risk if that connection breaks. |
| `tenure_days` | Days since user createdAt | **Account age.** Longer tenure = survivorship; user has already proven they can retain. Early tenure (0–90 days) = higher churn risk. |
| `skip_bank_reconnect_count` | Bank reconnect skips | **Avoided re-linking.** User was prompted to reconnect bank but skipped. May indicate friction, disengagement, or intent to churn. |
| `days_since_signup` | Days since subscription createdAt | **Subscription tenure.** Complements tenure_days; focuses on subscription lifecycle. |
| `no_payment_method_count` | No payment method failures | **Critical for subscription.** Duplicate of Payment & Funds section; no payment method = involuntary churn at next billing. |

---

## 11. AppsFlyer (Attribution & Engagement)

**Source:** `cache/appsflyer_data_filtered.parquet`  
**Join:** Prior month (M-1) to avoid leakage — snapshot month M uses AppsFlyer data from month M-1.  
**Coverage:** AppsFlyer data starts 2025-01; ~38–50% of rows have no match (af_missing=1). Users with no match get 0 for all af_* features.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `af_active_days` | Number of distinct days the user had any AppsFlyer-tracked activity in the prior month | Measures engagement breadth. Users active on fewer days are more likely to churn; low or zero suggests disengagement. Complements backend login data with attribution-side activity. |
| `af_app_opens` | Count of app open events in the prior month (from AppsFlyer SDK) | Direct engagement frequency. Declining or zero app opens indicates the user may have stopped using the app. Correlates with churn risk; helps identify users who are "going dark" before they cancel. |
| `af_logins` | Count of login events in the prior month (from AppsFlyer) | Session frequency from attribution. Low logins = disengagement. May capture login paths (e.g., deep links, re-engagement) that backend logins miss. Declining trend is a leading churn indicator. |
| `af_everdraft_qualified` | Count of Everdraft qualification events in the prior month | Core product usage. Users who qualify for Everdraft are actively using the product; zero suggests they may have stopped engaging with the flagship feature. Strong signal of product-market fit. |
| `af_wallet_topups` | Count of wallet top-up / add-money events in the prior month | Core product usage. Adding money indicates active engagement and intent to use the product. Zero top-ups may indicate the user has moved their financial activity elsewhere. |
| `af_withdrawals` | Count of withdrawal events in the prior month | Core product usage. Withdrawals are a primary use case; users who stop withdrawing may be churning. Complements af_wallet_topups in measuring usage depth. |
| `af_repayments` | Count of repayment events in the prior month | Repayment behavior. Active repayments indicate the user is using the product and staying current. Low or zero may indicate disengagement or that they've stopped taking advances. |
| `af_bank_linked` | Count of bank linking events in the prior month | Onboarding completeness. Users who recently linked a bank are in setup or re-engagement; zero for established users is normal. May help identify users who never completed setup (higher early churn). |
| `af_card_added` | Count of card addition events in the prior month | Payment method setup. Similar to bank linking; users who add cards are actively configuring the product. Zero for established users is normal. |
| `af_unsubscribe_events` | Count of unsubscribe events in the prior month (from AppsFlyer SDK) | **Direct churn intent.** Users who trigger unsubscribe events are explicitly signaling exit. One of the strongest signals; highly sparse (99%+ zeros) but when present, strongly predicts churn. |
| `af_cancel_events` | Count of cancel/subscription cancellation events in the prior month | **Strongest churn signal.** Direct intent to cancel. Very sparse (99.9% zeros) but highest correlation with churn. Prior-month join avoids leakage (same-month cancel would leak the outcome). |
| `af_media_source_organic` | 1 if user was acquired organically (app store, referral), 0 if paid | Acquisition channel. Organic users often have higher intent and lower churn than paid-acquisition users, who may have been drawn in by promotions and have weaker commitment. |
| `af_media_source_category` | 0=organic, 1=top paid (e.g., Facebook, Google), 2=other | Acquisition channel risk. Paid users often churn at higher rates; category helps the model adjust baseline risk by acquisition source. |
| `af_cancel_or_unsubscribe` | 1 if af_cancel_events > 0 or af_unsubscribe_events > 0, else 0 | Combined churn-intent flag. Binary version of the two strongest signals; easier for the model to use when raw counts are sparse. |
| `af_engagement_score` | (af_app_opens + af_logins) / (af_active_days + 1) | Engagement intensity per active day. Normalizes by days active to avoid conflating "few days but very active" with "many days but barely active." High score = engaged; low = at risk. |
| `af_has_activity` | 1 if af_app_opens > 0 or af_logins > 0, else 0 | Binary activity flag. Distinguishes users with any AppsFlyer activity from those with none. When af_missing=0, zero activity is a strong disengagement signal. |
| `af_usage_total` | af_wallet_topups + af_withdrawals + af_repayments | Total core usage volume. Sum of the three primary product actions. Low total = user is not using the product; high = engaged. |
| `af_usage_intensity` | af_usage_total / max(af_active_days, 1) | Usage per active day. Normalizes by activity days; users with high usage on few days may differ from those with low usage spread across many days. |
| `af_user_created` | Count of user creation events in the prior month | Onboarding / new user signal. For users in early lifecycle, high count indicates recent signup; for established users, typically 0. Helps segment new vs mature users. |
| `af_kyc_success` | Count of KYC verification success events in the prior month | Verification completion. Users who complete KYC are progressing through onboarding; zero for established users is normal. May help identify users stuck in verification (higher churn). |
| `af_subscribed` | Count of subscription events in the prior month | Subscription lifecycle. Captures subscription events from AppsFlyer; helps identify users who recently subscribed or renewed. |
| `af_device_category` | 0=phone, 1=tablet, 2=unknown | Device type. Phone vs tablet users may have different engagement patterns. Unknown may indicate tracking gaps. |
| `af_installs` | 1 if install count > 0 in the prior month, else 0 | Install presence. Confirms the user has an install record in AppsFlyer. Helps with the match/no-match distinction when combined with af_missing. |
| `af_everdraft_usage_ratio` | af_everdraft_qualified / max(af_app_opens, 1) | Product usage intensity relative to app opens. High ratio = user opens app primarily for Everdraft; low = opens app but doesn't use core feature. |
| `af_app_opens_trend` | Current month af_app_opens − prior month af_app_opens | Engagement trend. Negative = declining engagement; one of the leading churn indicators. Churners often show -1 to -2+ in this metric. |
| `af_logins_trend` | Current month af_logins − prior month af_logins | **Login trend from AppsFlyer.** Declining logins (negative trend) = user is pulling back from the app. Complements af_app_opens_trend; both capture engagement decline. Negative trend is a strong risk signal — churners often show -1 to -2+ in this metric. |
| `af_missing` | 1 if no AppsFlyer match for (userId, prior month), else 0 | **Critical for interpretation.** ~38–50% of rows have no match (AppsFlyer data starts 2025-01; userId overlap ~50–62%). When af_missing=1, all other af_* are 0. Helps model distinguish "no data" (user may or may not be active) from "truly zero activity" (user has no AppsFlyer events). Without this, the model would treat missing data as zero engagement. |
| `af_has_data` | 1 − af_missing | Inverse of af_missing. 1 = we have AppsFlyer data for this user; 0 = no match. |
| `af_active_when_present` | When af_missing=0, 1 if af_app_opens > 0 or af_logins > 0, else 0 | Activity when data exists. Among users with AppsFlyer data, flags those who had any activity. Helps separate "has data and is active" from "has data but zero activity" (latter = higher churn risk). |

---

## 12. Derived / Composite Features

Derived features combine signals across sources to capture *cross-channel disengagement*, *engagement trends*, and *post-hoc rules* that improve precision and recall. They encode domain knowledge (e.g., "no email and no login = high risk") that the model may not learn from raw features alone.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `no_email_opens_and_no_logins_30d` | 1 if no email opens and no logins in 30d | **Complete disengagement across channels.** User is unreachable via email and has not logged in. One of the strongest composite signals; these users are "going dark" on all touchpoints. |
| `logins_per_tenure` | logins_last_30d / (tenure + 1) | **Engagement normalized by tenure.** A new user with 5 logins is more engaged than a 24-month user with 5 logins (relative to their tenure). Helps the model compare engagement fairly across tenure segments. |
| `fp_active_signal` | 1 if recent login/email/retention engagement | **False-positive rescue.** User looks risky on other features but has recent engagement (login ≤21d or email ≤14d or retention engagement). Helps reduce false alarms — do not flag users who are actually active. |
| `recent_login_or_email` | 1 if login≤21d or email≤14d | **Recent engagement. Protective.** Binary version of fp_active_signal; strong negative correlate of churn. |
| `has_any_engagement_30d` | 1 if email opens≥1 or logins≥1 | **Any engagement.** Distinguishes "completely dark" from "some engagement." Even one open or login reduces churn risk. |
| `engagement_recency_score` | Mean recency (login, email, moe) | **Composite recency.** Averages recency across channels; user with recent login but stale email gets a moderate score. Single metric for "how recently engaged overall." |
| `posthoc_fp_rule_match` | 1 if likely active (login≤7d or logins≥5 or email engaged) | **Post-hoc FP reduction.** Used in production to downgrade predictions for users who are clearly active. Reduces false alarms and improves precision. |
| `posthoc_fn_rule_match` | 1 if likely churner (no login 90d or no email 60d) | **Post-hoc FN catch.** Used to upgrade predictions for users who are clearly dormant. Catches churners the model might miss, improving recall. |
| `dormant_high_risk` | 1 if login>60d, no logins, no loan_paid | **Dormant + no product use.** User has not logged in for 60+ days, has no recent logins, and has not repaid loans. High-risk segment; strong churn signal. |
| `email_bounce_and_no_opens` | 1 if bounce>0 and opens=0 | **Delivery issues + no engagement.** We cannot reach the user (bounces) and they are not engaging. Double risk — unreachable and disengaged. |
| `opted_out_push_and_low_engagement` | 1 if push opt-out and no af activity | **Push off + low engagement.** User opted out of push and has no AppsFlyer activity. Declared disengagement plus behavioral disengagement. |
| `af_missing_and_high_churn_risk` | 1 if af_missing and (no loan_paid or no income_check) | **No AppsFlyer + risk flags.** When we lack AppsFlyer data, users with no loan repayment or unverified income are at elevated risk. Helps avoid underestimating risk for users with incomplete data. |
| `qualification_utilization` | current_outstanding / max(qualified_amount, 1) | **Loan utilization.** High utilization = user is maxed out; may churn when they cannot get more value. Low = headroom; may indicate underuse. |
| `logins_trend` | logins_last_30d − logins_prior_30d | **Login trend.** Declining (negative) = user is pulling back. Leading indicator; often precedes churn by 1–2 months. |
| `engagement_momentum` | email_open_trend + logins_trend | **Combined engagement trend.** Sums email and login trends; negative = declining across both channels. Stronger signal than either alone. |
| `cross_channel_engaged` | 1 if (email or login) and not push opt-out | **Multi-channel engagement.** User engages via at least one channel and has not opted out of push. Healthy engagement profile. |
| `recency_min_channel` | min(days_since_login, days_since_email, moe_days) | **Best recency across channels.** Uses the most recent touchpoint; avoids penalizing users who prefer one channel. |
| `comms_engagement_score` | (opens + clicks) / (push_received + 1) | **Communication engagement.** Normalizes engagement by push volume; high score = user engages when we reach them. |
| `nsf_rate` | nsf_count / max(loans_count, 1) | **NSF per loan.** High rate = user hits NSF frequently relative to loan activity. Indicates payment friction and financial stress. |
| `repayment_rate` | loan_paid_45 / max(loans_count, 1) | **Repayment discipline.** Proportion of loans repaid on time. Low = poor repayment; high churn risk. |
| `nsf_per_loan` | nsf_count / max(loans_count, 1) | **NSF per loan (duplicate of nsf_rate).** Identical formula; normalizes NSF by loan activity. High value = user hits NSF frequently relative to advances taken. Redundant with nsf_rate; consider dropping one to reduce multicollinearity. |
| `loan_repayment_risk` | (1 − repayment_rate) × (1 + nsf_rate) | **Composite repayment risk.** Combines low repayment and high NSF into a single risk score. High value = user is both not repaying and hitting NSF — severe risk. |

---

## 13. Mixpanel — Engagement (30d / 90d Counts)

Mixpanel engagement features capture *in-app behavior* from product analytics — screen opens, feature clicks, and session quality. They complement backend login data with event-level granularity. Zero engagement in Mixpanel is a strong churn signal; it indicates the user has stopped using the app.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `mp_beem_home_opens_last_30d` | Beem Home opens in 30d | **Core app engagement.** Beem Home is the main landing screen. Zero opens = user has not opened the app in 30 days. Strong disengagement signal; churners typically have 0. |
| `mp_app_opens_last_30d` | App/landing screen opens in 30d | **Broader app engagement.** Captures all app/landing opens; complements Beem Home for users who land on other screens. |
| `mp_swipe_app_landing_last_30d` | Swipe on app landing in 30d | **Onboarding/engagement.** Swipe events indicate user is interacting with the landing flow. May capture new or re-engaging users. |
| `mp_login_sessions_last_30d` | Login sessions in 30d | **Session frequency.** Complements backend logins with Mixpanel's session definition. Zero = no Mixpanel-tracked sessions; high churn risk. |
| `mp_autologin_success_last_30d` | Auto-login success in 30d | **Session quality.** Auto-login = seamless return; indicates the user is coming back regularly. |
| `mp_core_feature_clicks_last_30d` | Clicks on core features (add money, withdraw, send, transactions, etc.) | **Depth of usage.** Clicks indicate the user is not just opening the app but using features. High count = engaged; zero = superficial use or disengagement. |
| `mp_everdraft_engagement_last_30d` | Everdraft-related events in 30d | **Core product engagement.** Everdraft is the flagship feature. Zero = user is not engaging with the primary value proposition; strong churn signal. |
| `mp_arcade_engagement_last_30d` | Arcade/play-and-earn in 30d | **Gamification engagement.** Arcade users may be stickier; zero is normal for non-Arcade users but helps segment engagement depth. |
| `mp_send_receive_engagement_last_30d` | Send/receive money events in 30d | **P2P engagement.** P2P is a core use case; engagement indicates active wallet usage. |
| `mp_dues_screen_opens_last_30d` | Dues screen opens in 30d | **Payment awareness.** Users who open the dues screen are aware of subscription status. May indicate payment friction (checking dues) or engagement (managing subscription). |
| `mp_wallet_transactions_last_30d` | Wallet/transaction screen views in 30d | **Financial engagement.** Transaction views indicate the user is monitoring their wallet; zero may indicate disengagement. |
| `mp_ai_gpt_engagement_last_30d` | AI GPT interactions in 30d | **AI feature adoption.** Early indicator of feature adoption; may correlate with stickiness. |
| `mp_activity_section_engagement_last_30d` | Activity feed engagement in 30d | **Social/content engagement.** Activity feed users may be more embedded. |
| `mp_profile_settings_engagement_last_30d` | Profile/settings engagement in 30d | **Account management.** Settings engagement can indicate intent to stay (configuring account) or leave (checking cancellation). |
| `mp_beem_pass_engagement_last_30d` | Beem Pass engagement in 30d | **Social feature.** Beem Pass users may have different engagement profiles. |
| `mp_add_money_success_last_30d` | Add money success in 30d | **Funding behavior.** Add money = active use. Zero = user has not funded in 30d; may indicate they have moved financial activity elsewhere. |
| `mp_getstatus_api_success_last_30d` | getStatus API success in 30d | **Session/status check.** Technical signal; indicates app is communicating with backend. |
| `mp_web_homepage_opens_last_30d` | Web homepage opens in 30d | **Web engagement.** Some users prefer web; captures cross-platform engagement. |
| `mp_web_landing_page_opens_last_30d` | Web landing page opens in 30d | **Web landing engagement.** Captures users who land on the web product (vs app). Complements mp_web_homepage_opens; zero across both = no web engagement. Web-only users may have different churn profiles than app users. |
| `mp_web_login_engagement_last_30d` | Web login engagement in 30d | **Web login.** Web logins complement app logins; zero across both = disengaged. |

---

## 14. Mixpanel — Churn Intent & Flows (90d)

Mixpanel churn-intent features capture *explicit exit behavior* — users who open cancel flows, deactivate flows, or abandon dues. These are among the strongest predictors: they measure intent *before* the user actually churns. Retention teams can target these users for save campaigns.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `mp_cancel_flow_hover_events` | Cancel flow screen opens (hover, not completed) | **Exploring cancel = churn intent.** User has navigated to the cancel flow. Even if they did not complete, the intent is clear. High-priority segment for save campaigns. |
| `mp_cancel_flow_completion_events` | Cancel flow completed | **Actual churn.** User completed the cancel flow. In many cases, this is the churn event itself or immediately precedes it. |
| `mp_cancel_flow_hovered` | 1 if hovered but did not complete | **Strong intent signal.** Binary flag for users who explored cancel but did not complete. "Considered leaving" — ideal for intervention before they return and complete. |
| `mp_deactivate_flow_hover_events` | Deactivate flow screen opens | **Account deactivation intent.** User has navigated to the deactivate flow — exploring full account closure (stronger than cancel, which may be subscription-only). High-priority for save campaigns; deactivation often precedes or equals churn. |
| `mp_deactivate_flow_completion_events` | Deactivate completed | **Account deactivation.** User completed deactivation; often coincides with or precedes churn. |
| `mp_deactivate_flow_abandoned` | 1 if opened deactivate then "I Changed My Mind" | **Explored but stayed.** User considered deactivating but chose to stay. Positive signal — they were at the brink and decided to remain. |
| `mp_wt_unsub_hover_events` | WT unsubscription flow hover | **WT (Wallet/Subscription) cancel intent.** User is exploring WT unsubscription. |
| `mp_wt_unsub_completion_events` | WT unsubscription completed | **WT (Wallet/Subscription) unsubscription completed.** User completed the WT-specific unsubscription flow. Captures a distinct cancel path from the main cancel flow; completion indicates strong churn intent. |
| `mp_wt_unsub_flow_hovered` | 1 if hovered WT unsub but did not complete | **Intent.** Explored WT unsub but did not complete; intervention opportunity. |
| `mp_pending_dues_opened` | Pending dues screen opened | **Payment friction.** User opened the pending dues screen — they are aware of unpaid dues. May indicate they are struggling to pay or considering cancellation. |
| `mp_pending_dues_abandoned` | 1 if opened dues then "I'll do this later" | **Chose not to pay. Churn risk.** User saw dues and explicitly deferred. Strong signal — they had the opportunity to pay and chose not to. High churn risk. |
| `mp_churn_intent_any` | 1 if any of cancel/deactivate/WT/dues hovered | **Combined churn intent.** Single flag for any churn-intent flow. Simplifies targeting for save campaigns. |

---

## 15. Mixpanel — Friction & Errors (90d)

Mixpanel friction features capture *blockers and failures* the user encountered — account blocks, payment failures, bank connection issues, verification failures, and technical errors. Repeated friction erodes trust and drives churn; recovery events (e.g., card add success, bank linked) are positive signals.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `mp_account_blocked_last_90d` | Account blocked events | **Hard friction.** User cannot proceed — account is blocked. Severe experience; often precedes or accompanies churn. |
| `mp_account_restored_last_90d` | Account restored | **Recovery.** User's account was restored; positive signal after a block. |
| `mp_bank_connection_issues_last_90d` | Bank connection/re-link issues | **Friction.** Bank connection problems block Everdraft, balance checks, and funding. User cannot use core features; may churn out of frustration. |
| `mp_card_verification_failed_last_90d` | Card verification failures | **Payment friction.** Card verification failures block payment method setup. User may be unable to pay subscription; drives involuntary churn. |
| `mp_payment_failed_last_90d` | Payment failures | **Payment friction.** Each payment failure is a negative experience. Repeated failures predict churn; user may give up or be unable to pay. |
| `mp_withdrawal_friction_last_90d` | Withdrawal blocked/failed | **Core action blocked.** Withdrawal is a primary use case. Blocked withdrawal = user cannot get their money; severe friction, high churn risk. |
| `mp_idv_failure_last_90d` | IDV/verification failures | **Onboarding/verification friction.** IDV failures block account verification. User may be stuck in onboarding or unable to access features. |
| `mp_api_errors_last_90d` | API/backend errors | **Technical friction.** Backend errors create a poor experience. Repeated errors erode trust and may drive churn. |
| `mp_network_call_failure_last_90d` | Network failures | **Technical friction.** Network issues can prevent the user from completing actions. May indicate connectivity problems or app instability. |
| `mp_otp_verification_failed_last_90d` | OTP verification failed | **Login friction.** OTP failures block login. User may give up and churn if they cannot access the app. |
| `mp_prove_prefill_failed_last_90d` | Prove prefill failed | **Bank linking friction.** Bank linking failed; blocks core setup. |
| `mp_prove_prefill_success_last_90d` | Prove prefill success | **Recovery.** Bank linking succeeded after prior failure. |
| `mp_reactivate_failed_last_90d` | Reactivation failed | **Re-engagement friction.** User tried to reactivate but failed. Lost save opportunity. |
| `mp_subscribing_failed_last_90d` | Subscribing failed | **Subscription friction.** User tried to subscribe but failed. May indicate payment or technical issues. |
| `mp_setting_primary_bank_failed_last_90d` | Primary bank setting failed | **Bank friction.** Could not set primary bank; may block features. |
| `mp_wt_collection_failed_last_90d` | WT collection failed | **Payment friction.** WT (Wallet/Subscription) collection failed; payment problem. |
| `mp_wt_collection_success_last_90d` | WT collection success | **Recovery.** Collection succeeded. |
| `mp_astra_onboarding_failed_last_90d` | Astra onboarding failed | **Card friction.** Card onboarding failed. |
| `mp_astra_ach_failed_last_90d` | Astra ACH failed | **Payment friction.** ACH payment failed. |
| `mp_astra_user_onboarded_last_90d` | Astra user onboarded | **Success.** Card onboarding completed. |
| `mp_no_verified_card_last_90d` | No verified card | **Payment method gap.** User has no verified card; cannot pay subscription. Critical for involuntary churn risk. |
| `mp_status_incomplete_last_90d` | Status incomplete | **Verification gap.** User's status is incomplete; may block features. |
| `mp_mandatory_input_missing_last_90d` | Mandatory input missing | **Form friction.** User could not complete a form; may indicate confusion or technical issue. |
| `mp_unable_to_service_last_90d` | Unable to service screen | **Hard block.** User was shown "unable to service" — product cannot serve them. Strong churn signal. |
| `mp_unable_to_cancel_popup_last_90d` | Unable to cancel popup | **Cancel friction.** User tried to cancel but hit a blocker. May indicate technical issue or retention flow. |
| `mp_dispute_filed_last_90d` | Dispute filed | **Payment dispute.** User filed a dispute; severe dissatisfaction. Often precedes or accompanies churn. |
| `mp_tracking_status_unavailable_last_90d` | Tracking status unavailable | **Technical issue.** Status unavailable; may indicate backend or tracking problems. |
| `mp_unsupported_app_version_last_90d` | Unsupported app version | **Update friction.** User is on an unsupported version; may be blocked from features or have a poor experience. |
| `mp_create_stripe_unsupported_plan_last_90d` | Unsupported plan selected | **Subscription friction.** User selected an unsupported plan; subscription flow failed. |
| `mp_duplicate_payment_last_90d` | Duplicate payment | **Payment issue.** Duplicate payment may indicate user confusion or system error; can cause frustration. |
| `mp_medical_cancel_started_last_90d` | Medical cancel started | **Plan change.** User started medical/hardship cancel flow. |
| `mp_medical_cancel_completed_last_90d` | Medical cancel completed | **Medical/hardship cancel completed.** User completed a medical or hardship-based cancellation flow. These users may churn for different reasons (e.g., financial hardship); helps segment churn drivers. |
| `mp_subscription_downgrade_request_last_90d` | Downgrade request | **Downgrade intent.** User requested downgrade; may precede full churn. |
| `mp_transaction_payment_lock_initiated_last_90d` | Payment lock initiated | **Payment hold.** Payment was locked; may indicate risk or dispute. |
| `mp_transaction_payment_lock_removed_last_90d` | Payment lock removed | **Payment lock resolution.** User had a payment lock that was subsequently removed. Positive signal — issue was resolved. Complements mp_transaction_payment_lock_initiated for full lock lifecycle. |
| `mp_add_new_card_process_started_last_90d` | Add card process started | **Card setup.** User is trying to add a card; may indicate payment method problems. |
| `mp_cancel_request_already_queued_last_90d` | Cancel already queued | **Duplicate cancel.** User has already requested cancel; strong churn intent. |
| `mp_sardine_status_pending_last_90d` | Sardine status pending | **Verification.** Verification (Sardine) pending; user may be stuck. |
| `mp_sardine_status_qualified_last_90d` | Sardine qualified | **Verification success.** User qualified; positive signal. |
| `mp_dev_deeplink_logged_out_last_90d` | Dev deeplink logged out | **Edge case.** Development/deeplink edge case; low volume. |
| `mp_clicked_dont_want_to_cancel_last_90d` | Clicked "Don't want to cancel" | **Retention action. Positive.** User was in cancel flow but chose to stay. Strong positive signal. |
| `mp_keep_or_resubscribe_last_90d` | Keep or resubscribe actions | **Retention action.** User chose to keep or resubscribe; positive. |
| `mp_card_add_success_last_90d` | Card add success | **Recovery.** User successfully added a card; resolved payment method gap. |
| `mp_bank_account_linked_last_90d` | Bank linked | **Recovery.** User linked a bank; resolved bank connection gap. |
| `mp_document_uploaded_last_90d` | Document uploaded | **Verification.** User uploaded document; progressing verification. |
| `mp_restore_application_opens_last_90d` | Restore application opened | **Re-engagement.** User opened restore flow; may be returning after churn. |

---

## 16. Mixpanel — Recency (Days Since)

Mixpanel recency features measure *how long since* key events. They complement count-based engagement with time-based signals. Long gaps (e.g., 60+ days) indicate disengagement; recent activity is protective.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `mp_days_since_last_add_money_success` | Days since last add money success (999 if never) | **Funding recency.** Add money is a core action. Long gap = user has not funded in a long time; they may have moved financial activity elsewhere. 999 = never (e.g., new user or never used add money). |
| `mp_days_since_last_beem_home_open` | Days since Beem Home open | **Core engagement recency.** Beem Home is the main screen. Long gap = user has not opened the app recently; strong disengagement signal. |
| `mp_days_since_last_core_click` | Days since last core feature click | **Usage recency.** Core clicks (add money, withdraw, send, etc.) indicate active use. Long gap = user is not using features; may be superficial or disengaged. |
| `mp_days_since_last_dues_screen` | Days since dues screen open | **Payment awareness recency.** Dues screen = user is aware of subscription status. Long gap may indicate they have stopped checking — or stopped caring. |
| `mp_days_since_last_login` | Days since login (Mixpanel) | **Session recency.** Complements backend days_since_last_login with Mixpanel's login events. Long gap = no Mixpanel-tracked logins; disengagement. |

---

## 17. Mixpanel — Derived

Mixpanel derived features combine raw Mixpanel events into composite signals. They encode domain knowledge (e.g., "zero Beem Home + zero logins = high risk") and simplify model interpretation.

| Feature | Definition | Why Useful for Churn |
|---------|------------|----------------------|
| `mp_zero_engagement_last_30d` | 1 if 0 Beem Home opens and 0 login sessions in 30d | **Complete app disengagement.** User has neither opened Beem Home nor had a login session in 30 days. One of the strongest Mixpanel signals; these users have effectively stopped using the app. |
| `mp_app_vs_web_ratio_last_30d` | App opens / (app + web opens); 0.5 if both 0 | **Channel preference.** 1 = app-only; 0 = web-only; 0.5 = no data. App-only users may have different engagement and churn profiles than web users. |
| `mp_friction_count_total_last_90d` | Sum of bank issues, card fail, payment fail, withdrawal friction, IDV fail, account blocked | **Composite friction.** Single metric for total friction experienced. High count = user hit multiple blockers; bad experience, high churn risk. Complements individual friction features. |

---

## Summary by Group

| Group | Count | Description |
|-------|-------|-------------|
| **Base 11** | 11 | Core product usage (loans, repayment), payment health (NSF, card), risk flags (velocity, volatility, tickets, feedback). Strongest individual predictors; model foundation. |
| **Subscription & Billing** | 10 | Renewals, tenure, prior churn, downgrades/upgrades, late fees, waivers. Captures subscription lifecycle and payment delinquency. |
| **Logins & Sessions** | 9 | Login frequency, recency, failure ratio, devices. Engagement recency is among the top predictors. |
| **Payment & Funds** | 6 | Card expiry, payment method gaps, Stripe failures, payment recency. Direct causes of involuntary churn. |
| **Recurring Transactions** | 2 | Financial planning engagement. Users tracking recurring expenses are more embedded. |
| **MoEngage** | 28 | Email opens/clicks, push, retention campaigns, trends. Communication engagement predicts churn; no opens = disengagement. |
| **Behavioural Score Metrics** | 12 | Income, cash flow, overdraft, NSF, qualification. Financial health from bank data. |
| **Qualification Metrics** | 9 | LQM scores, liquidity, income gaps, fees. Lending-risk view. |
| **Everdraft & Loans** | 7 | Everdraft recency, usage, repayment, writeoffs. Core product usage; top predictors. |
| **Onboarding & User** | 13 | Onboarding completion, bank verification, Plaid, tenure. Activation and integration depth. |
| **AppsFlyer** | 28 | Attribution, app opens, logins, cancel/unsubscribe events, trends. In-app engagement from attribution; af_cancel and af_unsubscribe are strong signals. |
| **Derived/Composite** | 22 | Cross-channel disengagement, engagement trends, FP/FN rules, repayment risk. Encodes domain knowledge. |
| **Mixpanel Engagement** | 20 | Beem Home, app opens, core clicks, Everdraft, P2P, dues. In-app behavior from product analytics. |
| **Mixpanel Churn Intent** | 12 | Cancel, deactivate, WT flows, pending dues abandoned. Explicit exit behavior; high-priority for save campaigns. |
| **Mixpanel Friction** | 47 | Account blocks, payment failures, bank/card issues, verification failures. Friction erodes trust and drives churn. |
| **Mixpanel Recency** | 5 | Days since add money, Beem Home, core click, dues, login. Time-based engagement signals. |
| **Mixpanel Derived** | 3 | Zero engagement, app vs web ratio, friction total. Composite Mixpanel signals. |
| **Total** | **245** | |

---

## Key Takeaways for Stakeholders

1. **Product usage is the strongest signal.** Features like `loans_count_in_month`, `days_since_last_everdraft`, and `everdraft_usage_count_30d` consistently rank at the top. Users who stop using Everdraft have effectively decided the product is no longer valuable.

2. **Engagement recency matters.** `days_since_last_login`, `days_since_last_email_open`, and Mixpanel recency features capture "going dark" — users who stop engaging before they cancel. These are leading indicators.

3. **Payment health drives involuntary churn.** Card expiry, NSF, Stripe failures, and no payment method are direct causes of subscription payment failure. Proactive interventions (card-update prompts, dunning) can prevent churn.

4. **Churn intent flows are actionable.** Mixpanel cancel/deactivate/dues flows identify users who are *considering* churn. These are ideal targets for save campaigns before they complete the exit.

5. **Communication engagement predicts retention.** Users who open emails and engage with retention campaigns are more likely to stay. No email opens + no logins = high risk.

6. **Friction erodes trust.** Bank connection issues, payment failures, and account blocks create negative experiences. Reducing friction and monitoring friction events can improve retention.

---

*Last updated: March 2026. Dataset: 150_complete_churn_feature_data_with_mixpanel.parquet.*
