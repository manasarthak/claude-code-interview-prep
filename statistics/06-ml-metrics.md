# ML Metrics & Evaluation — Confusion Matrix, AUC, Bias-Variance, CV

**Phase:** 2 (Modeling)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Hypothesis Testing](./03-hypothesis-testing.md) · [Regression](./05-regression.md) · [Descriptive Stats](./02-descriptive-stats.md)

---

## Why This File Exists

Picking the right metric is half of ML in practice. Accuracy is wrong half the time you reach for it (imbalanced classes). Cross-validation has dozens of flavors and using the wrong one leaks data silently. This file gives you the metrics, when each is the right choice, and the canonical interview pitfalls.

---

## BEGINNER — The Confusion Matrix and Friends

### The 2×2 Confusion Matrix

For binary classification:

|  | Predicted 1 | Predicted 0 |
|---|---|---|
| **Actual 1** | TP | FN |
| **Actual 0** | FP | TN |

Memorize:

- **TP** (true positive) — correctly predicted 1.
- **TN** (true negative) — correctly predicted 0.
- **FP** (false positive / Type I error) — predicted 1 but actually 0.
- **FN** (false negative / Type II error) — predicted 0 but actually 1.

### The Core Rates

$$\text{Accuracy} = \frac{TP + TN}{TP + TN + FP + FN}$$

$$\text{Precision} = \frac{TP}{TP + FP} \quad \text{(of those predicted 1, how many were really 1?)}$$

$$\text{Recall (Sensitivity, TPR)} = \frac{TP}{TP + FN} \quad \text{(of those actually 1, how many did we catch?)}$$

$$\text{Specificity (TNR)} = \frac{TN}{TN + FP} \quad \text{(of those actually 0, how many did we correctly clear?)}$$

$$\text{F1} = 2 \cdot \frac{\text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}} \quad \text{(harmonic mean — punishes imbalance)}$$

$$\text{False Positive Rate (FPR)} = \frac{FP}{FP + TN} = 1 - \text{Specificity}$$

### Why Accuracy Is Often Wrong

Cancer screening: 1% of patients have cancer. A model that always predicts "no cancer" has **99% accuracy** and zero clinical value.

When **classes are imbalanced** or **misclassification costs differ asymmetrically**, accuracy is misleading. Use precision, recall, F1, or business-cost-weighted metrics instead.

### Precision vs Recall — The Trade-off

Move the decision threshold (default 0.5 for probabilistic classifiers):

- **Lower threshold** → predict 1 more often → ↑ recall, ↓ precision.
- **Higher threshold** → predict 1 less often → ↑ precision, ↓ recall.

**Which to prioritize depends on cost asymmetry:**

| Use case | Care more about | Why |
|---|---|---|
| Cancer screening | **Recall** | Missing a real case is much worse than a false alarm |
| Spam filter (default) | **Precision** | Inbox-as-spam is much worse than a spam-in-inbox |
| Search engine | **Precision (top-K)** | First page must be relevant |
| Fraud detection | **Recall, then precision** | Catch all fraud; allow some review-team load |
| Recommendation | **Precision (top-K)** + diversity | Quality over coverage |
| Content moderation | **Recall** with manual review | Compliance — better to over-flag |

The F1 score balances both. **F_β** generalizes — $F_\beta = (1+\beta^2) \cdot \frac{P \cdot R}{\beta^2 P + R}$ — with β > 1 weighting recall more, β < 1 weighting precision more. F2 is common in fraud / safety.

---

## INTERMEDIATE — ROC, PR Curves, AUC

### ROC Curve

Plot TPR (recall) vs FPR for every possible threshold. Each point is a (precision, recall) trade-off at that threshold.

- The diagonal (TPR = FPR) is random guessing.
- Top-left corner is perfect (TPR = 1, FPR = 0).
- A curve "above" another dominates it.

### AUC-ROC (Area Under ROC Curve)

