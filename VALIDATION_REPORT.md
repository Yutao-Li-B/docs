# Sentiment & Rating Model Validation Report

**Date:** February 5, 2026  
**Dataset:** Beem App Reviews from MongoDB  
**Baseline:** OpenAI GPT-4.1-mini  
**Models Tested:** HuggingFace Local Inference Pipeline

---

## Executive Summary

This report validates the performance of our HuggingFace-based sentiment and rating prediction models against OpenAI GPT-4.1-mini as the ground truth baseline. We analyzed reviews from both the App Store (5,515 reviews) and Play Store (6,373 reviews).

### Key Findings

| Metric | App Store | Play Store |
|--------|-----------|------------|
| **Sentiment Accuracy** | 95.8% | 91.2% |
| **Rating Accuracy (Random Sample)** | 96.7% | 94.1% |
| **Rating FP Rate (Random Sample)** | 8.2% | 8.5% |
| **Rating FP Rate (Extreme Cases)** | 59.8% | 73.1% |

**Sentiment Analysis:** Excellent performance on positive/negative classification (96-98% precision), but struggles with neutral sentiment (36-42% precision).

**Rating Prediction:** 
- **Random Sample:** The `Yanni8/star-predictor` model performs well overall with **94-97% accuracy** and low **8% False Positive Rate**.
- **Extreme Cases:** The model shows **systematic bias** in extreme mismatch scenarios (actual vs predicted differ by 3-4 stars), with False Positive Rates of 60-73%.

---

## Part 1: Sentiment Analysis Validation

### Methodology

- **Sample Size:** 500 random reviews per source
- **Model:** `finiteautomata/bertweet-base-sentiment-analysis`
- **Baseline:** OpenAI GPT-4.1-mini classifying sentiment as positive/negative/neutral
- **Metrics:** Accuracy, Precision, Recall, F1 Score, False Positive Rate

### Overall Results

| Metric | App Store | Play Store |
|--------|-----------|------------|
| **Overall Accuracy** | **95.8%** (479/500) | **91.2%** (456/500) |
| Sample Size | 500 | 500 |

### Confusion Matrices

#### App Store
```
                     HuggingFace Predicted
OpenAI (Truth)    positive  negative  neutral  Total
positive              292         1        4    297
negative                0       179        7    186
neutral                 6         3        8     17
Total                 298       183       19    500
```

#### Play Store
```
                     HuggingFace Predicted
OpenAI (Truth)    positive  negative  neutral  Total
positive              177         1        9    187
negative                3       260       24    287
neutral                 4         3       19     26
Total                 184       264       52    500
```

### Per-Sentiment Metrics

#### App Store

| Sentiment | TP | TN | FP | FN | Precision | Recall | F1 | FPR |
|-----------|----|----|----|----|-----------|--------|-----|-----|
| **Positive** | 292 | 197 | 6 | 5 | 98.0% | 98.3% | 98.2% | 3.0% |
| **Negative** | 179 | 310 | 4 | 7 | 97.8% | 96.2% | 97.0% | 1.3% |
| **Neutral** | 8 | 472 | 11 | 9 | 42.1% | 47.1% | 44.4% | 2.3% |

#### Play Store

| Sentiment | TP | TN | FP | FN | Precision | Recall | F1 | FPR |
|-----------|----|----|----|----|-----------|--------|-----|-----|
| **Positive** | 177 | 306 | 7 | 10 | 96.2% | 94.7% | 95.4% | 2.2% |
| **Negative** | 260 | 209 | 4 | 27 | 98.5% | 90.6% | 94.4% | 1.9% |
| **Neutral** | 19 | 441 | 33 | 7 | 36.5% | 73.1% | 48.7% | 7.0% |

### Sentiment Analysis Insights

1. **Excellent Positive/Negative Classification**
   - Precision: 96-98% for both classes
   - Very low false positive rates (1-3%)
   - The model reliably distinguishes positive from negative sentiment

2. **Weak Neutral Detection**
   - Precision: Only 36-42%
   - The model tends to over-classify neutral reviews as positive or negative
   - This is expected behavior for sentiment models trained on polarized data

3. **App Store vs Play Store**
   - App Store performs better (95.8% vs 91.2% accuracy)
   - Play Store reviews may be shorter or more ambiguous

---

## Part 2: Extreme Rating Mismatch Validation

### Methodology

We analyzed cases where the predicted rating differs drastically from the actual rating (e.g., actual=1 star, predicted=5 stars). OpenAI was used to classify whether the **review text** represents a genuinely HIGH (4-5 stars) or LOW (1-2 stars) review.

