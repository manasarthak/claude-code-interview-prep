# Probability Foundations — Random Variables, Distributions, CLT, Bayes

**Phase:** 0 (Mandatory foundation)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Descriptive Stats](./02-descriptive-stats.md) · [Hypothesis Testing](./03-hypothesis-testing.md)

---

## Why This File Exists

Every later file uses the vocabulary here: random variables, expectations, variance, distributions, the Central Limit Theorem, Bayes' theorem. If any of these are fuzzy, hypothesis testing becomes spell-casting.

---

## BEGINNER — Random Variables, Expectation, Variance

### Random Variables (RV)

A random variable X is a function that maps each outcome of a random experiment to a number. Two flavors:

- **Discrete** — takes countable values. Number of heads in 10 flips: 0, 1, ..., 10.
- **Continuous** — takes values on a continuum. Height in meters.

For a discrete RV we use the **probability mass function (PMF)** p(x) = P(X = x). For a continuous RV we use the **probability density function (PDF)** f(x), where probability over an interval is the area under f:

$$P(a \le X \le b) = \int_a^b f(x)\, dx$$

For both we have the **cumulative distribution function (CDF)**:

$$F(x) = P(X \le x)$$

For a continuous RV, P(X = x) = 0 exactly; only intervals have nonzero probability.

### Expectation (the mean of a random variable)

$$E[X] = \sum_x x \cdot p(x) \quad \text{(discrete)} \qquad E[X] = \int x f(x) \, dx \quad \text{(continuous)}$$

**Key properties** (memorize):
- **Linearity:** E[aX + bY] = aE[X] + bE[Y], for any X, Y (even dependent).
- E[c] = c for a constant.
- If X and Y are independent: E[XY] = E[X]E[Y].

### Variance and Standard Deviation

Variance measures spread:

$$\text{Var}(X) = E[(X - E[X])^2] = E[X^2] - (E[X])^2$$

Standard deviation: σ = √Var(X). Same units as X (mean is in dollars, variance is in dollars², SD is in dollars).

**Properties:**
- Var(aX + b) = a²·Var(X).
- For independent X, Y: Var(X + Y) = Var(X) + Var(Y).
- Var is **not linear** in general: if X, Y aren't independent, Var(X + Y) = Var(X) + Var(Y) + 2·Cov(X, Y).

### Covariance and Correlation

**Covariance** measures linear co-movement:

$$\text{Cov}(X, Y) = E[(X - E[X])(Y - E[Y])] = E[XY] - E[X]E[Y]$$

**Correlation** is unit-free covariance:

$$\rho(X, Y) = \frac{\text{Cov}(X, Y)}{\sigma_X \sigma_Y} \in [-1, 1]$$

ρ = 1 means perfect positive linear relationship; ρ = -1 perfect negative; ρ = 0 means linearly uncorrelated (NOT necessarily independent). See [Descriptive Stats](./02-descriptive-stats.md) for the data-side version.

---

## INTERMEDIATE — Conditional Probability and Bayes' Theorem

### Conditional Probability

$$P(A \mid B) = \frac{P(A \cap B)}{P(B)}, \quad P(B) > 0$$

In words: among the worlds where B happened, what fraction also have A?

**Multiplication rule:** P(A ∩ B) = P(A | B) · P(B) = P(B | A) · P(A).

