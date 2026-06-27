# Technical Specification: ML-Only Market Abuse Alert Prioritization System
## Investment Banking Compliance — Surveillance & Monitoring

**Version:** 1.0  
**Date:** 2026-06-27  
**Classification:** Internal — Compliance Technology  
**Status:** Draft for Review  

---

## 1. Executive Summary

This specification defines the architecture, data, and machine learning requirements for a **fully automated, ML-only alert prioritization system** for Market Abuse Detection (MAD) within investment banking compliance. The system replaces rule-based severity scoring with a three-stage probabilistic model that learns directly from confirmed regulatory violations and unlabeled alert history.

**Key Design Principles:**
- **No hard-coded rules:** All prioritization logic is learned from data.
- **Three operational tiers:** Escalate (Tier 1), Standard (Tier 2), Automated (Tier 3).
- **Positive-Unlabeled (PU) learning:** Acknowledges that "closed as NFA" does not equal "no violation."
- **Cost-optimal thresholding:** Tier boundaries are learned from the firm's specific cost structure, not fixed percentiles.
- **Regulatory defensibility:** Full explainability, audit trails, and model risk management compliance.

---

## 2. Problem Statement & Scope

### 2.1 Current State
Upstream MAD surveillance engines (e.g., Behavox, NICE Actimize, Symphony) generate alerts based on rule-based or statistical anomaly detection. These engines produce high false-positive rates (typically 85–98%), overwhelming compliance investigators and delaying response to genuine market abuse.

### 2.2 Objective
Build a post-alert prioritization layer that:
1. Scores every MAD alert with a calibrated risk probability.
2. Assigns alerts to exactly one of three operational tiers.
3. Auto-closes Tier 3 alerts with full audit logging.
4. Maximizes the yield of confirmed violations in Tier 1 while respecting investigator capacity constraints.

### 2.3 Scope Boundaries
- **In-scope:** Equity, fixed income, FX, and derivatives cash/spot trading alerts; order book manipulation; insider dealing; wash trading; layering; spoofing; front-running.
- **Out-of-scope:** Pre-trade controls, communication surveillance (e.g., e-comms/v-comms NLP), AML transaction monitoring, sanctions screening.

### 2.4 Constraints
- **Latency:** < 50ms end-to-end per alert for real-time scoring.
- **Throughput:** Minimum 10,000 alerts/day (peak: 50,000 alerts/day).
- **Explainability:** Every Tier 1 and Tier 2 alert must have a human-readable justification.
- **Audit:** Complete reproducibility of any tier assignment decision for 7 years.

---

## 3. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ALERT PRIORITIZATION SYSTEM                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────┐   │
│   │   Upstream   │───►│   Feature Store  │───►│  3-Stage ML Stack    │   │
│   │   MAD Engine │    │   (Temporal)     │    │                      │   │
│   └──────────────┘    └──────────────────┘    │  ┌────────────────┐    │   │
│                                               │  │ Stage 1: PU    │    │   │
│                                               │  │ Violation Prob │    │   │
│                                               │  └───────┬────────┘    │   │
│                                               │          │             │   │
│                                               │  ┌───────▼────────┐    │   │
│                                               │  │ Stage 2: Cond. │    │   │
│                                               │  │ Severity       │    │   │
│                                               │  └───────┬────────┘    │   │
│                                               │          │             │   │
│                                               │  ┌───────▼────────┐    │   │
│                                               │  │ Stage 3: Comp. │    │   │
│                                               │  │ Risk Score     │    │   │
│                                               │  └───────┬────────┘    │   │
│                                               └──────────┼────────────┘   │
│                                                          │                │
│   ┌──────────────────────────────────────────────────────┼──────┐       │
│   │              TIER ASSIGNMENT & ROUTING                 │      │       │
│   │  ┌────────────────┐  ┌────────────────┐  ┌───────────▼───┐  │       │
│   │  │   TIER 1       │  │   TIER 2       │  │   TIER 3      │  │       │
│   │  │   Escalate     │  │   Standard     │  │   Automated   │  │       │
│   │  │   < 4 hrs SLA  │  │   < 48 hrs SLA │  │   Auto-close  │  │       │
│   │  └────────────────┘  └────────────────┘  └───────────────┘  │       │
│   └───────────────────────────────────────────────────────────┘       │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐     │
│   │  MONITORING & GOVERNANCE                                        │     │
│   │  - Drift Detection (PSI, SHAP drift)                            │     │
│   │  - Tier 3 Random Audit (2% weekly)                              │     │
│   │  - Model Versioning & Audit Trail                               │     │
│   │  - Regulatory Reporting Interface                               │     │
│   └─────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Data & Feature Engineering

