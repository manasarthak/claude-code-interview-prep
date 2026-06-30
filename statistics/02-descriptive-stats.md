# Descriptive Statistics — Measures of Center, Spread, Shape, and the Z-Score

**Phase:** 0 (Foundation)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Probability Foundations](./01-probability-foundations.md) · [Hypothesis Testing](./03-hypothesis-testing.md) · [ML Metrics](./06-ml-metrics.md)

---

## Why This File Exists

Descriptive statistics summarize data. The **z-score** is the bridge from descriptive stats to inferential stats — every t-test, z-test, regression coefficient test, and standardization step in ML uses it. Beyond z-scores, the choice between mean and median, variance and IQR, all matters because real data is messy (outliers, skew, missing).

---

## BEGINNER — Measures of Center

### Mean (arithmetic average)

$$\bar{x} = \frac{1}{n} \sum_{i=1}^n x_i$$

The sum divided by count. The most-used measure, but **sensitive to outliers** — one extreme value can move it arbitrarily far.

### Median

The middle value when data is sorted. For odd n, the (n+1)/2-th value; for even n, the average of the two middle values.

- **Robust to outliers** — moving a single extreme value doesn't change it.
- Use for skewed data (income, response times, file sizes).
- Equivalent to the 50th percentile.

### Mode

The most-frequent value. Mostly used for categorical data; on continuous data it's defined via the peak of a density estimate.

### Mean vs Median — When to Use Each

| Data shape | Use |
|---|---|
| Symmetric, no outliers | Mean (more efficient) |
| Skewed (income, latency) | Median |
| Reported by management to mislead | Mean of right-skewed |
| Reported honestly | Both, plus skew/IQR |

**Rule:** if you can only report one number, report the median. The mean lies about typical experience whenever the distribution has a long tail.

### Geometric and Harmonic Means (worth knowing)

- **Geometric mean** = (x₁ · x₂ · ... · xₙ)^(1/n). For multiplicative data: returns, growth rates. CAGR is the geometric mean of yearly growth factors.
- **Harmonic mean** = n / Σ(1/xᵢ). For rates: average speed when distance is fixed. The **F1 score** in ML is the harmonic mean of precision and recall — that's why it punishes imbalanced precision/recall harder than the arithmetic mean would.

---

## BEGINNER — Measures of Spread

### Range, IQR

- **Range** = max - min. Trivially outlier-sensitive; rarely used alone.
- **Interquartile range (IQR)** = Q₃ - Q₁ (75th percentile minus 25th). Robust to outliers; basis of the box plot.

### Variance and Standard Deviation

For a sample of size n with mean $\bar{x}$:

$$s^2 = \frac{1}{n-1} \sum_{i=1}^n (x_i - \bar{x})^2 \qquad s = \sqrt{s^2}$$

**Why n − 1, not n?** Bessel's correction. Using n underestimates the true variance (because $\bar{x}$ is itself estimated from the data — it "uses up" one degree of freedom). Dividing by n − 1 makes s² an *unbiased* estimator of the population variance σ².

For a population of size N (you have everyone), use:

$$\sigma^2 = \frac{1}{N} \sum_{i=1}^N (x_i - \mu)^2$$

**In practice:** `numpy.std(x)` defaults to dividing by n (population), while `pandas.Series.std()` defaults to n − 1 (sample). Mismatched defaults are a common bug. To force: `np.std(x, ddof=1)`.

### Coefficient of Variation (CV)

$$CV = \frac{s}{\bar{x}}$$

Unit-free measure of spread *relative* to the mean. Useful when comparing the variability of quantities measured in different units. CV = 0.5 means SD is half the size of the mean.

### IQR-Based Outlier Rule

A common (somewhat arbitrary) rule: x is an outlier if x < Q₁ - 1.5·IQR or x > Q₃ + 1.5·IQR. Used in box plots. Better-justified alternatives exist; this is just a heuristic.

---

## INTERMEDIATE — Z-Score / Standardization (Critical for ML)

### Definition

For a value x from a distribution with mean μ and standard deviation σ:

$$z = \frac{x - \mu}{\sigma}$$

z measures **how many standard deviations x is from the mean**. Positive z is above the mean; negative below. Distribution-agnostic: z always means the same thing.