A single number summarizing the ROC curve. Interpretation:

> **AUC = P(model scores a random positive higher than a random negative).**

- AUC = 0.5 — random.
- AUC = 1.0 — perfect.
- AUC < 0.5 — your labels are flipped; invert the model.

AUC is **threshold-independent** — measures the model's ranking ability across all thresholds. That's why it's robust to class imbalance compared to accuracy.

### Precision-Recall Curve and AUC-PR

For heavily imbalanced classes (e.g. 0.1% positive), **AUC-PR is often more informative than AUC-ROC** because ROC averages over the FPR axis, which is dominated by the abundant negatives — small movements there look "good" but are meaningless.

- Random baseline for AUC-PR = base rate (positive fraction), not 0.5.
- Use AUC-PR when positives are rare and you care about ranking them.

### When to Prefer ROC vs PR

| Setting | Prefer |
|---|---|
| Balanced classes | AUC-ROC |
| Imbalanced classes, care about positives | **AUC-PR** |
| Care about both errors symmetrically | F1 at optimal threshold |
| Need to pick an operating threshold | Look at precision/recall curve directly |

### Log Loss (Cross-Entropy)

$$\text{LogLoss} = -\frac{1}{n} \sum_i \left[y_i \log p_i + (1 - y_i) \log(1 - p_i)\right]$$

Penalizes confident-and-wrong predictions hardest. Used as both an evaluation metric and the **loss function** for logistic regression and most classification neural nets.

Properties:
- Range: [0, ∞). Lower is better.
- Calibrated probabilities → low log loss.
- A single overconfident wrong prediction (p = 0.99, y = 0) sends log loss toward infinity. So log loss rewards models that **know what they don't know** (output probabilities near 0.5 when unsure).

### Brier Score

Mean squared error of probabilistic predictions: $\frac{1}{n}\sum (p_i - y_i)^2$. Less harsh than log loss on confident-wrong; useful when calibration matters.

### Calibration

A classifier is **calibrated** if, of all examples it predicts with probability p, the actual rate of positives is p. **Reliability diagram**: bin predictions, plot mean predicted probability vs mean actual outcome per bin.

Random forests and SVMs are usually poorly calibrated; logistic regression is naturally well-calibrated; tree-boosted models (XGBoost) are often reasonably calibrated.

Fixes: **Platt scaling** (sigmoid fit) or **isotonic regression** on a held-out set.

---

## INTERMEDIATE — Regression Metrics

### MSE / RMSE / MAE

$$\text{MSE} = \frac{1}{n}\sum (y_i - \hat{y}_i)^2, \quad \text{RMSE} = \sqrt{\text{MSE}}, \quad \text{MAE} = \frac{1}{n}\sum |y_i - \hat{y}_i|$$

- **MSE / RMSE** — penalize large errors heavily (quadratic). Sensitive to outliers. Differentiable everywhere → easier to optimize. Standard for most regression.
- **MAE** — robust to outliers. Median-friendly (MAE-optimal predictor is the median; MSE-optimal is the mean).
- **RMSE has same units as y** — more interpretable than MSE.

### MAPE and Friends

$$\text{MAPE} = \frac{100\%}{n}\sum_i \left|\frac{y_i - \hat{y}_i}{y_i}\right|$$

Percentage error. Useful when relative error matters more than absolute. **Breaks when y is near zero** (division explodes). Variants like **sMAPE** mitigate this.

### R² for Out-of-Sample

CV R² or test-set R² is the right "explained variance" measure for predictive models. Training R² is meaningless for model selection — always use a held-out estimate.

### Quantile Loss / Pinball Loss

For predicting quantiles (e.g. P95 latency, P10 risk): minimizes asymmetric loss that recovers the τ-quantile. Foundation of quantile regression. Used heavily in forecasting and risk modeling.

---

## INTERMEDIATE — Cross-Validation Done Right

### k-Fold Cross-Validation