### 4.1 Data Sources

| Source | Data | Latency | Retention |
|--------|------|---------|-----------|
| **OMS/EMS** | Order, execution, cancellation messages | Real-time | 7 years |
| **Market Data** | L1/L2 tick data, reference prices, corporate actions | Real-time | 7 years |
| **HR / Entitlement** | Trader tenure, desk assignment, role changes | Daily batch | 10 years |
| **Investigation Outcomes** | Confirmed violations, NFA closures, regulatory actions | Event-driven | Permanent |
| **Corporate Events** | Earnings calendars, M&A announcements, macro events | Daily batch | 7 years |
| **News / NLP** | Issuer news volume, sentiment (metadata only) | Hourly | 2 years |

### 4.2 Feature Taxonomy

All features are **continuous or embedded**. No binary rule flags are permitted as model inputs.

#### 4.2.1 Alert Metadata
| Feature | Construction | Rationale |
|---------|-------------|-----------|
| `upstream_rule_embedding` | Target encoding of detection rule type, regularized by violation rate | Rules have different base rates |
| `upstream_anomaly_score` | Raw anomaly score from upstream engine | Captures signal the upstream engine already found |
| `concurrent_rule_count` | Number of distinct rules triggered by same entity/instrument in 15-min window | Multi-signal alerts are higher risk |
| `alert_sequence_day` | Nth alert for this trader today | Escalating pattern detection |

#### 4.2.2 Trader Behavioral Profile
| Feature | Construction | Rationale |
|---------|-------------|-----------|
| `historical_violation_rate` | Confirmed violations / total alerts, exponentially weighted (half-life 1 year) | Recidivism is a strong signal |
| `volume_zscore_90d` | Trader's daily volume vs. personal 90-day mean/std | Detects unusual activity |
| `order_trade_entropy` | Shannon entropy of order-to-trade ratio over 30 days | Low entropy = robotic / structured behavior |
| `instrument_hhi` | Herfindahl-Hirschman Index of trading concentration across ISINs | Concentration increases manipulation risk |
| `tenure_embedding` | Learned embedding of tenure buckets (0-1yr, 1-3yr, 3-7yr, 7yr+) | Junior traders have different patterns |

#### 4.2.3 Execution Microstructure
| Feature | Construction | Rationale |
|---------|-------------|-----------|
| `realized_spread_deviation` | (Execution price - mid) / historical spread for ISIN | Aggressive execution timing |
| `price_impact_coefficient` | Signed return in 1-min post-trade / order size | Market impact proxy |
| `cancellation_rate_60s` | Cancelled orders / total orders in 60s pre-execution | Layering / spoofing signal |
| `trade_to_order_size_ratio` | Executed size / average order size | Iceberg / hidden intent detection |
| `passive_aggressive_ratio` | Passive fills vs. aggressive fills | Manipulators often use passive orders |

#### 4.2.4 Temporal Context
| Feature | Construction | Rationale |
|---------|-------------|-----------|
| `hours_to_next_event` | Continuous hours to next earnings, M&A, or macro event | Insider trading window |
| `vix_percentile` | Current VIX vs. 90-day distribution | Volatility affects normal behavior |
| `market_volume_percentile` | Current market volume vs. 30-day distribution | Contextualizes trader volume |
| `day_of_week_embedding` | Learned embedding (Mon-Fri) | End-of-week patterns differ |
| `month_end_flag` | Continuous proximity to month/quarter end (0-1) | Window dressing risk |

#### 4.2.5 Network & Counterparty
| Feature | Construction | Rationale |
|---------|-------------|-----------|
| `counterparty_gini` | Gini coefficient of counterparty concentration | Concentrated counterparty = wash trade risk |
| `comm_graph_centrality` | Eigenvector centrality in communication metadata graph | Information flow hubs |
| `cross_desk_correlation` | Pearson correlation of order flow with other desks in same ISIN | Collusion signal |

