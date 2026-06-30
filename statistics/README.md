# Statistics — For ML/DS Interviews

**Phase:** 1 (Foundations) → 3 (Differentiator)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Sister tracks:** [AI Engineering](../ai/) · [Data Engineering](../data-engineer/) · [Python](../python/)

---

## Why This Section Exists

DS/ML interviews — especially at quant shops, FAANG DS roles, McKinsey/QuantumBlack, and any "we make decisions from data" team — test statistics. Not measure-theoretic statistics; **practical inferential statistics** plus a clear grasp of what a p-value actually means. Most candidates parrot definitions ("p-value is the probability under the null") and stop. The candidates who get offers can:

1. Compute test statistics by hand for small problems.
2. Explain *why* each assumption matters (normality, independence, equal variance).
3. Pick the right test for the data type (continuous vs discrete, paired vs independent, parametric vs non-parametric).
4. Run a power analysis to size an A/B test.
5. Read a regression output (coefficients, p-values, R²) and call out what's wrong.

This folder gives you that, with the math explicit.

---

## Topic Index

### Tier 0 — Foundation (mandatory for everything else)

1. [**Probability Foundations**](./01-probability-foundations.md) — random variables, expectation, variance, conditional probability, Bayes' theorem, CLT, LLN, common distributions (Normal, Binomial, Poisson, t, χ², F).
2. [**Descriptive Statistics**](./02-descriptive-stats.md) — mean/median/mode, variance/std, skew/kurtosis, percentiles, correlation vs covariance, **z-score / standardization**.

### Tier 1 — Inferential Statistics (the most-tested cluster)

3. [**Hypothesis Testing — the Complete Toolkit**](./03-hypothesis-testing.md) — null vs alternative, Type I/II errors, **p-value rigorously defined**, power, confidence intervals, **z-test, one/two-sample/paired t-test, χ² test, ANOVA, Mann-Whitney**, multiple comparisons.
4. [**A/B Testing & Experimentation**](./04-ab-testing.md) — sample-size / power calculations, multi-arm, sequential testing, Bonferroni vs FDR, peeking problem, novelty / network effects.

### Tier 2 — Modeling

5. [**Regression Fundamentals**](./05-regression.md) — OLS derivation, assumptions, R²/adjusted R², coefficient p-values, multicollinearity/VIF, heteroskedasticity, logistic regression, ridge/lasso/elastic-net.
6. [**ML Metrics & Evaluation**](./06-ml-metrics.md) — confusion matrix, precision/recall/F1, AUC-ROC, log loss, calibration, MSE/MAE/RMSE, bias-variance tradeoff, cross-validation, class imbalance.

### Tier 3 — Differentiators

7. [**Bayesian Stats & Causal Inference**](./07-bayesian-and-causal.md) — prior/likelihood/posterior, MAP vs MLE, conjugate priors, confounders, Simpson's paradox, DAGs, propensity scores, diff-in-diff.

---

## How Each File Is Structured

Every file follows the same template:

1. **Concept first** — what the thing is in plain English.
2. **The math** — formulas explicit, derivations sketched where they illuminate.
3. **Python implementation** — `scipy.stats` calls so you know the real-world tool.
4. **When to use it / when it breaks** — assumptions, edge cases, common misuses.
5. **Where it shows up in ML/DS interviews** — concrete questions and how to answer.
6. **Cheat sheet** — Q&A format for last-night review.

---

## The 10 Statistics Q&As You Must Be Able to Answer

These come up in essentially every DS/ML interview. If any is fuzzy, drill it.

1. What does a p-value actually mean? (And what does it *not* mean?)
2. What's the difference between Type I and Type II error?
3. What's the Central Limit Theorem and why does it matter?
4. When do you use a t-test vs a z-test?
5. How would you design an A/B test for a new feature? How big does the sample need to be?
6. What's the bias-variance tradeoff?
7. How do you handle multiple comparisons?
8. What assumptions does linear regression make? How do you check them?
9. Why is AUC-ROC better than accuracy on imbalanced data?
10. Frequentist vs Bayesian — when would you use each?

Each is answered in detail somewhere in this folder.

---

## Suggested Reading Order

**1 week (interview tomorrow):** files 3, 6, 4 — hypothesis testing, ML metrics, A/B testing.

**1 month:** all seven files in order.

**3 months:** the above plus working through problems from *All of Statistics* (Wasserman) or *Statistical Inference* (Casella & Berger).

---

## Foundational References

- *All of Statistics* — Larry Wasserman. Best modern compact intro.
- *Statistical Inference* — Casella & Berger. The PhD-prep classic.
- *Statistical Rethinking* — McElreath. Best Bayesian intro by far.
- *The Elements of Statistical Learning* — Hastie, Tibshirani, Friedman. ML-statistics bridge. [Free PDF.](https://hastie.su.domains/ElemStatLearn/)
- *Trustworthy Online Controlled Experiments* — Kohavi, Tang, Xu. The A/B testing book.
- *Mostly Harmless Econometrics* — Angrist & Pischke. Causal inference.
- [scipy.stats docs](https://docs.scipy.org/doc/scipy/reference/stats.html) — the Python reference.
- [Seeing Theory (Brown)](https://seeing-theory.brown.edu/) — beautiful visual intuition.
- [StatQuest YouTube](https://www.youtube.com/c/joshstarmer) — friendly explainers.

---

*Start with [01-probability-foundations.md](./01-probability-foundations.md). The CLT and Bayes' theorem cited everywhere else live there.*