**Independence:** A and B independent ⟺ P(A | B) = P(A) ⟺ P(A ∩ B) = P(A)·P(B). **Independence does NOT mean mutually exclusive.** Mutually exclusive events with positive probability are *strongly dependent* (knowing one happened guarantees the other didn't).

### Law of Total Probability

If B₁, ..., Bₙ partition the sample space:

$$P(A) = \sum_i P(A \mid B_i) \cdot P(B_i)$$

### Bayes' Theorem — The Most-Tested Formula in DS Interviews

$$P(H \mid E) = \frac{P(E \mid H) \cdot P(H)}{P(E)}$$

In words: **posterior ∝ likelihood × prior**.

- P(H) — **prior** belief in hypothesis H before seeing evidence.
- P(E | H) — **likelihood** of evidence under H.
- P(E) — **marginal** probability of evidence; often expanded via total probability: P(E) = Σ P(E | Hᵢ)·P(Hᵢ).
- P(H | E) — **posterior** belief in H after seeing E.

### The Classic Bayes' Question (drill this)

> "A disease affects 1 in 1000 people. A test is 99% sensitive (catches 99% of disease cases) and 99% specific (correctly clears 99% of healthy people). A random person tests positive. What's the probability they have the disease?"

Define:
- P(D) = 0.001 (disease prevalence)
- P(+ | D) = 0.99 (sensitivity)
- P(− | ¬D) = 0.99 ⟹ P(+ | ¬D) = 0.01 (false positive rate)

We want P(D | +).

$$P(+) = P(+ \mid D)P(D) + P(+ \mid \neg D)P(\neg D) = 0.99 \cdot 0.001 + 0.01 \cdot 0.999 \approx 0.01098$$

$$P(D \mid +) = \frac{0.99 \cdot 0.001}{0.01098} \approx 0.09$$

**Only 9% chance.** The counter-intuitive answer: when the disease is rare, even a "99% accurate" test produces mostly false positives. **This is why screening tests for rare conditions are followed by confirmatory tests.**

### The "Tree" Intuition

Imagine 100,000 people:
- 100 have the disease → 99 test positive, 1 test negative.
- 99,900 are healthy → 999 test positive (false), 98,901 test negative.
- Total positives = 99 + 999 = 1,098. True positives / total positives = 99/1,098 ≈ 9%.

**Always sketch the tree.** Trees beat formulas for staying mistake-free under interview pressure.

---

## ADVANCED — Distributions You Must Know Cold

### Discrete Distributions

| Distribution | PMF / parameters | E[X] | Var(X) | Use case |
|---|---|---|---|---|
| **Bernoulli(p)** | P(X=1)=p, P(X=0)=1-p | p | p(1-p) | Single binary trial |
| **Binomial(n, p)** | $\binom{n}{k} p^k (1-p)^{n-k}$ | np | np(1-p) | k successes in n trials |
| **Geometric(p)** | (1-p)^(k-1)·p | 1/p | (1-p)/p² | Trials until first success |
| **Poisson(λ)** | e^{-λ} λ^k / k! | λ | λ | Counts over fixed interval |
| **Negative Binomial(r, p)** | counts trials for r successes | r/p | r(1-p)/p² | Generalization of geometric |

#### Binomial intuition

If you flip a fair coin 10 times, the count of heads is Binomial(10, 0.5). Expected: 5. Variance: 2.5. SD: ~1.58. So 95% of the time you'll see 5 ± ~3 heads.

#### Poisson — "the rare-event distribution"

Models counts of independent events in a fixed interval (clicks/hour, accidents/year, requests/sec). The single parameter λ is **both the mean and the variance** — that's the Poisson signature. If your data's variance is much larger than its mean, you have **overdispersion** and need a Negative Binomial instead.

#### Binomial → Poisson approximation

As n → ∞, p → 0, with np = λ fixed, Binomial(n, p) → Poisson(λ). Useful when n is huge and p tiny.

### Continuous Distributions

| Distribution | PDF / params | E[X] | Var(X) | Use case |
|---|---|---|---|---|
| **Uniform(a, b)** | 1/(b-a) on [a,b] | (a+b)/2 | (b-a)²/12 | "No information" baseline |
| **Normal(μ, σ²)** | (1/√(2πσ²)) exp(-(x-μ)²/(2σ²)) | μ | σ² | The default; CLT |
| **Exponential(λ)** | λ e^{-λx} on x≥0 | 1/λ | 1/λ² | Time until next Poisson event |
| **Gamma(k, θ)** | generalization of exp | kθ | kθ² | Waiting time for k events |
| **Beta(α, β)** | on [0,1] | α/(α+β) | αβ/((α+β)²(α+β+1)) | Distribution over probabilities (Bayesian) |
| **t(ν)** | (heavier-tailed normal) | 0 (ν>1) | ν/(ν-2) (ν>2) | Small-sample inference |
| **χ²(k)** | sum of k squared standard normals | k | 2k | Variance tests, GoF |
| **F(d₁, d₂)** | ratio of two χ²s | d₂/(d₂-2) | — | ANOVA |

### The Normal Distribution — Tools You Need

If X ~ N(μ, σ²):

- **Standardization (z-score):** Z = (X - μ)/σ, where Z ~ N(0, 1). Tells you "how many SDs from the mean."
- **68–95–99.7 rule:** P(|Z| ≤ 1) ≈ 0.68, P(|Z| ≤ 2) ≈ 0.95, P(|Z| ≤ 3) ≈ 0.997.
- **Key quantiles** to memorize:
  - z₀.₀₂₅ = 1.96 (two-tailed 95%)
  - z₀.₀₅ = 1.645 (one-tailed 95%)
  - z₀.₀₀₅ = 2.576 (two-tailed 99%)
- **Sum of independent normals is normal.** If X ~ N(μ₁, σ₁²) and Y ~ N(μ₂, σ₂²) and X⊥Y, then X + Y ~ N(μ₁+μ₂, σ₁²+σ₂²).

### t-Distribution — When You Don't Know σ

If you're inferring a mean from a small sample and estimate σ from the data, the studentized statistic (sample_mean − true_mean) / (s/√n) is not normal — it's **Student's t** with ν = n−1 degrees of freedom. Heavier tails than normal; as ν → ∞, t → N(0, 1).

This is the foundation of the **t-test** ([file 03](./03-hypothesis-testing.md)).

### χ² Distribution — Sum of Squared Standard Normals

If Z₁, ..., Z_k ~ N(0,1) independent, then Z₁² + ... + Z_k² ~ χ²(k). Used in:
- Goodness-of-fit tests
- Tests for variance
- Likelihood-ratio tests

### F Distribution — Ratio of χ²s

F = (χ²(d₁)/d₁) / (χ²(d₂)/d₂). Foundation of ANOVA — comparing variances between groups.

### Where to Compute These in Python

```python
from scipy import stats

# Discrete
stats.binom(n=10, p=0.5).pmf(5)       # P(X = 5)
stats.binom(n=10, p=0.5).cdf(5)       # P(X ≤ 5)
stats.poisson(mu=3).pmf(2)            # λ=3, P(X=2)

# Continuous
stats.norm(loc=0, scale=1).pdf(1.96)  # PDF at x=1.96
stats.norm(loc=0, scale=1).cdf(1.96)  # CDF at x=1.96 ≈ 0.975
stats.norm.ppf(0.975)                  # Inverse CDF (quantile): returns 1.96

# Sampling
stats.norm(0, 1).rvs(size=1000)        # 1000 i.i.d. samples
```

`ppf` (percent-point function) is the inverse-CDF — use it to get critical values for hypothesis tests.

---

## ADVANCED — The Two Most-Cited Theorems

### Law of Large Numbers (LLN)

For X₁, ..., Xₙ i.i.d. with mean μ:

$$\bar{X}_n = \frac{1}{n}\sum_{i=1}^n X_i \to \mu \quad \text{as } n \to \infty$$

The sample mean converges to the population mean. Why polling works.

### Central Limit Theorem (CLT) — The Most Important Theorem in Applied Stats

For X₁, ..., Xₙ i.i.d. with mean μ and variance σ² (finite):

$$\frac{\bar{X}_n - \mu}{\sigma / \sqrt{n}} \xrightarrow{d} N(0, 1) \quad \text{as } n \to \infty$$

In words: **whatever the underlying distribution, the sample mean is approximately normal for large n**, with mean μ and standard error σ/√n.

**Why it matters:**
- Justifies the normal approximation used in z-tests, t-tests, confidence intervals.
- Lets us reason about means without knowing the underlying distribution shape.
- "Large n" is fuzzy: n ≥ 30 is the textbook rule of thumb; skewed distributions need more.

**What CLT does NOT say:**
- It does **not** say individual data points are normal.
- It does **not** apply when variance is infinite (e.g. Cauchy distribution — no mean, no variance, CLT fails).
- It does **not** apply to dependent or non-identically-distributed samples without modifications.

### Worked CLT example

A factory makes widgets weighing 100g on average with σ = 10g (not normally distributed). You take a sample of 100. What's the probability the sample mean is below 99g?

By CLT, $\bar{X} \approx N(100, 10^2 / 100) = N(100, 1)$. So:

$$P(\bar{X} < 99) = P\left(Z < \frac{99 - 100}{1}\right) = P(Z < -1) \approx 0.16$$

About 16%.

---

## Common Interview Gotchas

**Q: What's the difference between independence and being mutually exclusive?**
A: Independent: P(A | B) = P(A). Mutually exclusive: P(A ∩ B) = 0 (only one can happen). Mutually exclusive events with positive probability are *strongly dependent*.

**Q: What's a probability density?**
A: For a continuous RV, f(x) isn't a probability — it's a *density*. P(X exactly = x) = 0; probabilities come from integrating f over an interval. Density can exceed 1.

**Q: When does the CLT fail?**
A: When variance is infinite (e.g. Cauchy), when samples are heavily dependent without correction, or when n is too small for the distribution's skew (highly skewed → need bigger n).

**Q: Why does Poisson approximate Binomial?**
A: When n is large and p small with np = λ, the binomial PMF converges to the Poisson PMF. Useful for rare-event modeling.

**Q: What's a z-score?**
A: (x - μ)/σ — distance from the mean in units of standard deviations. Standardizes any distribution onto a comparable scale.

**Q: If two variables have correlation 0, are they independent?**
A: Not necessarily. Correlation = 0 means no *linear* relationship. Y = X² with X ~ N(0,1) has Cov(X, Y) = 0 but X and Y are not independent.

**Q: What's E[X²] - E[X]² called?**
A: Var(X). It's the most useful identity for computing variance.

**Q: A coin lands heads 60 times in 100 flips. Is it fair?**
A: This is a hypothesis test. Under H₀: p = 0.5, the count is Binomial(100, 0.5) with mean 50, SD = √25 = 5. Observing 60 is 2 SDs from mean — p ≈ 0.046 two-sided. **Marginally significant**. See [Hypothesis Testing](./03-hypothesis-testing.md).

---

## Interview-Ready Cheat Sheet

### Memorize these formulas cold

- E[X] = Σ x·p(x), Var(X) = E[X²] - E[X]²
- E[aX + bY] = aE[X] + bE[Y] (always; even dependent)
- Var(X + Y) = Var(X) + Var(Y) only if X⊥Y
- Bayes: P(H|E) = P(E|H)P(H)/P(E)
- z-score: z = (x - μ)/σ
- CLT: $\bar{X} \approx N(\mu, \sigma^2/n)$ for large n
- Standard error of the mean: SE = σ/√n (or s/√n if estimated)

### Memorize these z-quantiles

| Confidence | Two-tailed z | One-tailed z |
|---|---|---|
| 90% | 1.645 | 1.282 |
| 95% | 1.960 | 1.645 |
| 99% | 2.576 | 2.326 |

### Distribution picker

| Setting | Distribution |
|---|---|
| Single binary trial | Bernoulli |
| Sum of n binary trials | Binomial |
| Counts of rare events | Poisson |
| Time between Poisson events | Exponential |
| Sum of k waiting times | Gamma |
| Probability of probabilities (Bayesian) | Beta |
| Real-valued "default" | Normal |
| Small-sample mean | t |
| Variance comparisons | χ², F |

---

## Resources & Links

- *All of Statistics* (Wasserman) — chapters 1–5.
- [3Blue1Brown — Bayes Theorem](https://www.youtube.com/watch?v=HZGCoVF3YvM) — best visual intuition.
- [Seeing Theory — Probability Distributions](https://seeing-theory.brown.edu/probability-distributions/index.html)
- [scipy.stats reference](https://docs.scipy.org/doc/scipy/reference/stats.html)
- [Wikipedia: List of probability distributions](https://en.wikipedia.org/wiki/List_of_probability_distributions)
- [Khan Academy AP Stats](https://www.khanacademy.org/math/ap-statistics) — clearest 101.

### Companion Files
- [Descriptive Statistics](./02-descriptive-stats.md) — measures on actual data.
- [Hypothesis Testing](./03-hypothesis-testing.md) — uses everything here.

---

*Next: [Descriptive Statistics](./02-descriptive-stats.md) — measures of center, spread, shape, and the all-important z-score.*