#### 4.2.6 Unsupervised Deviation Signals (as features, not decision logic)
| Feature | Construction | Rationale |
|---------|-------------|-----------|
| `autoencoder_reconstruction_error` | L2 reconstruction error of 5-min order flow vector | Unusual pattern flag |
| `isolation_forest_score` | Anomaly score from per-desk isolation forest | Density-based rarity |
| `mahalanobis_distance` | Distance from trader's 6-month execution centroid | Multivariate deviation |
| `lstm_sequence_deviation` | Prediction error of next-order-type LSTM | Temporal pattern break |

### 4.3 Feature Store Requirements
- **Temporal correctness:** All features computed with lookback windows ending at `t_0` (alert generation time). Strict no-lookahead enforcement.
- **Point-in-time joins:** Entity attributes (desk, role) joined as-of `t_0`, not current values.
- **Versioning:** Feature definitions versioned. Model trained on v2.3 features cannot be served with v2.4 definitions without retraining.
- **Storage:** Redis for real-time features (< 5ms retrieval); Parquet/S3 for batch historical features.

---

## 5. Model Architecture: Three-Stage Stack

### 5.1 Stage 1: Violation Probability Estimator

**Objective:** Estimate $P(	ext{violation} \mid \mathbf{x})$ using Positive-Unlabeled (PU) learning.

**Base Model:** LightGBM (gradient boosting decision trees).

**Why LightGBM:**
- Native handling of missing values and categorical features.
- Fast inference (~2–5ms per prediction).
- Strong empirical performance on tabular PU learning benchmarks.
- SHAP-compatible for explainability.

**PU Learning Framework: nnPU (Non-Negative Unbiased Risk Estimator)**

The standard binary cross-entropy is biased when negatives are unlabeled. The nnPU risk estimator is:

$$\hat{R}(f) = \pi_p \hat{R}_p^+(f) + \max\left\{0, \hat{R}_u^-(f) - \pi_p \hat{R}_p^-(f)ight\}$$

Where:
- $\pi_p$ = prior probability of positive in the unlabeled set
- $\hat{R}_p^+$ = empirical positive-class risk on confirmed positives
- $\hat{R}_u^-$ = empirical negative-class risk on unlabeled alerts
- $\hat{R}_p^-$ = empirical negative-class risk on confirmed positives

**Training Procedure:**
1. **Estimate $\pi_p$** using the KM2 estimator:
   - Train a standard classifier to distinguish P from U (treating U as negative).
   - $\pi_p = rac{n_p}{n_p + n_u} \cdot rac{1}{	ext{mean predicted probability on U}}$.
2. **Initialize** LightGBM with focal loss (to handle remaining imbalance).
3. **Compute nnPU risk** on validation set after each boosting round.
4. **Early stopping** when validation nnPU risk plateaus for 50 rounds.
5. **Gradient clipping** at $\pm 5.0$ to prevent the `max(0, ...)` term from destabilizing.

**Alternative (Fallback):** Two-step PU method if nnPU is unstable:
- Step A: Train one-class SVM on confirmed positives to identify reliable negatives from U (samples with lowest similarity to P).
- Step B: Train LightGBM on P + reliable negatives with standard binary cross-entropy and `scale_pos_weight`.

**Output:** Raw score $s_v \in [0,1]$, calibrated via Platt scaling or isotonic regression on a holdout set of confirmed positives + reliable negatives.

**Hyperparameters (starting point):**
```
objective: binary
metric: custom (nnPU risk)
boosting_type: gbdt
num_leaves: 31
max_depth: 6
learning_rate: 0.05
feature_fraction: 0.8
bagging_fraction: 0.8
bagging_freq: 5
min_data_in_leaf: 50
lambda_l1: 0.1
lambda_l2: 1.0
early_stopping_rounds: 50
```

---

### 5.2 Stage 2: Conditional Severity Estimator

**Objective:** Estimate expected regulatory and market impact, conditioned on a violation having occurred.

**Base Model:** LightGBM Regressor (or Ordinal Classifier if fine data is sparse).