### Why Z-Scores Matter

1. **Comparability across scales.** Compare a height of 6'2" to an income of $80,000 — meaningless raw, comparable as z-scores.
2. **Statistical tests use them.** Every z-test, t-test test statistic is essentially a z-score of a sample statistic.
3. **ML model preprocessing.** Many ML models (k-NN, SVM, neural nets, regularized linear models) require **standardized features** to work properly; otherwise features with larger numeric ranges dominate the learning.

### Standardization in ML

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)   # learns μ, σ on train
X_test_scaled  = scaler.transform(X_test)        # applies the SAME μ, σ
```

**Critical:** always `fit` on training data only. Fitting on test (or worse, the whole dataset) **leaks information** from the test set into the model. This is the #1 silent bug in ML pipelines.

### Z-Score for Outlier Detection

A traditional rule: flag points with |z| > 3 as outliers. Works when the data is roughly normal; breaks down on heavy-tailed distributions (where extreme values are *normal*, not outliers).

Better alternatives for non-normal data:
- **Modified z-score** with median + MAD (median absolute deviation): more robust.
- **IQR rule** for box-plot outliers.
- **Isolation Forest** / **LOF** for multivariate outliers.

### Modified Z-Score (robust)

$$z_{\text{mod}} = \frac{0.6745 \cdot (x - \text{median})}{\text{MAD}}$$

where MAD = median(|xᵢ - median|). Flag |z_mod| > 3.5 as outliers. Doesn't break on heavy-tailed data.

---

## INTERMEDIATE — Shape Measures

### Skewness

$$\text{skew} = E\left[\left(\frac{X - \mu}{\sigma}\right)^3\right]$$

- skew = 0: symmetric (normal).
- skew > 0: **right-skewed** — long right tail (income, latency, file sizes). Mean > Median.
- skew < 0: **left-skewed** — long left tail (exam scores near ceiling). Mean < Median.

**Interview rule:** mean > median ⟹ right-skewed; mean < median ⟹ left-skewed. Almost all "real-world rare-event" data is right-skewed.

### Kurtosis

$$\text{kurt} = E\left[\left(\frac{X - \mu}{\sigma}\right)^4\right]$$

Measures tail heaviness. Normal distribution has kurtosis 3. **Excess kurtosis** = kurtosis − 3.

- Excess kurt > 0: heavy tails (leptokurtic). Stock returns, latency.
- Excess kurt < 0: light tails (platykurtic). Uniform.

**Why care?** Heavy-tailed data makes outliers look extreme but they're actually expected. Fitting a normal distribution to heavy-tailed data underestimates risk (this is the canonical risk-management failure: Black Swan, LTCM, 2008).

---

## INTERMEDIATE — Correlation and Covariance (on data)

### Sample covariance

$$\text{Cov}(X, Y) = \frac{1}{n-1} \sum_{i=1}^n (x_i - \bar{x})(y_i - \bar{y})$$

Units = units(X) · units(Y). Hard to interpret on its own.

### Pearson correlation

$$r = \frac{\text{Cov}(X, Y)}{s_X \cdot s_Y} \in [-1, 1]$$

Measures **linear** association only. r = 0 means no linear association — **does not mean independent**. The famous "Anscombe's quartet" shows four datasets with identical r ≈ 0.816 but radically different shapes; always plot the data.

### Spearman (rank correlation)

Pearson on the ranks rather than raw values. Captures any monotonic relationship (not just linear). Robust to outliers. Use when:
- Data is ordinal.
- Relationship is monotonic but nonlinear.
- Outliers would distort Pearson.

### Kendall's τ

Counts concordant vs discordant pairs. Even more outlier-robust than Spearman; smaller absolute values for the same data.

### Correlation ≠ Causation

The single most-quoted phrase in stats. Three reasons correlated variables may not be causally related:
1. **Confounding** — a third variable causes both.
2. **Reverse causation** — Y causes X, not X causes Y.
3. **Chance** — small samples, multiple comparisons.

Causal inference techniques (RCTs, instrumental variables, DAGs, propensity scores) try to recover causation from observational data. See [Bayesian & Causal](./07-bayesian-and-causal.md).

---

## ADVANCED — Practical Considerations

### Robust Statistics

When data has outliers, use robust alternatives:

| Standard | Robust |
|---|---|
| Mean | Median, trimmed mean |
| Standard deviation | MAD, IQR |
| Variance | IQR² scaled |
| Pearson r | Spearman, Kendall τ |
| Z-score | Modified z-score (median + MAD) |
| Linear regression | Huber, RANSAC, quantile regression |

### Missing Data Handling

| Mechanism | What it means |
|---|---|
| **MCAR** (Missing Completely At Random) | Missingness independent of anything. Safe to drop. |
| **MAR** (Missing At Random) | Missingness depends on *observed* variables. Multiple imputation OK. |
| **MNAR** (Missing Not At Random) | Missingness depends on *unobserved* values (e.g. people skip income question because income is high). Hard; need a model of missingness. |

Standard fills: mean, median, mode. Smarter: predictive imputation (k-NN, IterativeImputer, MICE). Avoid drop-NaN if you can't show MCAR.

### Computing on Streams

For huge data you can't fit in memory, use **streaming algorithms**:

- **Welford's algorithm** for streaming mean + variance (numerically stable, one pass).
- **Reservoir sampling** for uniform random sample of unknown-length stream.
- **t-digest** for streaming quantiles. See [DE: Probabilistic Structures](../python/12-probabilistic-structures.md).
- **HyperLogLog** for streaming distinct counts.

### Bessel's Correction Intuition

Why divide by n - 1 instead of n? Because the sample mean $\bar{x}$ is itself an estimate from the data, and squared deviations from $\bar{x}$ are systematically smaller than squared deviations from the true μ (the sample mean minimizes the sum of squared deviations). Dividing by n - 1 corrects for this bias.

For large n, n vs n - 1 barely matters. For small n (< 20), it matters a lot.

### Standard Error vs Standard Deviation

Constantly confused; always clarify:

- **Standard deviation (σ or s)** — spread of individual data points.
- **Standard error (SE)** — spread of a *sample statistic* across repeated samples. For the mean: SE = σ/√n (or s/√n).

Reporting "mean ± SD" describes data spread; "mean ± SE" describes how uncertain the mean estimate is. Confidence intervals use SE, not SD.

### Anscombe's Quartet and the Datasaurus Dozen

[Anscombe](https://en.wikipedia.org/wiki/Anscombe%27s_quartet) constructed four datasets with identical means, variances, correlations, and regression lines but visually very different. The [Datasaurus](https://www.autodesk.com/research/publications/same-stats-different-graphs) generalizes — 13 datasets with the same summary stats including one that draws a dinosaur. **Lesson: always visualize before trusting summary stats.**

---

## Computing All of This in Python

```python
import numpy as np
import pandas as pd
from scipy import stats

