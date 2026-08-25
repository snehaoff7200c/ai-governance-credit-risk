# Model Card: Credit Default Risk Model

## Overview
| | |
|---|---|
| **Model name** | `credit-default-risk-model` (v1) |
| **Model type** | Logistic Regression (BigQuery ML) |
| **Registered in** | Vertex AI Model Registry (Gemini Enterprise Agent Platform → Models) |
| **Owner / Steward** | Sneha Munipally |
| **Date trained** | [today's date] |
| **Risk classification** | **High-Risk** — credit scoring / creditworthiness assessment falls under Annex III of the EU AI Act as a high-risk AI use case |

## Purpose and Intended Use
Predicts the likelihood that a credit card applicant will default on payment in the following month, based on demographic and repayment history features. Intended as a **decision-support tool for human underwriters** — not for fully automated approval/rejection decisions.

## Training Data
- **Source:** `bigquery-public-data.ml_datasets.credit_card_default` (Taiwan Credit Card Default dataset), copied into `credit_risk.applicant_data`
- **Size:** ~30,000 records
- **Features used:** credit limit, sex, education level, marital status, age, six months of repayment status (`pay_0`–`pay_6`), six months of bill amounts, six months of payment amounts
- **Label:** `default_payment_next_month`
- **Known data quality issue:** `pay_5` and `pay_6` are typed as STRING while `pay_0`–`pay_4` are FLOAT — a type inconsistency identified during cataloging. [Note how you handled this, or flag as a follow-up fix.]

## Performance
| Metric | Value |
|---|---|
| Accuracy | 80.6% |
| Precision | 0.69 |
| **Recall** | **0.23** |
| F1 Score | 0.34 |
| ROC AUC | 0.685 |

## Known Limitations — read this section first
**High accuracy is misleading here.** The dataset is imbalanced (most applicants do not default), so accuracy alone overstates real-world performance. The metric that matters most for this use case is **recall (0.23)** — the model correctly identifies only 23% of applicants who actually default. In production, this means roughly 3 out of 4 real defaulters would be missed.

**This model is not recommended for production deployment as-is.** Before any real use, it would need:
- Class rebalancing (e.g., weighted loss, oversampling) or threshold tuning to improve recall
- A cost-sensitive evaluation — in credit risk, a missed defaulter (false negative) is typically far more costly than a false alarm, so recall should likely be prioritized over precision
- Re-evaluation after rebalancing before any human-in-the-loop deployment

## Fairness Considerations

A group-wise evaluation was run by slicing `ML.EVALUATE` on the `sex` attribute (flagged as a Protected Attribute during cataloging).

| Segment | Precision | Recall | Accuracy | ROC AUC |
|---|---|---|---|---|
| Male (1) | 0.696 | **0.271** | 0.809 | 0.771 |
| Female (2) | 0.800 | **0.193** | 0.823 | 0.737 |

**Finding:** Recall differs meaningfully across `sex` — the model identifies actual defaulters at a 27.1% rate for male applicants versus 19.3% for female applicants. Using the four-fifths rule commonly applied in disparate-impact analysis (a cross-group ratio below 0.80 warrants review), the recall ratio here is **0.71 (0.193 ÷ 0.271)** — below that threshold.

**Notably, accuracy and precision are both slightly *higher* for the female group**, which would mask this disparity if accuracy were the only metric reviewed. This reinforces the model's core limitation: aggregate accuracy is an unreliable signal on its own, including for fairness review, and can hide unevenly distributed errors across protected groups.

**Implication:** before any production use, this disparity would need investigation — e.g., checking whether the imbalance stems from underlying label distribution differences by group, feature correlations with `sex`, or model calibration — and should be a required part of sign-off, not an optional check. `age`, `education_level`, and `marital_status` were also tagged as Protected Attributes but were not sliced in this pass; a full fairness review would repeat this evaluation across all four.

## Live Endpoint Validation
The model (v3, `SAFE_CAST`-corrected) was deployed to a live endpoint and tested against two known defaulters pulled directly from the dataset, to confirm the offline recall limitation holds under real-time serving conditions — not just in batch evaluation.

| Case | Actual outcome | Model prediction | Confidence (no-default) |
|---|---|---|---|
| Male defaulter (id 4169) | Defaulted | **Missed** — predicted "no default" | 67.6% |
| Female defaulter (id 15190) | Defaulted | **Missed** — predicted "no default" | 80.0% |

Both real defaulters in this small live-traffic test were missed, consistent with the model's documented recall of 0.23-0.27. This is treated as illustrative, not statistical — two cases don't establish a live-serving fairness gap on their own — but the confidence gap between the two misses (a closer 67.6% call for the male case vs. a more confident 80.0% miss for the female case) is directionally consistent with the batch fairness finding above, and worth further validation at scale before production use.

The endpoint was undeployed immediately after this test to avoid unnecessary hosting cost.

## Human Oversight
All predictions from this model are intended to support, not replace, a human underwriter's decision. Any applicant flagged as high-risk should be routed to manual review rather than auto-rejected, given the recall limitation documented above.

## Governance Artifacts
- Cataloged in Dataplex / Knowledge Catalog under `credit-risk-lake → curated-zone`
- Business glossary terms linked: `Credit Default Risk`, `Protected Attribute`
- Sensitive fields tagged with `Fairness Monitoring Required` aspect: `sex`, `education_level`, `marital_status`, `age`
- Lineage tracked automatically from `applicant_data` → `default_risk_model` via BigQuery Data Lineage