**Target Definition:**
$$y_i = \log(1 + 	ext{regulatory fine} + 	ext{estimated market impact})$$

If fine data is insufficient (< 500 confirmed positives with outcomes), use ordinal classification:
- Class 0: No action / warning letter
- Class 1: Fine < \$100K
- Class 2: Fine \$100K–\$1M
- Class 3: Fine > \$1M or criminal referral

**Training Data:**
- Train **only** on confirmed positives with known outcomes.
- If internal data is sparse, use **transfer learning**: pre-train on public enforcement actions (SEC, FCA, FINRA) with aligned feature schemas, then fine-tune on internal data.

**Feature Augmentation (beyond Stage 1):**
| Feature | Description |
|---------|-------------|
| `instrument_market_cap_log` | Log market capitalization |
| `benchmark_constituency_embedding` | S&P 500, EURO STOXX, FTSE, etc. |
| `settlement_window` | T+0, T+1, T+2 (embedded) |
| `media_attention_proxy` | News article count on issuer in prior 7 days |
| `historical_desk_fine_rate` | Rolling 2-year confirmed violation rate for desk |
| `regulatory_jurisdiction_embedding` | SEC, FCA, ECB, BaFin, etc. |
| `corporate_event_proximity` | Hours to next material event |

**Output:** Severity score $\hat{s}_i \in \mathbb{R}^+$. If ordinal classifier used, map to continuous score via expected value of class distribution.

**Regularization:** Heavy regularization required due to small sample size:
```
max_depth: 4
min_data_in_leaf: 100
lambda_l1: 1.0
lambda_l2: 5.0
feature_fraction: 0.6
```

---

### 5.3 Stage 3: Composite Risk Score & Tier Assignment

**Objective:** Combine violation probability and conditional severity into a single actionable risk score, then assign to one of three tiers using cost-optimal thresholds.

**Composite Score:**
$$R(\mathbf{x}) = \hat{p}_v(\mathbf{x})^{lpha} \cdot \hat{s}_i(\mathbf{x})^{1-lpha}$$

The geometric mean ensures both dimensions must be high for a top-tier alert. A high-severity alert with low violation probability is suppressed; a high-probability alert with low severity is deprioritized.

**Learning $lpha$:**

$lpha$ is learned via cost optimization on the validation set, not grid search on accuracy.

**Cost Model:**

| Outcome | Cost |
|---------|------|
| False Negative (violation in Tier 3) | $c_{miss} = 	ext{expected regulatory fine} + 	ext{reputational damage} + 	ext{remediation}$ |
| False Positive (NFA in Tier 1) | $c_{fp} = 	ext{senior investigator hours} + 	ext{trader friction} + 	ext{opportunity cost}$ |
| Delayed True Positive (violation in Tier 2 vs Tier 1) | $c_{delay} = f(	ext{settlement window}, 	ext{reporting deadline risk})$ |

**Expected Cost Function for tier assignment $T$:**
$$C(T \mid \hat{p}_v, \hat{s}_i) = \mathbb{1}_{T=1} \cdot c_{fp} + \mathbb{1}_{T \in \{2,3\}} \cdot \hat{p}_v \cdot \hat{s}_i \cdot c_{miss} + \mathbb{1}_{T=2} \cdot c_{delay}$$

**Optimization:**
1. Define a grid of $lpha \in [0.1, 0.9]$ in increments of 0.05.
2. For each $lpha$, compute $R(\mathbf{x})$ on the validation set.
3. Find thresholds $	au_1, 	au_2$ that minimize total expected cost.
4. Select $lpha^*$ yielding the lowest cost.
5. Expected range: $lpha^* pprox 0.65$–$0.75$.

**Tier Assignment Logic:**

**Tier 1 — Escalate (Immediate)**
- **Selection:** Top $K$ alerts by $R$, where $K$ = daily senior investigator capacity.
- **SLA:** < 4 hours to senior compliance officer.
- **Action:** Mandatory human investigation. Model provides SHAP-based narrative.
- **Capacity Constraint:** Tier 1 volume is **fixed** at $K$ alerts/day, regardless of total alert volume. On high-volume days, $	au_1$ floats upward automatically.

