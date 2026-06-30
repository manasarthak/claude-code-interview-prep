# Hypothesis Testing — p-values, t-tests, z-tests, χ², ANOVA

**Phase:** 1 (Inferential statistics)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Probability Foundations](./01-probability-foundations.md) · [Descriptive Stats](./02-descriptive-stats.md) · [A/B Testing](./04-ab-testing.md) · [Regression](./05-regression.md)

---

## Why This File Exists

This is the cluster every DS/ML interview tests: p-values, type I/II errors, t-test, z-test, χ², ANOVA, when each applies, what their assumptions are, and how to compute them. Most candidates can recite definitions and freeze when asked to actually use them. This file gives you the math, the assumptions, and the *intuition* you need to talk fluently under pressure.

---

## BEGINNER — The Framework

### The Hypothesis-Testing Setup

You have data. You want to decide between two claims:

- **Null hypothesis (H₀)** — "no effect / no difference / status quo." The skeptic's claim.
- **Alternative hypothesis (H₁ or Hₐ)** — "there is an effect."

You compute a **test statistic** from the data, and ask: *if H₀ were true, how surprising is this test statistic?* If sufficiently surprising → reject H₀ in favor of H₁.

### The p-Value — Rigorous Definition

> The **p-value** is the probability of observing a test statistic *at least as extreme* as the one you got, **assuming H₀ is true**.

Symbolically: p = P(T ≥ t_observed | H₀).

Key points:
- p is **NOT** the probability that H₀ is true.
- p is **NOT** the probability that you're wrong if you reject H₀.
- p **IS** a conditional probability of the data given a hypothesis.

### The Decision Rule

Pick a **significance level α** before looking at data (commonly 0.05). Then:

- p < α → reject H₀ ("statistically significant" at level α).
- p ≥ α → fail to reject H₀ ("not significant").

You **never "accept" H₀.** You either reject it or fail to reject it. Failure to reject doesn't prove H₀; it means you don't have enough evidence to overturn it.

### One-Tailed vs Two-Tailed

- **Two-tailed:** H₁ is "μ ≠ μ₀". Extreme means either direction matters. Uses |T|.
- **One-tailed:** H₁ is "μ > μ₀" (or "μ < μ₀"). Only one direction matters.

**Use two-tailed by default.** One-tailed is justified only when you genuinely don't care about the other direction (rare in research). Picking one-tailed *after* seeing the data — "p-hacking by direction" — is a classic ethics red flag.

### Type I and Type II Errors

| | H₀ true | H₀ false |
|---|---|---|
| **Reject H₀** | Type I error (α) | Correct ✓ |
| **Fail to reject H₀** | Correct ✓ | Type II error (β) |

- **α** = P(Type I) = false-positive rate = significance level (you choose this).
- **β** = P(Type II) = false-negative rate.
- **Power** = 1 − β = P(reject H₀ | H₁ true) — the probability of detecting a real effect.

**Trade-off:** lowering α reduces false positives but increases β (more false negatives), reducing power. To raise power without raising α: **collect more data**.

### Confidence Intervals (CIs)

A 95% confidence interval is a range computed from the data such that, in repeated sampling, 95% of such intervals would contain the true parameter.