x = np.array([1, 2, 2, 3, 4, 5, 5, 5, 6, 100])  # has an outlier

# Center
np.mean(x)               # 13.3 — pulled by outlier
np.median(x)             # 4.5  — robust
stats.mode(x).mode       # 5    — most frequent

# Spread
np.var(x, ddof=0)        # population variance (divides by n)
np.var(x, ddof=1)        # sample variance   (divides by n-1)  <-- usually what you want
np.std(x, ddof=1)
np.percentile(x, [25, 50, 75])   # quartiles
stats.iqr(x)

# Shape
stats.skew(x)            # heavily right-skewed
stats.kurtosis(x)        # excess kurtosis (Fisher convention)

# Robust outlier flag
from scipy.stats import median_abs_deviation
mad = median_abs_deviation(x)
mod_z = 0.6745 * (x - np.median(x)) / mad
outliers = np.abs(mod_z) > 3.5

# Correlation
df = pd.DataFrame({'x': [1,2,3,4,5], 'y': [2,4,6,8,11]})
df.corr(method='pearson')
df.corr(method='spearman')
df.corr(method='kendall')

# Standardize
from sklearn.preprocessing import StandardScaler
X_scaled = StandardScaler().fit_transform(X)
```

---

## Common Interview Gotchas

**Q: When would you use median over mean?**
A: Skewed distributions, outliers, ordinal data, or when reporting a "typical" experience honestly.

**Q: Why divide by n − 1 for sample variance?**
A: Bessel's correction — unbiased estimator of population variance. The sample mean uses up one degree of freedom.

**Q: NumPy and pandas give different std values — why?**
A: Different default ddof. `np.std` uses ddof=0 (n); `pd.Series.std` uses ddof=1 (n−1). For sample stats, pass `ddof=1` to NumPy.

**Q: What's a z-score and when do you use it?**
A: (x − μ)/σ — distance from mean in SD units. Used for standardizing features, outlier detection, and as the building block of every z- and t-test.

**Q: Is correlation 0 the same as independence?**
A: No. Correlation 0 means no *linear* relationship; non-linear relationships can have r = 0 (e.g. Y = X² with X symmetric around 0).

**Q: Mean is 50, median is 30 — what's the skew?**
A: Right-skewed (long right tail). Mean > Median is the signature.

**Q: How would you flag outliers?**
A: IQR rule (Q₁ − 1.5·IQR, Q₃ + 1.5·IQR), modified z-score with MAD for non-normal data, or model-based (Isolation Forest, LOF) for multivariate.

**Q: Why do many ML models need feature scaling?**
A: Distance/penalty-based models (k-NN, SVM, regularized regression) treat all features as comparable distances; without scaling, large-magnitude features dominate. Tree-based models (random forest, gradient boosting) don't need scaling.

**Q: What's the difference between standard deviation and standard error?**
A: SD: spread of data. SE: spread of a sample statistic across repeated samples. SE = SD/√n for the mean. Confidence intervals use SE.

**Q: What's wrong with reporting only the mean?**
A: Hides skew, kurtosis, outliers. Datasaurus / Anscombe's quartet show wildly different datasets with identical means. Always pair with spread and ideally a plot.

---

## Interview-Ready Cheat Sheet

### Key formulas

- Sample mean: $\bar{x} = (1/n) \sum x_i$
- Sample variance: $s^2 = (1/(n-1)) \sum (x_i - \bar{x})^2$
- z-score: $z = (x - \mu)/\sigma$
- Modified z-score: $z_{\text{mod}} = 0.6745(x - \text{med})/\text{MAD}$
- CV = s / $\bar{x}$
- Pearson r = Cov(X,Y) / (s_X · s_Y)

### Decision flowchart

| Question | Answer |
|---|---|
| Center of skewed data? | Median |
| Spread of skewed data? | IQR |
| Outliers in normal data? | \|z\| > 3 |
| Outliers in heavy-tailed data? | Modified z-score with MAD |
| Comparing across units? | z-score |
| Standardize features for ML? | `StandardScaler` (fit on train only) |
| Monotonic but nonlinear relation? | Spearman correlation |
| Stream-compute mean+var? | Welford's algorithm |
| Stream-compute quantiles? | t-digest |
| Stream-compute distinct count? | HyperLogLog |

### Quick trade-off pairs

- **Mean vs median:** efficient vs robust.
- **Variance vs IQR:** Gaussian-friendly vs outlier-robust.
- **Pearson vs Spearman vs Kendall:** linear vs monotonic vs heavily-robust monotonic.
- **z-score vs modified z-score:** normal-data assumption vs robust to outliers.
- **Drop-NaN vs imputation:** simple but biased if not MCAR vs more correct but assumption-dependent.

---

## Resources & Links

- *All of Statistics* (Wasserman) — ch. 6.
- [Datasaurus Dozen](https://www.autodesk.com/research/publications/same-stats-different-graphs) — why visualization beats summary.
- [Anscombe's quartet](https://en.wikipedia.org/wiki/Anscombe%27s_quartet)
- [scipy.stats robust statistics](https://docs.scipy.org/doc/scipy/reference/stats.html)
- [scikit-learn preprocessing guide](https://scikit-learn.org/stable/modules/preprocessing.html)
- [Pandas `describe()` and friends](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.describe.html)
- [Khan Academy — Statistics intro](https://www.khanacademy.org/math/statistics-probability)

### Companion Files
- [Probability Foundations](./01-probability-foundations.md) — random variables and distributions behind these measures.
- [Hypothesis Testing](./03-hypothesis-testing.md) — where z-scores become test statistics.
- [ML Metrics](./06-ml-metrics.md) — F1 = harmonic mean of precision and recall.

---

*Next: [Hypothesis Testing](./03-hypothesis-testing.md) — the most-tested cluster in DS interviews.*