**Tier 2 — Standard (Investigate)**
- **Selection:** Alerts where $R \geq 	au_2$ but not in top $K$.
- **SLA:** < 48 hours to desk compliance analyst.
- **Action:** Standard investigation workflow. Model provides ranked list within tier.
- **Volume:** Cost-optimized. Typically ~20–25% of total alerts.

**Tier 3 — Automated (Close)**
- **Selection:** Alerts where $R < 	au_2$.
- **Action:** Auto-close with full feature vector, model version, and SHAP values logged. No human review.
- **Audit:** 2–3% random stratified sample forced to junior analyst review weekly to detect model drift and novel abuse patterns.
- **Volume:** Remaining ~70–75% of alerts.

**Threshold Dynamics:**
- $	au_1$ recomputed daily based on investigator capacity and expected alert volume.
- $	au_2$ recomputed weekly via cost optimization on rolling 90-day validation window.
- Emergency recalculation triggered if alert volume changes > 30% week-over-week.

---

## 6. Training Pipeline

### 6.1 Data Splits

**Strict temporal splits only.** Random splits are prohibited due to look-ahead risk.

| Split | Time Period | Purpose |
|-------|-------------|---------|
| Train | [T-12 months, T-3 months] | Model fitting |
| Validation | [T-3 months, T-1 month] | Hyperparameter tuning, $lpha$ optimization, threshold selection |
| Test | [T-1 month, present] | Final evaluation, shadow deployment |

**Leakage Guards:**
- No future information in features (e.g., rolling windows end at alert time).
- Corporate events joined as-of alert time, not announcement time.
- Trader desk assignments joined as-of alert time.

### 6.2 Label Handling

**Confirmed Positives (P):**
- Weighted by confirmation confidence (1.0 for regulatory fine, 0.7 for internal disciplinary, 0.5 for strong evidence but NFA due to settlement).
- Exponential time decay: $w_t = \exp(-0.5 \cdot 	ext{years since alert})$. Half-life ~1.4 years.

**Unlabeled Set (U):**
- All alerts not in P. Includes true negatives and hidden positives.
- **Never** treated as negative in Stage 1.

**Investigator Bias Correction:**
- If multiple investigator reviews exist per alert, use Dawid-Skene or majority voting to denoise labels.
- If certain desks historically receive rubber-stamp NFA closures, include desk ID as a feature in an initial biased model, then train a second model **without** desk ID. Compare and document disparate impact.

### 6.3 Class Imbalance

- **Focal Loss** in Stage 1: Down-weights easy negatives, focuses on hard positives.
- **Cost-proportionate rejection sampling:** Oversample confirmed positives during training, but calibrate probabilities post-hoc.
- **No SMOTE/ADASYN:** Synthetic sampling creates impossible financial time series patterns.

### 6.4 Retraining Cadence

| Frequency | Action |
|-----------|--------|
| **Weekly** | Ingest new alerts, backfill confirmed positives from closed investigations, recompute thresholds ($	au_1, 	au_2$). |
| **Monthly** | Full retrain of Stage 1 and Stage 2 on rolling 12-month window. A/B test against production model. |
| **Quarterly** | Architecture review, feature importance drift analysis, regulatory feedback integration. |
| **Event-driven** | Retrain within 48 hours if a regulatory miss is detected or a new abuse pattern is confirmed. |

### 6.5 A/B Testing Protocol

Before promoting a new model:
1. **Shadow mode:** New model scores all alerts but does not route them. Compare Precision@K vs. production model.
2. **Parallel queues:** For 2 weeks, send new model Tier 1 alerts to a parallel senior compliance queue. Measure investigation yield difference.
3. **Promotion criteria:** New model must achieve > 2% improvement in Precision@K on Tier 1 with no increase in regulatory miss rate.

---

## 7. Inference & Serving Architecture

### 7.1 Real-Time Pipeline

