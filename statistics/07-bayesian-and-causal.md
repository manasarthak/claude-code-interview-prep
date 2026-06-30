# Bayesian Statistics & Causal Inference

**Phase:** 3 (Differentiator)
**Difficulty progression:** Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Probability Foundations](./01-probability-foundations.md) · [Hypothesis Testing](./03-hypothesis-testing.md) · [A/B Testing](./04-ab-testing.md)

---

## Why This File Exists

Bayesian statistics and causal inference are the two "I read papers" topics in DS interviews. You're not expected to be an expert, but mentioning **conjugate priors**, **MAP vs MLE**, **confounders**, **Simpson's paradox**, and **propensity scores** in the right moments shifts the perceived seniority of your answers.

---

## BAYESIAN STATISTICS

### The Core Identity

$$P(\theta \mid \text{data}) = \frac{P(\text{data} \mid \theta) \cdot P(\theta)}{P(\text{data})}$$

- **Posterior** P(θ | data) — what we believe about θ after seeing data.
- **Likelihood** P(data | θ) — how the data is generated given θ.
- **Prior** P(θ) — what we believed about θ before.
- **Evidence** P(data) = ∫ P(data | θ) P(θ) dθ — normalizing constant.

Equivalently: **posterior ∝ likelihood × prior**.

This is just [Bayes' theorem](./01-probability-foundations.md) applied to *parameters* rather than events. The Bayesian move is treating θ as a random variable with a distribution, not a fixed unknown.

### Frequentist vs Bayesian — The Two Mindsets

| | Frequentist | Bayesian |
|---|---|---|
| Parameter θ | Fixed unknown | Random variable |
| What's random | The data, given θ | θ, given data |
| Output | Point estimate, CI, p-value | Posterior distribution |
| 95% CI | "95% of such intervals would contain true θ in repeated sampling" | **Credible interval:** "95% probability θ is in this interval given the data" |
| Prior info | No formal mechanism | Encoded in prior |
| Interview reflex | "p-value, t-test, A/B test" | "posterior, credible interval, Thompson sampling" |

Modern practice blends both. Frequentist for regulated/standardized inference; Bayesian for sequential decisions, hierarchical models, and incorporating prior knowledge.

### MLE vs MAP

**Maximum Likelihood Estimate (MLE):**

$$\hat{\theta}_{\text{MLE}} = \arg\max_\theta P(\text{data} \mid \theta)$$

The parameter that makes the observed data most likely. No prior.

**Maximum A Posteriori (MAP):**

$$\hat{\theta}_{\text{MAP}} = \arg\max_\theta P(\theta \mid \text{data}) = \arg\max_\theta P(\text{data} \mid \theta) \cdot P(\theta)$$

MLE + prior. With a uniform prior, MAP = MLE.

**Connection to regularization:**

- A Gaussian prior on β corresponds to **Ridge regression** (L2). MAP under N(0, τ²I) prior = OLS + λ‖β‖².
- A Laplace prior on β corresponds to **Lasso** (L1). MAP under Laplace prior = OLS + λ‖β‖₁.

So regularized regression is secretly Bayesian. See [Regression](./05-regression.md).

### Conjugate Priors

A prior is **conjugate** to a likelihood if the posterior has the same form as the prior. Conjugate pairs you should know:

| Likelihood | Conjugate prior | Posterior |
|---|---|---|
| Bernoulli / Binomial | **Beta(α, β)** | Beta(α + successes, β + failures) |
| Poisson | **Gamma(α, β)** | Gamma(α + Σx, β + n) |
| Normal (known σ²) | **Normal** | Normal (mean is precision-weighted average) |
| Normal (unknown σ²) | Normal-Inverse-Gamma | Same family |
| Categorical | Dirichlet | Dirichlet (add counts) |

The **Beta-Bernoulli conjugate** is the workhorse of [Bayesian A/B testing](./04-ab-testing.md):

```python
from scipy.stats import beta
# Start with uninformative Beta(1, 1) prior (uniform)
# Observe 80 conversions in 1000 trials → posterior Beta(81, 921)
post = beta(81, 921)
print(post.mean(), post.interval(0.95))  # estimate + 95% credible interval
```

### Computing Posteriors (when conjugacy fails)

For non-conjugate problems, the integral in the denominator is intractable. Approaches:

- **Markov Chain Monte Carlo (MCMC)** — sample from the posterior. Tools: PyMC, Stan, NumPyro.
- **Variational Inference (VI)** — approximate the posterior with a simpler distribution. Faster than MCMC, less exact.
- **Hamiltonian Monte Carlo (HMC) / NUTS** — modern MCMC; what Stan/PyMC use by default.

```python
import pymc as pm
with pm.Model() as model:
    p = pm.Beta('p', alpha=1, beta=1)
    obs = pm.Bernoulli('obs', p=p, observed=data)
    trace = pm.sample(2000)
```

### Bayesian Credible Interval vs Frequentist CI

- **Credible:** "Given the data and prior, 95% probability θ is in [a, b]." (What people *want* a CI to mean.)
- **Frequentist:** "If we repeated the experiment many times, 95% of computed intervals would contain θ." (What CIs actually mean.)

Credible intervals are easier to communicate. Frequentists object that they depend on the prior.

### Why Bayesian Wins for Sequential Decisions

Bayesian posteriors update naturally as data arrives. You can peek anytime, make decisions on partial data, and have a clean probabilistic statement at each step. Frequentist sequential testing requires careful correction. Bayesian A/B testing and **Thompson sampling** for multi-armed bandits exploit this.

### Hierarchical (Multilevel) Models

Real-world data has structure: students within schools, users within markets, products within categories. **Hierarchical models** share information across groups by placing a prior on the group-level parameters.

Example: per-user conversion rate with **partial pooling**:

$$p_u \sim \text{Beta}(\alpha, \beta), \quad y_u \sim \text{Binomial}(n_u, p_u)$$

with α, β estimated from data. Users with few observations get pulled toward the global mean (regularized); users with many observations stay close to their MLE. This is the canonical "shrinkage" estimator — the **James-Stein** intuition.

Used in: A/B test segments, recommender warm-start, ML feature smoothing.

---

## CAUSAL INFERENCE

### The Core Problem

**Correlation does not imply causation.** Statistics describes; causation prescribes. To intervene on the world (ship a feature, prescribe a drug), we need a *causal* estimate, not a *correlational* one.

### Potential Outcomes (Rubin Causal Model)

For each unit i and treatment T ∈ {0, 1}, define:

- $Y_i(1)$ = outcome if treated.
- $Y_i(0)$ = outcome if not.

The **individual treatment effect** is $\tau_i = Y_i(1) - Y_i(0)$. We can never observe both for the same unit (the "fundamental problem of causal inference"). We observe only one — the **observed outcome** depends on the assigned treatment.

The **average treatment effect (ATE)** is $E[Y(1) - Y(0)]$.

### Why Randomization Solves Everything

Under randomized assignment, T ⊥ (Y(0), Y(1)). So:

$$E[Y \mid T=1] - E[Y \mid T=0] = E[Y(1)] - E[Y(0)] = \text{ATE}$$

The difference of observed means is an unbiased estimate of the ATE. **This is why A/B tests work.** See [A/B Testing](./04-ab-testing.md).

### Confounders

A confounder is a variable that affects both the treatment and the outcome.

```
        Z (confounder, e.g. income)
       / \
      ↓   ↓
      T → Y
   (drug)  (recovery)
```

In observational data (no randomization), comparing E[Y | T=1] vs E[Y | T=0] confounds the causal effect of T with the effect of Z. **You don't know what you're measuring.**

### Colliders — The Trap

A collider is a variable that's affected by *both* the treatment and the outcome:

```
T → C ← Y
```

**Conditioning on (controlling for) a collider induces a spurious association between T and Y, even if they're truly independent.** Classic example: Berkson's paradox, "good-looking people are jerks" sampling bias.

> **Rule:** control for confounders. Do **not** control for colliders. Do not "throw everything into the regression."

### Simpson's Paradox

A trend within subgroups can reverse when the subgroups are combined. Canonical example:

A drug looks effective overall: 60% recovery treated vs 50% untreated. But split by gender:
- Men: 70% treated, 80% untreated (drug looks bad).
- Women: 20% treated, 40% untreated (drug looks bad).

How can the aggregate flip the sign? **Different mixing proportions** — the treated group had more men (who recover more easily). Gender is a confounder; not controlling for it inverts the conclusion.

The lesson: **aggregate statistics can mislead in the presence of confounders.** Always think about what's stratifying your population.

### Identification Strategies When You Can't Randomize

Observational data + domain knowledge can sometimes identify causal effects:

#### Regression Adjustment / Conditioning

Control for measured confounders by including them in a regression. Assumes you've measured *all* relevant confounders (the "no unmeasured confounders" / NUC assumption — strong, often wrong).

#### Propensity Score Methods

Estimate $\pi(x) = P(T = 1 \mid X = x)$ — the probability of treatment given covariates. Then:

- **Matching:** pair treated units with control units of similar propensity. Compare matched pairs.
- **Inverse Propensity Weighting (IPW):** weight observations by $1/\pi$ or $1/(1-\pi)$ to make the treated/control populations look exchangeable.

Both still rely on NUC.

#### Difference-in-Differences (DiD)

When a treatment is rolled out at a specific time to part of the population:

$$\text{DiD} = (Y_{T,\text{post}} - Y_{T,\text{pre}}) - (Y_{C,\text{post}} - Y_{C,\text{pre}})$$

Subtracts unit-specific baseline and time trend. Assumes **parallel trends** (treated and control would have followed parallel paths absent treatment). Common in economics and product analytics for region-based rollouts.

#### Instrumental Variables (IV)

When you can find a variable Z that:
- Affects T,
- Affects Y *only through T* (the exclusion restriction),
- Is independent of confounders.

Then 2-stage least squares (2SLS) recovers the causal effect. Classic example: distance to college as IV for years of schooling on earnings (Angrist).

#### Regression Discontinuity (RD)

When treatment assignment has a sharp cutoff in some variable (test score → admission → outcome). Compare units just above vs just below the cutoff; they're "as good as randomized."

#### Synthetic Control

Construct a weighted combination of control units to mimic the treated unit's pre-treatment trajectory; the post-treatment gap is the estimated effect. Used for: state-level policy evaluation (Abadie 2003, California Prop 99), platform-level marketing tests.

### Causal DAGs (Pearl)

Draw a directed graph encoding your causal beliefs. Then mechanical rules (the **back-door criterion**, **front-door criterion**, **do-calculus**) tell you which variables to control for.

The key insight: **causal inference requires assumptions** that can't be tested from data alone. The DAG is where you make those assumptions explicit.

Pearl's *The Book of Why* is the accessible intro. *Causal Inference: The Mixtape* (Cunningham) and *Mostly Harmless Econometrics* (Angrist & Pischke) are the practical complement.

### Double Machine Learning (DML)

Combine ML for nuisance functions with classical causal-inference identification. The recipe (Chernozhukov et al., 2018):

1. Use ML to predict T from X (propensity).
2. Use ML to predict Y from X (outcome regression).
3. Run a partialled-out regression on the residuals.

Gives unbiased ATE estimates even with high-dimensional X, leveraging modern ML for the nuisance steps. Used at Microsoft (EconML library), Uber, Booking.

### Causal Forests

A random forest variant that estimates *heterogeneous* treatment effects — how the effect varies across subgroups. EconML and the `grf` R package. Used for personalization.

---

## Worked Example — A Bayesian A/B Test

**Setup.** Control: 80 / 1000 conversions. Variant: 100 / 1000.

```python
from scipy.stats import beta
import numpy as np

# Uniform prior Beta(1, 1)
post_C = beta(1 + 80, 1 + 920)
post_V = beta(1 + 100, 1 + 900)

# Monte Carlo: P(V > C | data)
samples_C = post_C.rvs(100_000)
samples_V = post_V.rvs(100_000)
p_v_better = (samples_V > samples_C).mean()
expected_lift = (samples_V - samples_C).mean()
ci_lift = np.quantile(samples_V - samples_C, [0.025, 0.975])

print(f"P(V > C) = {p_v_better:.3f}")
print(f"Expected absolute lift = {expected_lift:.4f}")
print(f"95% credible interval for lift = {ci_lift}")
# P(V > C) ≈ 0.93
# Expected lift ≈ 0.02
# 95% CI ≈ [-0.005, +0.045]
```

**Interpretation.** 93% posterior probability variant beats control. Expected absolute lift 2 percentage points. 95% credible interval for the lift includes zero (barely).

A frequentist two-proportion z-test gave p ≈ 0.118 — "not significant." Same data, both views are correct. The Bayesian phrasing gives the PM something **actionable** ("93% chance it's better; expected lift +2pp"), while the frequentist phrasing gives a **calibrated guarantee** ("can't reject H₀ at α = 0.05").

Both are right; what to ship is a decision about risk tolerance.

---

## Common Interview Gotchas

**Q: What's the difference between frequentist and Bayesian statistics?**
A: Frequentist treats θ as fixed and data as random; Bayesian treats θ as random with a prior. Different interpretations of probability, both useful.

**Q: What's MAP vs MLE?**
A: MAP includes a prior; MLE doesn't. With uniform prior, they coincide. Ridge regression = MAP under Gaussian prior; Lasso = MAP under Laplace prior.

**Q: Why use a conjugate prior?**
A: Closed-form posterior — no MCMC needed. Beta-Bernoulli for proportions, Gamma-Poisson for counts.

**Q: What's a credible interval?**
A: Given the data and prior, the parameter is in this range with stated probability. Direct probabilistic statement, unlike a frequentist CI.

**Q: When would you prefer Bayesian A/B over frequentist?**
A: For sequential decisions, when incorporating prior history, when you want "probability variant is better" as the output rather than a p-value.

**Q: What's a confounder?**
A: A variable that affects both treatment and outcome, biasing the observed association. Control for it.

**Q: What's a collider?**
A: A variable affected by both treatment and outcome. **Do not** control for it — doing so induces spurious correlation.

**Q: What's Simpson's paradox?**
A: An aggregate trend reversing in subgroups (or vice versa). Usually a confounding variable with unbalanced subgroup proportions. Always stratify when you suspect heterogeneity.

**Q: What's propensity score matching?**
A: Estimate P(T=1 | X), then match treated and control units with similar propensity to approximate randomization. Assumes no unmeasured confounders.

**Q: When does difference-in-differences work?**
A: When you have pre/post observations for both treated and untreated groups and can argue parallel trends would have held without treatment.

**Q: What's the fundamental problem of causal inference?**
A: We never observe both Y(0) and Y(1) for the same unit. Causal inference is missing-data imputation under assumptions.

---

## Interview-Ready Cheat Sheet

### One-liners to memorize

- **Posterior ∝ likelihood × prior.**
- **MAP = MLE + prior.**
- **Conjugate priors give closed-form posteriors.** Beta-Bernoulli is the workhorse.
- **Credible interval ≠ confidence interval** — different interpretations.
- **Frequentist: parameter fixed, data random. Bayesian: parameter random.**
- **Correlation ≠ causation.** RCTs (randomization) are the gold standard.
- **Control for confounders; don't control for colliders.**
- **Simpson's paradox = unconfounded subgroup analysis flipping aggregate conclusion.**
- **Propensity scores / IV / DiD / RD / synthetic control** are non-experimental causal-identification tools.
- **Hierarchical Bayes (partial pooling) is the right framework for grouped data.**

### Decision pairs

- **Frequentist vs Bayesian:** standardized inference vs sequential / prior-incorporating decisions.
- **MLE vs MAP vs full Bayes:** point estimate vs regularized point estimate vs full posterior.
- **A/B test vs observational:** clean causal estimate (expensive, slow) vs use existing data (cheap, requires identification assumptions).
- **Regression-adjusted vs propensity-matched:** parametric form on Y vs parametric form on T.
- **DiD vs IV vs RD:** different identification assumptions; each fits a different problem shape.

### Top Q&A summary

**"Bayesian or frequentist?"** → Both. Frequentist for standardized communication and regulatory inference; Bayesian for sequential decisions and uncertainty quantification.

**"How do you estimate causal effects?"** → Randomize when possible (RCT, A/B). Else: propensity score, DiD, IV, RD, synthetic control. State your identifying assumption.

**"How do you handle Simpson's paradox?"** → Stratify by suspected confounders; visualize per stratum; consider DAG analysis.

---

## Resources & Links

### Bayesian
- [*Statistical Rethinking* (McElreath)](https://xcelab.net/rm/statistical-rethinking/) — best Bayesian textbook.
- [*Bayesian Data Analysis* (Gelman et al.)](http://www.stat.columbia.edu/~gelman/book/) — the canonical reference. Free PDF.
- [PyMC docs](https://www.pymc.io/) and [tutorials](https://www.pymc.io/projects/examples/).
- [Stan docs](https://mc-stan.org/users/documentation/).
- [Probabilistic Programming & Bayesian Methods for Hackers](https://github.com/CamDavidsonPilon/Probabilistic-Programming-and-Bayesian-Methods-for-Hackers) — free book.

### Causal
- *The Book of Why* (Pearl) — accessible.
- *Causal Inference: The Mixtape* (Cunningham) — practical, free online.
- *Mostly Harmless Econometrics* (Angrist & Pischke) — applied microeconometrics.
- *Causal Inference: What If* (Hernán & Robins) — biostatistics flavor, [free PDF](https://www.hsph.harvard.edu/miguel-hernan/causal-inference-book/).
- [Microsoft EconML](https://github.com/microsoft/EconML) — causal ML library.
- [DoWhy](https://github.com/py-why/dowhy) — Python causal inference framework.

### Companion files
- [Probability Foundations](./01-probability-foundations.md) — Bayes' theorem.
- [Hypothesis Testing](./03-hypothesis-testing.md) — the frequentist counterpart.
- [A/B Testing](./04-ab-testing.md) — randomization as causal identification.
- [Regression](./05-regression.md) — regularization is hidden Bayesian.

---

*This is the last file in the statistics section. Combined with the others, you have the full inferential + ML statistics surface for DS interviews.*