Split data into k folds. For each i, train on the other k-1 folds, evaluate on fold i. Average the metrics.

- k = 5 or k = 10 are conventional.
- Higher k → more bias-free estimate but more compute.
- k = n is **leave-one-out CV (LOOCV)** — minimal bias, max variance.

### Stratified k-Fold

For classification with imbalanced classes, ensure each fold preserves the class distribution. Use `StratifiedKFold` in scikit-learn.

### Group / Block CV

If your data has natural groups (users, sessions), random folding can leak: same-user data shows up in both train and test. Use **GroupKFold** (each user appears in one fold only).

### Time-Series CV (the most-failed interview gotcha)

**Never use random k-fold on time-series data.** It leaks future into past. Use:

- **Expanding window** — train on [1..t], test on t+1. Roll forward.
- **Sliding window** — fixed-length window of training data, rolls forward.

```python
from sklearn.model_selection import TimeSeriesSplit
tscv = TimeSeriesSplit(n_splits=5)
```

### Nested CV — When You Tune Hyperparameters

If you both tune hyperparameters via CV and evaluate via CV, **use nested CV**: outer loop for evaluation, inner loop for tuning. Otherwise your reported metric is optimistically biased (you've leaked hyperparameter information).

### Train/Validation/Test Split

The simpler workflow:

- **Train** (60–70%) — fit the model.
- **Validation** (15–20%) — tune hyperparameters, model selection.
- **Test** (15–20%) — final evaluation, looked at ONCE.

Touching the test set during development invalidates its purpose. The test set is your *unbiased* measurement; touching it makes it biased.

### Common Leakage Sources

1. **Target leakage** — a feature that's actually a proxy for y (e.g. "did the loan default" used as a feature to predict default).
2. **Train-test contamination** — fitting normalization, encoders, imputers on the full dataset before split.
3. **Group leakage** — same user in train and test.
4. **Temporal leakage** — using future to predict past.
5. **Label leakage** in features derived from labels (e.g. one-hot encoding fit on train+test).

`scikit-learn`'s `Pipeline` exists to prevent #2: fit transformers and the model on train fold only, transform on test fold.

---

## ADVANCED — Bias-Variance Trade-off

### Decomposition

For squared-error loss:

$$E[(y - \hat{f}(x))^2] = \underbrace{\text{Bias}[\hat{f}(x)]^2}_{\text{model wrong}} + \underbrace{\text{Var}[\hat{f}(x)]}_{\text{model unstable}} + \underbrace{\sigma^2}_{\text{irreducible}}$$

- **Bias** — error from oversimplified model. High bias → underfitting.
- **Variance** — error from sensitivity to training data. High variance → overfitting.
- **Irreducible noise** — σ², the variance of the noise; can't be reduced.

### Model Complexity Curve

As complexity ↑:
- Training error → monotonically down.
- Test error → first down (less bias), then up (more variance) — U-shape.
- The sweet spot is "just enough" complexity.

### Knobs to Tune

| Direction | Levers |
|---|---|
| **↑ bias, ↓ variance** | Smaller model, more regularization, fewer features, more bias-inducing prior |
| **↓ bias, ↑ variance** | Bigger model, less regularization, more features, more data (variance only) |
| **Always good** | More training data (decreases variance, not bias) |

### Ensembles — The Variance-Killer

- **Bagging** (Random Forest) — train many models on bootstrap samples, average. Reduces variance, keeps bias roughly the same.
- **Boosting** (XGBoost, LightGBM, GBM) — sequentially fit residuals. Reduces bias, can overfit if too many trees.
- **Stacking** — combine outputs of multiple models with a meta-learner.

### "Double Descent" (modern wrinkle)

For very over-parameterized models (modern neural nets), the test error can decrease *again* past the interpolation threshold. So the classical U-shape is not the full picture. Active research; worth name-dropping for senior ML interviews.

---

## ADVANCED — Class Imbalance and Cost-Sensitive Learning

### Three Levels of Imbalance

| Imbalance | What works |
|---|---|
| Mild (90/10) | Use AUC-PR, F1; standard methods often fine |
| Moderate (99/1) | Class weights, threshold tuning, AUC-PR |
| Severe (99.9/0.1) | Resampling (SMOTE, undersample), anomaly detection framing, cost-sensitive |

### Techniques

- **Class weighting** — penalize errors on the minority class more heavily. `sklearn`'s `class_weight='balanced'`.
- **Resampling**:
  - **Oversample** the minority (SMOTE: synthetic minority oversampling).
  - **Undersample** the majority. Loses information but fast.
- **Threshold tuning** — most classifiers default to 0.5; tune via precision-recall curve to your business cost.
- **Anomaly-detection framing** — when positives are *extremely* rare (<0.1%), reframe as outlier detection (Isolation Forest, autoencoder reconstruction error).

### The Cost-Sensitive Objective

If FN costs $1000 and FP costs $10, the optimal decision threshold isn't 0.5. Use expected-cost minimization:

$$\text{Predict 1 iff } P(y=1 \mid x) \cdot C_{FN} > P(y=0 \mid x) \cdot C_{FP}$$

Equivalently, lower the threshold to favor "predict 1" when FN is expensive.

---

## ADVANCED — Other Useful Metrics

### Multiclass

- **Macro F1** — average F1 across classes (treats all classes equally).
- **Weighted F1** — average F1 weighted by class frequency.
- **Micro F1** — pool TP/FP/FN across classes, then compute F1 (equals accuracy in single-label classification).

Use **macro** when minority classes matter as much as majority; **micro/weighted** when class frequency reflects business importance.

### Ranking / Recommendation

- **NDCG** (Normalized Discounted Cumulative Gain) — top-of-list relevance weighted; standard for search ranking.
- **MAP** (Mean Average Precision) — top-K precision averaged across queries.
- **MRR** (Mean Reciprocal Rank) — 1 / rank of the first relevant item.
- **Recall@K, Precision@K** — at a fixed cutoff K (the page-1 paradigm).

### LLM / Text Generation

- **BLEU, ROUGE, METEOR** — n-gram overlap with reference. Weak proxies for actual quality.
- **BERTScore** — embedding-based semantic similarity.
- **LLM-as-judge** — use a strong LLM (GPT-4, Claude) to rate outputs. Now standard for [RAG eval](../ai/rag/01-rag-fundamentals.md).
- **Human eval** — gold standard; expensive.

For RAG specifically: faithfulness, context relevance, answer relevance, answer correctness — RAGAS framework. See [RAG Fundamentals](../ai/rag/01-rag-fundamentals.md).

### Object Detection

- **mAP@IoU** — mean average precision over IoU thresholds. Standard COCO metric.

---

## Common Interview Gotchas

**Q: Why is accuracy a bad metric for imbalanced data?**
A: A trivial "always predict majority" classifier gets high accuracy without learning. Use precision, recall, F1, AUC-PR.

**Q: When would you prefer recall over precision?**
A: When missing positives is much costlier than raising false alarms — cancer screening, fraud, security.

**Q: AUC-ROC = 0.5 means what?**
A: Random ranking — model has no discriminative ability. AUC = 0 means perfectly anti-correlated; flip the labels.

**Q: Why prefer AUC-PR over AUC-ROC for imbalanced data?**
A: ROC averages over FPR, dominated by abundant negatives — small absolute changes look good. PR focuses on positives, where the action is.

**Q: What's the difference between F1 and arithmetic mean of P and R?**
A: F1 is the harmonic mean, which is pulled down hard by the smaller of the two. A model with P=0.9, R=0.1 has arithmetic mean 0.5 but F1 = 0.18.

**Q: What's log loss?**
A: Negative log likelihood of the predicted probabilities. Punishes confident-wrong predictions exponentially. Standard for probability-output classifiers.

**Q: What's the bias-variance trade-off?**
A: Total error = bias² + variance + irreducible. More complex model → less bias, more variance. Sweet spot minimizes test error.

**Q: How do you do cross-validation on time-series data?**
A: Never random k-fold (leaks future). Use expanding-window or sliding-window splits.

**Q: When would CV give an optimistic estimate?**
A: When you tune hyperparameters and report the best CV score — you've effectively selected the noise. Use nested CV, or a held-out test set looked at once.

**Q: How do you handle class imbalance?**
A: Class weights, SMOTE / oversampling, undersampling, threshold tuning via precision-recall curve, or anomaly-detection framing for severe imbalance.

**Q: What's calibration?**
A: A classifier is calibrated if, among examples with predicted probability p, the actual positive rate is p. Logistic regression is naturally calibrated; tree models often aren't.

**Q: How do you compare two models?**
A: Same CV folds + paired statistical test on the per-fold scores (paired t-test or Wilcoxon signed-rank). Don't compare raw averages.

---

## Interview-Ready Cheat Sheet

### Quick metric picker

| Problem | Metric |
|---|---|
| Balanced classification | Accuracy, F1, AUC-ROC |
| Imbalanced classification | AUC-PR, F1 (or F2), recall at fixed precision |
| Probability output | Log loss, Brier score, calibration |
| Asymmetric cost | Expected cost, F_β |
| Regression | RMSE (sensitive), MAE (robust), R² (variance explained) |
| Ranking | NDCG, MAP, MRR, Recall@K |
| Multi-class | Macro F1 (fair) vs Weighted F1 (frequency-weighted) |
| LLM generation | BERTScore, RAGAS, LLM-as-judge, human eval |

### CV picker

| Setting | Use |
|---|---|
| i.i.d. data | Random k-fold |
| Imbalanced classification | Stratified k-fold |
| Grouped data (users) | GroupKFold |
| Time series | TimeSeriesSplit (expanding window) |
| Hyperparameter tuning | Nested CV |
| Final unbiased estimate | Held-out test set |

### Bias-variance levers

- **Reduce bias:** bigger model, fewer regularization, more features, boosting.
- **Reduce variance:** more data, regularization, bagging/random forest, simpler model, dropout (NN).
- **Always helpful:** more (representative) training data.

### Trade-off pairs

- **Precision vs recall:** threshold trade-off; pick based on cost.
- **AUC-ROC vs AUC-PR:** balanced vs imbalanced; ranking vs positive-focused.
- **Accuracy vs F1:** symmetric vs asymmetric importance.
- **Bias vs variance:** simpler vs more flexible model.
- **Training error vs CV error:** in-sample fit vs generalization.
- **MAE vs RMSE:** robust vs penalizes-large.

---

## Resources & Links

- *Elements of Statistical Learning* (Hastie et al.) — ch. 7.
- *Hands-On Machine Learning* (Géron) — practical metric + CV walkthroughs.
- [scikit-learn metrics docs](https://scikit-learn.org/stable/modules/model_evaluation.html)
- [Google ML Crash Course — Classification](https://developers.google.com/machine-learning/crash-course/classification/video-lecture)
- [Kaggle — Model Validation](https://www.kaggle.com/learn/intro-to-machine-learning)
- [Double descent paper (Belkin et al., 2019)](https://arxiv.org/abs/1812.11118)
- [RAGAS docs](https://docs.ragas.io/) — for LLM evaluation.

### Companion files
- [Regression](./05-regression.md) — the models being scored.
- [Hypothesis Testing](./03-hypothesis-testing.md) — paired tests for model comparison.
- [A/B Testing](./04-ab-testing.md) — picking metrics in product experimentation.
- [AI: RAG Fundamentals](../ai/rag/01-rag-fundamentals.md) — LLM/RAG-specific metrics.

---

*Next: [Bayesian & Causal](./07-bayesian-and-causal.md) — the Tier-3 differentiator.*