```
Alert Generated (t=0)
    │
    ▼
┌─────────────────────────────────────┐
│ Feature Store Lookup              │
│ - Real-time: < 5ms (Redis)        │
│ - Batch: < 20ms (S3/Parquet)     │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Stage 1: LightGBM (PU)            │
│ Inference: ~2ms                    │
│ Output: p_v (calibrated)          │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Stage 2: LightGBM (Severity)        │
│ Inference: ~2ms                    │
│ Output: s_i                        │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Stage 3: Composite Score + Tiering  │
│ Computation: < 1ms                 │
│ Output: Tier assignment             │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ SHAP Explanation Generation         │
│ Computation: ~5ms                  │
│ Output: Per-alert feature impact    │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│ Audit Log Write                     │
│ - Feature vector (JSON)             │
│ - Model version                     │
│ - SHAP values                       │
│ - Tier assignment + threshold values│
└─────────────────────────────────────┘
```

**End-to-end latency target:** < 50ms per alert.
**Throughput target:** 1,000 alerts/minute peak.

### 7.2 Batch Pipeline (Nightly)
- Recompute all batch features for active traders.
- Backfill any missing historical features.
- Generate training datasets for next retrain.

### 7.3 Explainability Output

For every Tier 1 and Tier 2 alert, the system generates:

1. **Global Context:** SHAP summary for the model version ("Top 3 features driving Tier 1 assignments this week: cancellation_rate_60s, historical_violation_rate, hours_to_next_event").
2. **Local Explanation:** Per-alert SHAP values showing push/pull of each feature.
3. **Counterfactual Narrative:** "To reduce this alert's risk score below the Tier 1 threshold, the trader's cancellation rate would need to decrease by X% and the execution timing relative to the news event would need to shift by Y hours."
4. **Model Version:** Exact version string of Stage 1, Stage 2, and threshold parameters used.

---

## 8. Tier Operationalization

### 8.1 Tier 1 — Escalate

| Attribute | Specification |
|-----------|--------------|
| **Selection** | Top $K$ alerts by $R$, capacity-constrained |
| **SLA** | < 4 hours to senior compliance officer |
| **Investigator** | Senior compliance officer (L6+) |
| **Workflow** | Mandatory investigation. Model provides SHAP narrative. Investigator must document conclusion in case management system. |
| **Escalation** | If investigation confirms violation, auto-generate regulatory filing prep (SAR, MRT, etc.). |
| **Capacity** | $K$ = 20 alerts/day (configurable). If alert volume < $K$, only top $N$ alerts are escalated. |

### 8.2 Tier 2 — Standard

| Attribute | Specification |
|-----------|--------------|
| **Selection** | $	au_1 > R \geq 	au_2$ |
| **SLA** | < 48 hours to desk compliance analyst |
| **Investigator** | Desk compliance analyst (L4–L5) |
| **Workflow** | Standard investigation. Ranked list within tier by $R$. |
| **Triage** | Analyst can escalate to Tier 1 if new evidence emerges. |

### 8.3 Tier 3 — Automated

| Attribute | Specification |
|-----------|--------------|
| **Selection** | $R < 	au_2$ |
| **Action** | Auto-close. No human review. |
| **Audit** | Full feature vector, model version, SHAP values, and threshold parameters persisted for 7 years. |
| **Random Audit** | 2% weekly sample forced to junior analyst review. Results feed back into label set. |
| **Dispute** | Traders can request manual review via compliance portal. Review is logged but does not trigger model retraining unless pattern emerges. |

---

## 9. Evaluation & Monitoring

### 9.1 Primary Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **Precision@K** | Of top $K$ alerts by $R$, % confirmed positives | > 25% (baseline: ~5% without prioritization) |
| **Investigation Yield** | Confirmed violations / total alerts in tier | Tier 1 > 5× Tier 2 > 10× Tier 3 |
| **Regulatory Miss Rate** | Violations detected by regulators/whistleblowers that system missed | < 1 per year |
| **Mean Time to Detection (MTTD)** | Hours from trade execution to Tier 1 alert | < 4 hours |
| **Tier 3 Audit Yield** | Confirmed violations found in Tier 3 random audit | < 2% of audited alerts |

### 9.2 Model Monitoring