**It is NOT** "95% probability the parameter is in this interval" (that's the Bayesian credible interval). In the frequentist world, the parameter is fixed; the interval is random.

For a sample mean:

$$\bar{x} \pm z_{\alpha/2} \cdot \frac{s}{\sqrt{n}} \quad \text{(use $t$ critical value for small $n$)}$$

**Duality:** a two-sided test at level α rejects H₀: μ = μ₀ iff μ₀ falls outside the 1−α confidence interval. So CIs and tests are two views of the same machinery.

---

## INTERMEDIATE — The Z-Test and the T-Test

### Z-Test for a Single Mean (when σ known)

Setup: sample of n i.i.d. observations from a distribution with unknown mean μ and **known** σ. Test H₀: μ = μ₀.

Test statistic:

$$Z = \frac{\bar{x} - \mu_0}{\sigma / \sqrt{n}}$$

Under H₀, by [CLT](./01-probability-foundations.md), Z ~ N(0, 1) approximately for large n. Compute p-value from the standard normal distribution.

**Reject H₀ at α = 0.05 (two-sided):** |Z| > 1.96.

When does the z-test actually apply? In practice: rarely. We almost never know σ. So we use the **t-test**.

### One-Sample t-Test

Setup: sample of n from a roughly-normal distribution; **σ unknown**, estimated by s. Test H₀: μ = μ₀.

$$t = \frac{\bar{x} - \mu_0}{s / \sqrt{n}}$$

Under H₀, t follows a **Student's t distribution with n − 1 degrees of freedom**. As n → ∞, t → N(0, 1). For small n the t distribution has heavier tails, giving you wider intervals and being properly conservative.

**Assumptions:**
1. Observations are i.i.d.
2. The underlying distribution is approximately normal (matters most for small n; CLT rescues you for n ≥ 30).
3. σ is unknown and estimated from s.

#### Python

```python
from scipy import stats
data = [5.1, 4.9, 5.2, 5.0, 4.8, 5.3, 5.1]
t_stat, p_value = stats.ttest_1samp(data, popmean=5.0)
```

### Two-Sample (Independent) t-Test

Two independent groups, asking whether their means differ. H₀: μ₁ = μ₂.

#### Welch's t-test (default, robust)

Does **not** assume equal variances. Test statistic:

$$t = \frac{\bar{x}_1 - \bar{x}_2}{\sqrt{s_1^2/n_1 + s_2^2/n_2}}$$

Degrees of freedom from the Welch-Satterthwaite formula (`scipy` handles it). **Use Welch's by default** — it's only marginally less powerful than the equal-variance version when variances really are equal, and much safer when they aren't.

#### Student's t-test (assumes equal variances)

If you can defend the equal-variance assumption (e.g. by Levene's test):

$$t = \frac{\bar{x}_1 - \bar{x}_2}{s_p \sqrt{1/n_1 + 1/n_2}}, \quad s_p^2 = \frac{(n_1-1)s_1^2 + (n_2-1)s_2^2}{n_1 + n_2 - 2}$$

DoF = n₁ + n₂ − 2.

#### Python

```python
# Welch's (default in scipy)
t_stat, p_value = stats.ttest_ind(group_A, group_B, equal_var=False)

# Student's
t_stat, p_value = stats.ttest_ind(group_A, group_B, equal_var=True)
```

### Paired t-Test

When the two samples are **paired** (same subject before/after, matched subjects). Compute differences dᵢ = xᵢ − yᵢ and run a one-sample t-test on the differences against H₀: μ_d = 0.

$$t = \frac{\bar{d}}{s_d / \sqrt{n}}, \quad \text{df} = n - 1$$

**Why pair when you can?** It removes between-subject variance, dramatically increasing power. If you have 20 patients each measured before and after a drug, the paired test is much stronger than treating them as two independent groups of 20.

```python
t_stat, p_value = stats.ttest_rel(before, after)
```

### When the Normality Assumption Fails

For small n with strongly skewed or heavy-tailed data, t-tests can be miscalibrated. Alternatives:

- **Wilcoxon signed-rank** (paired) — non-parametric paired test.
- **Mann-Whitney U** (independent) — non-parametric two-sample test.
- **Bootstrap** — resample with replacement to estimate the distribution of the test statistic directly. Distribution-free; very general.
- **Permutation test** — under H₀ the labels are exchangeable; permute them many times to get a null distribution of the statistic.

```python
stats.mannwhitneyu(a, b, alternative='two-sided')
stats.wilcoxon(before, after)
```

### Z vs t in One Sentence

Use **z** when you know σ (rare); use **t** when you estimate σ from the sample (almost always). For large n the two are essentially identical.

---

## INTERMEDIATE — Tests for Proportions and Counts

### Z-Test for a Proportion

A coin lands heads $\hat{p} = 60/100$ times. Is it fair?

$$Z = \frac{\hat{p} - p_0}{\sqrt{p_0(1-p_0)/n}}$$

Under H₀: p = p_0, by CLT, Z ≈ N(0, 1). For p_0 = 0.5, $\hat{p}$ = 0.6, n = 100:

$$Z = \frac{0.6 - 0.5}{\sqrt{0.25/100}} = \frac{0.1}{0.05} = 2.0$$

Two-sided p ≈ 0.046. **Marginally significant** at α = 0.05.

### Two-Proportion Z-Test (the A/B-test core)

Comparing conversion rates of two groups. H₀: p_A = p_B.

$$Z = \frac{\hat{p}_A - \hat{p}_B}{\sqrt{\hat{p}(1-\hat{p}) \cdot (1/n_A + 1/n_B)}}$$

where $\hat{p}$ is the pooled proportion = (xA + xB)/(nA + nB).

```python
from statsmodels.stats.proportion import proportions_ztest
stat, p = proportions_ztest(count=[35, 50], nobs=[1000, 1000])
```

### Chi-Squared (χ²) Test

For categorical data. Two flavors:

#### Goodness-of-fit

"Does my observed frequency match an expected distribution?"

$$\chi^2 = \sum_i \frac{(O_i - E_i)^2}{E_i}, \quad \text{df} = k - 1$$

where O_i is observed count in bin i, E_i is expected count under H₀. Under H₀, the statistic ~ χ²(k − 1).

Example: a die rolled 60 times shows {15, 8, 12, 10, 9, 6}. H₀: fair. E_i = 10 for each face.

$$\chi^2 = \frac{25 + 4 + 4 + 0 + 1 + 16}{10} = 5.0, \quad \text{df} = 5$$

Critical value at α=0.05 for χ²(5) is 11.07. 5.0 < 11.07 → fail to reject. Die looks fair.

#### Test of independence

"Are these two categorical variables associated?" Build a contingency table; compute expected counts assuming independence; compare to observed.

$$E_{ij} = \frac{(\text{row}_i\text{ total})(\text{col}_j\text{ total})}{n}, \quad \chi^2 = \sum_{ij} \frac{(O_{ij} - E_{ij})^2}{E_{ij}}, \quad \text{df} = (r-1)(c-1)$$

```python
from scipy.stats import chi2_contingency
table = [[35, 65], [50, 50]]   # rows = group; cols = success/failure
chi2, p, dof, expected = chi2_contingency(table)
```

**Assumption:** expected counts ≥ 5 in each cell (rule of thumb). For sparse tables, use **Fisher's exact test** (`scipy.stats.fisher_exact` for 2×2).

### When to Use Which Test for Proportions

- Two-proportion z-test ↔ 2×2 χ² test — algebraically equivalent.
- Sparse 2×2 → Fisher's exact.
- More than two groups, categorical outcome → χ² of independence.

---

## INTERMEDIATE — ANOVA (Comparing Many Means)

### One-Way ANOVA

You have k ≥ 3 groups and want to test H₀: μ₁ = μ₂ = ... = μ_k. Don't do pairwise t-tests — that inflates Type I error.

ANOVA partitions variance:

$$\text{SS}_{\text{total}} = \text{SS}_{\text{between}} + \text{SS}_{\text{within}}$$

The F-statistic:

$$F = \frac{\text{SS}_{\text{between}} / (k-1)}{\text{SS}_{\text{within}} / (n-k)} = \frac{\text{MSB}}{\text{MSW}}$$

Under H₀, F ~ F(k − 1, n − k). Large F → groups differ.

**Assumptions:**
1. Independence within and between groups.
2. Approximately normal within each group.
3. Equal variances across groups (Levene's test checks this).

```python
stats.f_oneway(group1, group2, group3)
```

If you reject H₀, you know *at least one* group differs — but not which. Follow with **post-hoc tests** (Tukey HSD, Bonferroni-corrected pairwise t).

```python
from statsmodels.stats.multicomp import pairwise_tukeyhsd
res = pairwise_tukeyhsd(values, group_labels, alpha=0.05)
```

Non-parametric ANOVA equivalent: **Kruskal-Wallis** test.

```python
stats.kruskal(group1, group2, group3)
```

---

## ADVANCED — Multiple Comparisons, Power, and the Replication Crisis

### Multiple Comparisons Problem

If you run 20 independent hypothesis tests at α = 0.05, the probability of at least one false positive under all-true-H₀ is 1 − 0.95²⁰ ≈ 64%. Run enough tests, find anything.

**Corrections:**

#### Bonferroni (Family-Wise Error Rate)

Test each hypothesis at α / m, where m is the number of tests. Guarantees FWER ≤ α. Conservative — power drops fast as m grows.

#### Benjamini-Hochberg (False Discovery Rate)

Controls the expected proportion of rejected nulls that are false positives, not the probability of any false positive. Less conservative. Standard for genomics and large-scale screening.

```python
from statsmodels.stats.multitest import multipletests
reject, p_adj, _, _ = multipletests(pvals, alpha=0.05, method='bonferroni')
reject, p_adj, _, _ = multipletests(pvals, alpha=0.05, method='fdr_bh')
```

### Power Analysis

Before collecting data, ask: **how many samples do I need to detect an effect of size δ with power 1 − β?**

For a two-sample t-test:

$$n \approx \frac{2 \sigma^2 (z_{\alpha/2} + z_\beta)^2}{\delta^2}$$

So sample size scales **quadratically** with the effect size you want to detect — halving the detectable effect requires 4× the samples.

```python
from statsmodels.stats.power import TTestIndPower
analysis = TTestIndPower()
n = analysis.solve_power(effect_size=0.3, alpha=0.05, power=0.8)
# n ≈ 175 per group
```

This is critical for [A/B testing](./04-ab-testing.md): if your test isn't powered to detect the effect you care about, "no significant difference" is meaningless.

### Effect Size

p-values describe statistical significance; **effect size** describes practical significance. With huge samples, tiny p < 0.05 effects can be meaningless in practice; with tiny samples, big effects might fail to reach significance.

Common effect-size measures:

- **Cohen's d** (two means): d = ($\bar{x}_1 - \bar{x}_2$) / s_pooled. Conventionally small ≈ 0.2, medium ≈ 0.5, large ≈ 0.8.
- **Cohen's h** (two proportions): arcsine-based.
- **Pearson r** (correlation).
- **Odds ratio / relative risk** (categorical).

**Always report effect size with the p-value.** A senior interviewer will ask.

### The Replication Crisis (worth name-dropping)

Many published "significant" findings don't replicate. Causes:
- p-hacking (try many tests, report only those that "work").
- HARKing (Hypothesizing After Results are Known).
- Publication bias (only significant results are published).
- Low power (small samples + big effect-size estimate → inflated when "significant").

Standards now require: pre-registration, larger samples, reported effect sizes + CIs, and replicating across datasets. The [Nature 2016 survey](https://www.nature.com/articles/533452a) is the canonical reference.

### Bootstrap Confidence Intervals

When you can't compute a closed-form CI:

1. Resample your data with replacement, B times (B = 10,000 typical).
2. Compute your statistic on each resample.
3. Take the 2.5th and 97.5th percentiles of those B values as a 95% CI.

```python
from scipy.stats import bootstrap
res = bootstrap((data,), statistic=np.mean, n_resamples=10000, confidence_level=0.95)
print(res.confidence_interval)
```

Works for any statistic — median, correlation, model parameter, ratio of means. **Universal escape hatch when the math is hard.**

### Permutation Tests

For two-sample tests where the parametric assumptions look shaky:

1. Combine both samples, shuffle labels, recompute the test stat.
2. Repeat B times. Fraction of shuffles ≥ observed → permutation p-value.

```python
stats.permutation_test((a, b), statistic=lambda x, y: np.mean(x) - np.mean(y),
                       n_resamples=10000, alternative='two-sided')
```

Distribution-free; only assumes exchangeability under H₀.

---

## Test Picker (the most-asked interview chart)

| Question | Test |
|---|---|
| One sample mean, σ known | z-test (1-sample) |
| One sample mean, σ unknown | **t-test (1-sample)** |
| Two independent group means | **Welch's t-test** (default) or Student's t |
| Two paired groups | **Paired t-test** |
| Two means, non-normal small n | Mann-Whitney (independent), Wilcoxon (paired) |
| 3+ group means | **One-way ANOVA**; non-parametric: Kruskal-Wallis |
| ANOVA pairwise post-hoc | Tukey HSD |
| One proportion vs target | z-test for proportion |
| Two proportions (A/B test core) | **Two-proportion z-test** ↔ 2×2 χ² |
| Categorical association | **χ² of independence**; sparse → Fisher's exact |
| Distribution fit | χ² goodness-of-fit; KS test |
| Comparing variances | F-test (Levene's is more robust) |
| Many tests at once | Bonferroni or BH-FDR correction |
| No assumptions safe | **Bootstrap or permutation test** |

---

## Worked Example — End-to-End Hypothesis Test

**Problem:** Two checkout flows. Control: 1000 visitors, 80 conversions (8%). Variant: 1000 visitors, 100 conversions (10%). Is the variant better?

**Step 1 — Hypotheses.**
H₀: p_v = p_c. H₁: p_v ≠ p_c (two-tailed; we care about either direction).

**Step 2 — Pick the test.**
Two proportions, large n. Two-proportion z-test (equivalent: χ² of 2×2).

**Step 3 — Compute.**

Pooled $\hat{p}$ = 180/2000 = 0.09.

SE = √(0.09 · 0.91 · (1/1000 + 1/1000)) ≈ √(0.0001638) ≈ 0.0128.

Z = (0.10 − 0.08) / 0.0128 ≈ 1.563.

Two-sided p ≈ 0.118.

**Step 4 — Decide.**
At α = 0.05, p = 0.118 > 0.05 → fail to reject H₀. Not significant. The 2 percentage-point lift could be noise at this sample size.

**Step 5 — Power check.**
What sample size *would* have been needed to detect a 0.02 lift with 80% power?

For p ≈ 0.09, σ² ≈ 0.0819. Using the two-proportion sample-size formula (Cohen's h ≈ 0.0703):

$$n \approx \frac{2(z_{0.025} + z_{0.2})^2}{h^2} = \frac{2(1.96 + 0.84)^2}{0.0703^2} \approx 3170 \text{ per group}$$

We needed ~3170 visitors per group, not 1000. The test was **underpowered** — "no significant difference" doesn't mean "no real difference."

**Step 6 — Report.**
"Observed lift: 8% → 10% (relative +25%). Two-proportion z-test p = 0.118; not significant at α = 0.05. 95% CI for the lift: [-0.5%, +4.5%]. Test was underpowered; need ~3000 per group to detect a 2pp lift at 80% power."

This is what a senior answer looks like.

---

## Common Interview Gotchas

**Q: What does p = 0.04 mean?**
A: If H₀ were true, there'd be a 4% chance of seeing data this extreme or more. NOT "4% chance H₀ is true."

**Q: Why is p = 0.06 "not significant" while p = 0.04 is?**
A: Only because we chose α = 0.05. The threshold is conventional, not magic. Report the p-value; let readers decide.

**Q: Can a test be statistically significant but practically meaningless?**
A: Yes — with huge n, tiny effects clear any threshold. Always report effect size.

**Q: What's the difference between Type I and Type II errors?**
A: Type I: reject H₀ when it's true (false positive). Type II: fail to reject H₀ when H₁ is true (false negative).

**Q: When do you use a paired test?**
A: When the two samples are dependent — same subject measured twice, matched pairs. It removes between-subject variance.

**Q: Why not just run pairwise t-tests for k groups?**
A: Multiple-comparisons inflation. With k groups you have k(k-1)/2 pairs; at α = 0.05 each, the FWER explodes. Use ANOVA first, then post-hoc.

**Q: When is the t-test invalid?**
A: When data is non-normal and n is small. Switch to non-parametric (Mann-Whitney, Wilcoxon) or use bootstrap.

**Q: What's the F-statistic in ANOVA?**
A: Ratio of between-group variance to within-group variance. Large F → groups differ.

**Q: What's power? What does β = 0.2 mean?**
A: Power = 1 − β = probability of correctly rejecting H₀ when H₁ is true. β = 0.2 → 80% power → 20% chance of missing a real effect.

**Q: How do you control for multiple comparisons in a 1000-feature genomics study?**
A: Bonferroni is too conservative; use Benjamini-Hochberg to control FDR.

**Q: What's a confidence interval?**
A: A range computed from data; under repeated sampling, 95% of such intervals would contain the true parameter. **Not** "95% chance the parameter is in this interval."

**Q: When would you bootstrap?**
A: When the sampling distribution of your statistic isn't analytically tractable (e.g. CI for the median, correlation, ratio of means, model parameter).

---

## Interview-Ready Cheat Sheet

### Key formulas

- **One-sample t:** $t = (\bar{x} - \mu_0)/(s/\sqrt{n})$, df = n − 1
- **Two-sample Welch's t:** $t = (\bar{x}_1 - \bar{x}_2)/\sqrt{s_1^2/n_1 + s_2^2/n_2}$
- **Paired t:** $t = \bar{d}/(s_d/\sqrt{n})$, df = n − 1
- **One-proportion z:** $Z = (\hat{p} - p_0)/\sqrt{p_0(1-p_0)/n}$
- **Two-proportion z (pooled):** $Z = (\hat{p}_1 - \hat{p}_2)/\sqrt{\hat{p}(1-\hat{p})(1/n_1+1/n_2)}$
- **χ²:** $\sum (O - E)^2/E$
- **ANOVA F:** MS_between / MS_within
- **Sample-size for two means:** n ≈ 2σ²(z_{α/2} + z_β)² / δ²

### Quick z critical values

| α (two-sided) | z |
|---|---|
| 0.10 | 1.645 |
| 0.05 | 1.960 |
| 0.01 | 2.576 |
| 0.001 | 3.291 |

### Trade-off pairs

- **z-test vs t-test:** σ known vs σ estimated.
- **One-tailed vs two-tailed:** directional vs both-direction; default two.
- **Welch's vs Student's t:** robust to unequal variances vs assumes equal.
- **Parametric vs non-parametric:** normality assumed vs distribution-free.
- **Bonferroni vs BH-FDR:** controls FWER (conservative) vs FDR (powerful).
- **p-value vs effect size:** statistical vs practical significance — always report both.

---

## Resources & Links

- *All of Statistics* (Wasserman) ch. 10–11.
- *Statistical Inference* (Casella & Berger) — the rigorous treatment.
- [Wasserman's "Frequentist and Bayesian Methods"](https://www.stat.cmu.edu/~larry/=stat705/Lec14.pdf)
- [scipy.stats hypothesis tests](https://docs.scipy.org/doc/scipy/reference/stats.html#hypothesis-tests-and-related-functions)
- [statsmodels — power & sample size](https://www.statsmodels.org/stable/stats.html)
- [StatQuest — t-tests playlist](https://www.youtube.com/playlist?list=PLblh5JKOoLUK0FLuzwntyYI10UQFUhsY9)
- [Nature: 1,500 scientists lift the lid on reproducibility](https://www.nature.com/articles/533452a)
- [American Statistical Association — Statement on p-values](https://amstat.tandfonline.com/doi/full/10.1080/00031305.2016.1154108)

### Companion Files
- [Probability Foundations](./01-probability-foundations.md) — CLT, distributions.
- [Descriptive Stats](./02-descriptive-stats.md) — sample variance, SE.
- [A/B Testing](./04-ab-testing.md) — applying these in experimentation.
- [Regression](./05-regression.md) — coefficient p-values are these same tests.

---

*Next: [A/B Testing](./04-ab-testing.md) — apply hypothesis testing to experimentation.*