**Classification Framework:**
- **True Positive (TP):** OpenAI says HIGH, HF predicts HIGH → Correct
- **True Negative (TN):** OpenAI says LOW, HF predicts LOW → Correct
- **False Positive (FP):** OpenAI says LOW, HF predicts HIGH → **Error** (bad review called good)
- **False Negative (FN):** OpenAI says HIGH, HF predicts LOW → **Error** (good review called bad)

### Extreme Case Distribution

| Mismatch Type | App Store | Play Store | Description |
|---------------|-----------|------------|-------------|
| **1→5** | 87 (1.58%) | 187 (2.93%) | Actual 1 star, Predicted 5 stars |
| **5→1** | 48 (0.87%) | 63 (0.99%) | Actual 5 stars, Predicted 1 star |
| **1→4** | 14 (0.25%) | 29 (0.46%) | Actual 1 star, Predicted 4 stars |
| **4→1** | 15 (0.27%) | 30 (0.47%) | Actual 4 stars, Predicted 1 star |
| **2→5** | 36 (0.65%) | 39 (0.61%) | Actual 2 stars, Predicted 5 stars |
| **Total** | 200 (3.63%) | 348 (5.46%) | All extreme cases |

### Binary Classification Results

| Metric | App Store | Play Store |
|--------|-----------|------------|
| **Samples** | 140 | 159 |
| **True Positive (TP)** | 10 | 3 |
| **True Negative (TN)** | 51 | 36 |
| **False Positive (FP)** | 76 | 98 |
| **False Negative (FN)** | 3 | 22 |
| **Accuracy** | 43.6% | 24.5% |
| **Precision** | 11.6% | 3.0% |
| **Recall (TPR)** | 76.9% | 12.0% |
| **Specificity (TNR)** | 40.2% | 26.9% |
| **False Positive Rate** | **59.8%** | **73.1%** |
| **False Negative Rate** | 23.1% | 88.0% |

### Confusion Matrices

#### App Store (140 extreme cases)
```
                         HF Predicts
OpenAI Says          HIGH      LOW    Total
HIGH (good review)     10        3       13
LOW (bad review)       76       51      127
Total                  86       54      140
```

#### Play Store (159 extreme cases)
```
                         HF Predicts
OpenAI Says          HIGH      LOW    Total
HIGH (good review)      3       22       25
LOW (bad review)       98       36      134
Total                 101       58      159
```

### Breakdown by Mismatch Type

| Mismatch | Source | n | TP% | TN% | **FP%** | FN% |
|----------|--------|---|-----|-----|---------|-----|
| **1→5** | App Store | 47 | 6.4% | 0% | **93.6%** | 0% |
| **1→5** | Play Store | 43 | 0% | 0% | **100%** | 0% |
| **1→4** | App Store | 13 | 0% | 0% | **100%** | 0% |
| **1→4** | Play Store | 27 | 0% | 0% | **100%** | 0% |
| **2→5** | App Store | 26 | 26.9% | 0% | **73.1%** | 0% |
| **2→5** | Play Store | 31 | 9.7% | 0% | **90.3%** | 0% |
| **5→1** | App Store | 41 | 0% | 95.1% | 0% | 4.9% |
| **5→1** | Play Store | 38 | 0% | 42.1% | 0% | 57.9% |
| **4→1** | App Store | 13 | 0% | 92.3% | 0% | 7.7% |
| **4→1** | Play Store | 20 | 0% | 100% | 0% | 0% |

### Critical Finding: High False Positive Rate

**When HuggingFace predicts a HIGH rating (4-5 stars) in extreme mismatch cases:**
- **93-100% of 1→5 predictions are False Positives**
- **100% of 1→4 predictions are False Positives**
- **73-90% of 2→5 predictions are False Positives**

This means the `Yanni8/star-predictor` model has a **systematic bias toward predicting high ratings** even when the review text is clearly negative.

### Example False Positives

| Review Title | Review Text | Actual | HF | OpenAI |
|--------------|-------------|--------|-----|--------|
| "scam" | "App was made by potatoes with hands" | 1 ⭐ | 5 ⭐ | LOW |
| "Upgrade your website" | "Enough said…" | 1 ⭐ | 5 ⭐ | LOW |
| N/A | "Its not a cash advance app. It just find lenders. So don't get scammed." | 1 ⭐ | 5 ⭐ | LOW |
| N/A | "BEWARE OF EVERDRAFT! I got this app because..." | 1 ⭐ | 5 ⭐ | LOW |

### Example False Negatives