| Check | Frequency | Threshold | Action |
|-------|-----------|-----------|--------|
| **PSI (Population Stability Index)** | Daily | PSI > 0.25 on $R$ distribution | Trigger retraining |
| **Feature Drift** | Daily | Any feature PSI > 0.2 | Alert data engineering |
| **SHAP Drift** | Weekly | Top 5 feature importances shift > 20% | Architecture review |
| **Latency** | Real-time | p99 > 50ms | Page on-call engineer |
| **Throughput** | Real-time | Queue depth > 100 alerts | Scale inference replicas |
| **Label Flow** | Weekly | < 5 new confirmed positives / week | Investigate data pipeline |

### 9.3 Regulatory Miss Protocol

If a violation is detected by a regulator or whistleblower that the system missed:
1. **Immediate:** Extract full feature history for the missed case.
2. **Within 24 hours:** Root cause analysis — was it a model error, feature missingness, or threshold misalignment?
3. **Within 48 hours:** Emergency retrain if root cause is model-related.
4. **Within 1 week:** Report to Model Risk Committee with remediation plan.
5. **Quarterly:** Aggregate miss analysis presented to CRO and CCO.

---

## 10. Governance, Risk & Compliance (GRC)

### 10.1 Model Risk Management

This system is classified as a **high-risk model** under SR 11-7 (US) / SS1/13 (UK) / EU AI Act (High-Risk AI System).

**Required Documentation:**
- Model Concept Paper (this document)
- Model Development Document (MDD)
- Model Validation Report (independent validation)
- Ongoing Monitoring Report (quarterly)
- Model Inventory Entry (permanent)

**Independent Validation:**
- Must be performed by a team with no reporting line to Compliance Technology.
- Validates: data quality, methodology appropriateness, code review, stress testing, bias audit.

### 10.2 Bias & Fairness

**Disparate Impact Audit (Quarterly):**
- Test whether Tier 1 rate differs significantly across:
  - Trading desks (equities vs. fixed income vs. FX)
  - Geographic regions
  - Instrument types (small-cap vs. large-cap)
  - Trader demographics (tenure, role level)
- **Metric:** Adverse impact ratio (Tier 1 rate for group A / Tier 1 rate for group B). Flag if < 0.8 for any pair.
- **Remediation:** If bias detected, add fairness constraints to Stage 1 or remove proxy features (e.g., desk ID).

### 10.3 Audit Trail

Every tier assignment decision must be reproducible for 7 years:
- Alert ID and timestamp
- Complete feature vector (JSON)
- Model version strings for Stage 1, Stage 2, and composite logic
- Threshold values ($	au_1, 	au_2, lpha$) at time of assignment
- SHAP values for top 10 features
- Investigator outcome (if applicable)
- Regulatory action (if applicable)

### 10.4 Human-in-the-Loop Requirements

- **Tier 1:** Model recommends; human decides. No auto-escalation to regulators.
- **Tier 2:** Model recommends; human investigates. Human can override tier assignment with documented rationale.
- **Tier 3:** Model decides; human audits sample. No individual human review for closure.
- **Override Logging:** All human overrides are logged and analyzed monthly for model improvement.

### 10.5 Regulatory Reporting Interface

The system must expose APIs for:
- **SEC:** Real-time alert data for MRT (Market Abuse Regulatory Team) inquiries.
- **FCA:** Bulk export of Tier 1 alert histories for suspicious transaction reporting.
- **Internal Audit:** Read-only access to all audit logs, model versions, and SHAP explanations.

---

## 11. Implementation Roadmap

### Phase 1: Foundation (Weeks 1–8)
**Objective:** Build data infrastructure and baseline model.

| Week | Deliverable |
|------|-------------|
| 1–2 | Feature store architecture design and data source integration |
| 3–4 | Feature engineering pipeline (batch + real-time) |
| 5–6 | Baseline Stage 1 model (two-step PU method) on 12-month historical data |
| 7–8 | Validation against confirmed positive set; baseline Precision@K measurement |

**Milestone:** Feature store operational. Baseline model achieves Precision@K > 15% on historical data.

### Phase 2: Model Development (Weeks 9–16)
**Objective:** Build full 3-stage stack and optimize tiers.

| Week | Deliverable |
|------|-------------|
| 9–10 | Stage 1 nnPU implementation and hyperparameter tuning |
| 11–12 | Stage 2 severity model (transfer learning from public enforcement data if needed) |
| 13–14 | Stage 3 composite score and cost-optimal threshold optimization |
| 15–16 | End-to-end pipeline integration; SHAP explainability layer |

**Milestone:** Full 3-stage model operational in shadow mode. Precision@K > 20% on validation set.

### Phase 3: Operational Integration (Weeks 17–24)
**Objective:** Deploy with human oversight and measure live performance.

| Week | Deliverable |
|------|-------------|
| 17–18 | Shadow deployment: model scores all alerts but does not route. Parallel investigation of model Tier 1 vs. existing rule-based Tier 1. |
| 19–20 | A/B test analysis; model refinement based on live data |
| 21–22 | Graduated rollout: auto-route Tier 3 only; human oversight on Tier 1/2 |
| 23–24 | Full production deployment; investigator training on SHAP narratives |

**Milestone:** Live system routing 100% of alerts. Tier 3 auto-closure active. Investigator yield measured weekly.

### Phase 4: Optimization & Governance (Weeks 25–36)
**Objective:** Harden monitoring, governance, and continuous improvement.

| Week | Deliverable |
|------|-------------|
| 25–28 | Bias audit; disparate impact remediation; model risk documentation |
| 29–32 | Independent validation; regulatory pre-notification |
| 33–36 | Automated retraining pipeline; drift monitoring dashboard; quarterly review cycle |

**Milestone:** Model Risk Committee sign-off. System enters steady-state operations.

---

## 12. Appendices

### Appendix A: Glossary

| Term | Definition |
|------|-----------|
| **MAD** | Market Abuse Detection |
| **PU Learning** | Positive-Unlabeled learning; training with confirmed positives and unlabeled samples |
| **nnPU** | Non-negative unbiased risk estimator for PU learning |
| **SHAP** | SHapley Additive exPlanations; model-agnostic feature attribution |
| **PSI** | Population Stability Index; measures distribution drift |
| **NFA** | No Further Action; investigator closure without violation |
| **HHI** | Herfindahl-Hirschman Index; concentration measure |
| **MRT** | Market Abuse Regulatory Team (SEC) |
| **SAR** | Suspicious Activity Report |

### Appendix B: Data Retention & Privacy

- **Alert data:** 7 years (regulatory requirement).
- **Feature vectors:** 7 years, encrypted at rest.
- **SHAP explanations:** 7 years, linked to alert ID.
- **Trader PII:** Minimized in model features. Tenure and role are used; name, age, gender, ethnicity are **excluded** from all model inputs.
- **Communication metadata:** Aggregated graph centrality only; no message content or recipient identities in features.

### Appendix C: Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| PU learning instability (nnPU divergence) | Medium | High | Fallback to two-step method; gradient clipping; validation monitoring |
| Severe model drift (market regime change) | Low | Critical | Weekly PSI monitoring; emergency retraining protocol; human override capability |
| Regulatory miss due to Tier 3 auto-closure | Low | Critical | 2% random audit; quarterly regulatory miss analysis; feedback loop to labels |
| Investigator resistance to ML recommendations | Medium | Medium | SHAP narratives; override logging; monthly yield reporting to demonstrate value |
| Feature pipeline failure (stale data) | Medium | High | Real-time freshness checks; circuit breakers to rule-based fallback if features > 1 hour stale |
| Bias audit failure | Low | High | Quarterly disparate impact testing; fairness constraints; proxy feature removal |

### Appendix D: Fallback Procedures

**If model inference fails or latency exceeds 50ms:**
1. Circuit breaker activates after 3 consecutive failures.
2. System falls back to **upstream anomaly score percentile** as temporary prioritization.
3. All alerts default to Tier 2 (no auto-closure) until model recovers.
4. On-call engineer paged automatically.
5. Post-incident review within 24 hours.

**If feature store is unavailable:**
1. Use last-known feature values if < 1 hour old.
2. If > 1 hour old, reject alert scoring and queue for batch reprocessing.
3. Upstream MAD engine continues to generate alerts; they are held in queue.

---

**Document Control:**
- **Author:** Compliance Data Science
- **Reviewer:** Model Risk Management, Legal, Compliance Operations
- **Approver:** Chief Compliance Officer, Chief Risk Officer
- **Next Review Date:** 2026-09-27

**END OF SPECIFICATION**