| Review Title | Review Text | Actual | HF | OpenAI |
|--------------|-------------|--------|-----|--------|
| "Saving Grace" | "Was between paydays when my daughter caught RSV..." | 5 ⭐ | 1 ⭐ | HIGH |
| N/A | "So far this thing rocks!!" | 5 ⭐ | 1 ⭐ | HIGH |
| N/A | "Had to use this a couple times to help with gas..." | 5 ⭐ | 1 ⭐ | HIGH |

### Why 5→1 Has High True Negative Rate

Interestingly, for **5→1 mismatches** (actual 5 stars, HF predicts 1 star):
- App Store: 95.1% True Negative
- Play Store: 42.1% True Negative

This suggests many users gave 5-star ratings despite writing negative reviews (possibly user error, sarcasm, or misunderstanding the rating system). Both HuggingFace and OpenAI correctly identify the negative content.

---

## Part 4: Metric Definitions

### Confusion Matrix Terms

| Term | Definition |
|------|------------|
| **True Positive (TP)** | Both OpenAI AND HuggingFace agree on the classification (correctly identified) |
| **True Negative (TN)** | Both agree it's NOT this class (correctly rejected) |
| **False Positive (FP)** | HuggingFace says positive/high, but OpenAI disagrees ("false alarm") |
| **False Negative (FN)** | OpenAI says positive/high, but HuggingFace misses it ("missed detection") |

### Derived Metrics

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| **Accuracy** | (TP + TN) / Total | Overall correct rate |
| **Precision** | TP / (TP + FP) | When model says X, how often is it correct? |
| **Recall (TPR)** | TP / (TP + FN) | Of all actual X, how many did we catch? |
| **Specificity (TNR)** | TN / (TN + FP) | Of all actual NOT-X, how many did we correctly reject? |
| **False Positive Rate (FPR)** | FP / (FP + TN) | Of all actual NOT-X, how many did we wrongly label as X? |
| **False Negative Rate (FNR)** | FN / (FN + TP) | Of all actual X, how many did we miss? |
| **F1 Score** | 2 × (P × R) / (P + R) | Harmonic mean of precision and recall |

---

## Part 4: Conclusions & Recommendations

### Sentiment Model: Strong Performance ✅

The `finiteautomata/bertweet-base-sentiment-analysis` model performs well:
- **95.8%** accuracy on App Store reviews
- **91.2%** accuracy on Play Store reviews
- Excellent positive/negative classification (96-98% precision)

**Recommendation:** Continue using this model for sentiment analysis. Consider post-processing to combine positive/negative classifications only, treating neutral as a fallback.

### Rating Model: Concerning Bias ⚠️

The `Yanni8/star-predictor` model shows significant issues:
- **59-73% False Positive Rate** in extreme cases
- Systematically predicts high ratings for negative reviews
- Only **24-44% accuracy** on extreme mismatch cases

**Recommendation:** Consider alternatives:
1. **Use sentiment as a proxy for rating** (positive → 4-5 stars, negative → 1-2 stars)
2. **Try `nlptown/bert-base-multilingual-uncased-sentiment`** (more conservative predictions)
3. **Fine-tune a model on Beem app reviews specifically**
4. **Use OpenAI for rating prediction** (more accurate but higher cost/latency)

### Business Impact

For a cash advance app like Beem:
- **High FP rate is dangerous**: Negative user experiences may appear as satisfied customers
- **Customer complaints about fees/service could be missed**
- **Business decisions based on ratings could be misguided**

---

## Appendix: Files Generated

| File | Description |
|------|-------------|
| `results/sentiment_validation_metrics.csv` | Sentiment validation summary metrics |
| `results/app_store_sentiment_validation_sample.csv` | 500 App Store samples with OpenAI labels |
| `results/play_store_sentiment_validation_sample.csv` | 500 Play Store samples with OpenAI labels |
| `results/rating_random_sample_metrics.csv` | Random sample rating validation metrics |
| `results/rating_random_sample_validation.csv` | 500 samples per source with rating classifications |
| `results/extreme_rating_binary_metrics.csv` | Extreme rating validation metrics |
| `results/extreme_rating_binary_samples.csv` | All extreme case samples with classifications |
| `results/extreme_rating_binary_breakdown.csv` | Breakdown by mismatch type |

---

## Appendix: Models Used

| Task | Model | Source |
|------|-------|--------|
| Sentiment | `finiteautomata/bertweet-base-sentiment-analysis` | HuggingFace |
| Rating | `Yanni8/star-predictor` | HuggingFace |
| Emotion | `SamLowe/roberta-base-go_emotions` | HuggingFace |
| Baseline | `gpt-4.1-mini` | OpenAI |

---

*Report generated: February 5, 2026*
