# A/B Testing & Experimentation

**Phase:** 2 (Applied inference)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Hypothesis Testing](./03-hypothesis-testing.md) · [ML Metrics](./06-ml-metrics.md) · [Bayesian & Causal](./07-bayesian-and-causal.md)

---

## Why This File Exists

A/B testing is the most-tested practical statistics topic in tech DS/ML interviews. Every product company runs experiments. The interview question is rarely "what's a t-test" — it's "design an A/B test for [feature]." That answer combines hypothesis testing, power analysis, randomization, metric selection, multiple comparisons, and pitfalls (peeking, novelty, network effects).

---

## BEGINNER — The A/B Test Framework

### What an A/B Test Is

Randomly split users into two groups. **Control** sees the current experience; **treatment** sees the change. Compare a chosen **metric** between groups, decide if the difference is real.

The randomization is what makes it causal: any *systematic* between-group difference at end-of-test must come from the treatment (modulo statistical noise).

### The Six-Step Design Process

1. **State the hypothesis** in business terms ("the new checkout will increase conversion") and statistical terms (H₀: p_T = p_C).
2. **Pick the primary metric** — one number. Pick guardrails too.
3. **Pick α and the desired power.** Standard: α = 0.05, power = 0.80.
4. **Estimate the minimum detectable effect (MDE)** — smallest lift worth caring about.
5. **Compute sample size** required.
6. **Randomize, run, measure, decide.**

Skipping step 4 — the most common mistake — leads to underpowered tests where "no significant difference" doesn't actually rule out a meaningful effect.

### Choosing the Metric — OEC vs Guardrails

- **OEC (Overall Evaluation Criterion)** — the primary metric. Should be:
  - **Sensitive** (moves on real changes) but stable.
  - **Aligned with long-term value** (not a vanity metric).
  - **Computable quickly** so the experiment doesn't drag.
- **Guardrails** — metrics that should NOT regress, even if OEC improves: latency, error rate, revenue per user, retention. If the treatment kills the home page latency, you don't ship even if conversion jumps.

**Avoid metrics that are too noisy** (per-user revenue often is) or **lagging** (90-day retention from a 7-day test is impossible).

### Randomization Unit

Pick the unit at which you randomize. Common choices:

| Unit | Pros | Cons |
|---|---|---|
| User | Stable experience, clean independence | Cookie/session-tracking issues |
| Session | More samples, less stable | Same user can see both arms |
| Request | Maximum power | Wildly inconsistent UX; usually wrong for product changes |
| Cluster (region, city, market) | Avoids network spillovers | Few clusters → very low power |

For ML model A/B tests, **cluster randomization** is often required because a recommendation model affects what other users see (network effect). See *interference* below.

---

## INTERMEDIATE — Power, Sample Size, and Effect Size

### Why Sample Size Matters

Imagine you're testing a +0.5% lift in conversion. Conversion is 5%. Per-arm sample sizes:

| MDE (relative lift) | n per arm (α=0.05, power=0.80) |
|---|---|
| +10% (5.0% → 5.5%) | ~31,000 |
| +5% (5.0% → 5.25%) | ~123,000 |
| +2% (5.0% → 5.1%) | ~770,000 |
| +1% (5.0% → 5.05%) | ~3,080,000 |

**Halving the MDE quadruples sample size** (n ∝ 1/MDE²). This is why detecting "small but real" effects is genuinely hard.

### Sample-Size Formula for Two Proportions

For control rate p_C and minimum detectable effect δ = p_T − p_C:

$$n \approx \frac{(z_{\alpha/2} + z_\beta)^2 \cdot \left[p_C(1-p_C) + p_T(1-p_T)\right]}{\delta^2}$$

For α = 0.05, power = 0.80: z_{α/2} = 1.96, z_β = 0.84, so (z_{α/2} + z_β)² ≈ 7.85.

```python
from statsmodels.stats.power import NormalIndPower
from statsmodels.stats.proportion import proportion_effectsize

p1, p2 = 0.05, 0.055
effect = proportion_effectsize(p1, p2)             # Cohen's h
n = NormalIndPower().solve_power(effect_size=effect, alpha=0.05, power=0.8)
print(n)   # per arm
```

### Sample-Size for Two Means

$$n \approx \frac{2\sigma^2 (z_{\alpha/2} + z_\beta)^2}{\delta^2}$$

You need a variance estimate σ² (usually from historical data on the metric).

```python
from statsmodels.stats.power import TTestIndPower
n = TTestIndPower().solve_power(effect_size=0.2,   # Cohen's d
                                 alpha=0.05, power=0.80)
```

### What Determines Your MDE

- **Smaller variance** → smaller MDE for the same n.
- **Larger n** → smaller MDE.
- **Lower α** or **higher power** → larger MDE.

If business says "we need to detect a 1% lift," and you can only get 100k users in two weeks, do the math: if 100k can detect a 5% lift at best, **you can't run this test.** Either lower the bar (3-month test, bigger MDE) or use variance-reduction techniques (CUPED, stratification).

### Variance Reduction — CUPED

CUPED (Microsoft) uses **pre-period covariates** (e.g. the user's prior week conversion rate) to remove between-user variance from the metric, often reducing variance by 30–50% — equivalent to running the same test on a much larger sample.

```python
# Conceptual sketch
# Y' = Y - θ * (X - mean(X))  where X is the pre-period covariate
theta = cov(Y, X) / var(X)
Y_adjusted = Y - theta * (X - X.mean())
# Then run the standard t-test on Y_adjusted
```

Almost every mature experimentation platform implements CUPED. Worth name-dropping.

---

## INTERMEDIATE — Analysis and Pitfalls

### The Analysis Step

For a conversion-rate test, run a two-proportion z-test or χ² (equivalent). For a continuous metric, run Welch's t-test (or bootstrap if non-normal). See [Hypothesis Testing](./03-hypothesis-testing.md).

Always report:
- Observed lift (absolute and relative).
- **95% confidence interval** for the lift.
- p-value.
- Power achieved (post-hoc).
- Sample sizes per arm.

### Peeking Problem

If you check the test every day and stop when p < 0.05, **your true Type I rate is much higher than 0.05** — sometimes 20%+. You're sequentially sampling and stopping at the noisiest moment.

Solutions:
- **Don't peek.** Decide n in advance and only analyze at the end.
- **Sequential testing** with corrections (Pocock, O'Brien-Fleming, mSPRT, always-valid p-values from Howard et al. 2021).
- **Bayesian sequential testing** — posterior probabilities can be inspected anytime, with care.

### Sample Ratio Mismatch (SRM)

If you intended a 50/50 split but observed 50.5/49.5 of 1M users, run a χ² test of the bucket counts against the design. **SRM p < 0.001 is a red flag** — your randomization, bucketing, or logging is broken. Don't trust the test result; investigate the pipeline.

### Multiple Metrics and Tests

Running 20 metrics with α = 0.05 each → 64% chance of at least one false positive. Two corrections:

- Pre-declare your **primary OEC** + a small set of guardrails. Apply Bonferroni or BH-FDR only to the secondary metrics.
- Don't claim a "win" based on a secondary metric that's significant when the primary isn't.

### Novelty and Primacy Effects

Users behave differently when they see something new. Novelty: spike that decays. Primacy: regression as users get used to the old. **Don't make decisions on the first 24–48 hours of a test.** Run for at least one full weekly cycle (often two), and check whether the effect is stable.

### Network Effects / Interference (the SUTVA violation)

The Stable Unit Treatment Value Assumption requires that one unit's treatment doesn't affect another's outcome. Violations:

- **Two-sided marketplaces** — Uber A/B: if treatment riders book more, drivers are scarcer for control riders.
- **Social features** — testing a notification: control sees fewer messages because their treatment friends got the better experience.
- **Recommendation systems** — treatment model affects content others see.

Mitigations: **cluster randomization** (whole city/region in the same arm), **time-based switchbacks** (alternate arms by day), **synthetic control / quasi-experiment** designs.

### Choosing α and Power Wisely

α = 0.05 / power = 0.80 are conventions, not laws.

- **Lower α (0.01)** for high-stakes shipped changes that are hard to roll back, or when running many tests.
- **Higher power (0.90)** when missing a real effect is expensive (delays product strategy by months).
- Both increase sample size — make the trade explicit.

---

## ADVANCED — Beyond Simple A/B

### Multi-Armed Bandits

Instead of equal-split A/B, dynamically allocate traffic toward better-performing arms. Tradeoff: less rigor, faster learning, less regret (lost value during the test). Tools: Thompson sampling, epsilon-greedy, UCB.

Use bandits when:
- The cost of showing a losing arm is high (e.g. ad headlines).
- You don't strictly need a p-value — you need to learn fast.

Stick to classical A/B when:
- You need strong statistical guarantees (regulators, executives).
- You want to estimate the effect size precisely, not just pick a winner.

### Stratification and CUPAC

If you can split users into strata (heavy/medium/light, region, device) before randomization, you can guarantee balance within strata → less between-arm noise → smaller MDE.

CUPAC (Doordash extension of CUPED) uses ML predictions of post-period outcome as the pre-period covariate, often reducing variance further.

### Sequential Testing — Always-Valid p-values

Howard et al. (2021) and the "Always Valid Inference" framework let you check the test anytime without inflating Type I error. Used by Optimizely, statsig, and modern experimentation platforms. The math is non-trivial but the implementation is just a library call.

### Holdouts and Long-Term Effects

For changes you suspect have long-term effects (recommendation models, ranking changes), run a **persistent holdout** — a small slice of users who never see the change for 6+ months. Lets you measure long-term impact even after the test ends.

### Switchback Tests

For marketplace / network effects: assign the *whole platform* to treatment for one time window, control for the next. Alternate. Analyze by time blocks. Common at Uber, Lyft, Doordash.

### A/A Tests

Run treatment vs treatment (identical experiences). The "significant" rate across many A/A tests should be α. If it's 10% on a nominal-5% test, your pipeline has a problem (correlated users, broken randomization, autocorrelated metrics).

A/A tests are how mature platforms validate that their experimentation infra produces calibrated p-values.

### Bayesian A/B Testing

Frame: instead of "is the lift significant?" ask "what's P(treatment > control | data)?" via posterior distributions.

```python
# Beta-Bernoulli conjugate model
from scipy.stats import beta
post_C = beta(1 + 80, 1 + 920)        # conversion rate posterior for control
post_T = beta(1 + 100, 1 + 900)        # for treatment
# P(T > C) via Monte Carlo
samples_C = post_C.rvs(100000)
samples_T = post_T.rvs(100000)
print((samples_T > samples_C).mean())  # e.g. 0.94
```

Advantages:
- Direct probabilistic statements: "94% probability variant is better."
- Sequential decisions are natural.
- Easy to incorporate priors (e.g. historical experiment results).

Disadvantages:
- "What's a reasonable prior?" debates.
- Less standard in regulated environments.

See [Bayesian & Causal](./07-bayesian-and-causal.md) for the deeper version.

---

## Worked Example — Designing a Real A/B Test

**Problem:** PM wants to test a new checkout button. Current conversion is 5.0%. They want to detect a ≥0.25pp improvement (5.00% → 5.25%, relative +5%) with 80% power at α = 0.05.

**Step 1 — Sample size.**

```python
from statsmodels.stats.power import NormalIndPower
from statsmodels.stats.proportion import proportion_effectsize
effect = proportion_effectsize(0.05, 0.0525)
n = NormalIndPower().solve_power(effect_size=effect, alpha=0.05, power=0.80)
# n ≈ 123,000 per arm; 246,000 total
```

**Step 2 — Duration.**

Traffic is 50K users/day → 246K / 50K ≈ 5 days. Round to **two full weeks** (avoid day-of-week effects). Pre-commit.

**Step 3 — Metrics.**

- **OEC:** conversion rate (binary per user).
- **Guardrails:** page load latency, refund rate, 7-day retention.

**Step 4 — Randomization.**

User-id hash mod 100 → 50 buckets to control, 50 to treatment. Persistent across sessions.

**Step 5 — SRM check on day 1.**

Counts: 49,000 vs 51,000. χ² p = 0.10 — within noise. Continue.

**Step 6 — Day 7 peek.**

Conversion: control 5.0%, variant 5.4%. p = 0.03. **DO NOT STOP.** Pre-committed to 14 days.

**Step 7 — End of day 14.**

Conversion: control 5.05%, variant 5.20%. n_C = 350K, n_T = 350K.

```python
from statsmodels.stats.proportion import proportions_ztest, confint_proportions_2indep
stat, p = proportions_ztest([0.0520*350000, 0.0505*350000], [350000, 350000])
# p ≈ 0.005, significant
# CI for absolute lift: [+0.05pp, +0.25pp]
```

Significant. Relative lift +3.0%. Guardrails: latency unchanged; refund/retention stable.

**Step 8 — Ship decision.**

Above MDE? Just barely (3% rel vs 5% target) — note in the writeup that this is below the originally targeted effect but the CI excludes zero.

**Step 9 — Followup.**

Run a 5% holdout for 90 days to confirm the lift persists.

That's a senior answer.

---

## Common Interview Gotchas

**Q: How do you design an A/B test for [feature]?**
A: Walk the six-step framework. Always mention MDE / power analysis up front, then randomization unit, then metrics, then duration, then guardrails and pitfalls.

**Q: How long should you run an A/B test?**
A: Until you reach your pre-computed sample size **AND** at least one full weekly cycle (typically two). Not until p < 0.05 (peeking inflates Type I rate).

**Q: What's the peeking problem?**
A: Checking the test early and stopping when significant inflates the false-positive rate. Use sequential testing or pre-commit to n.

**Q: What's sample ratio mismatch (SRM)?**
A: When observed split deviates from intended (e.g. 50.5/49.5). Indicates pipeline bug; invalidates results.

**Q: How would you A/B test a recommendation model?**
A: Network effects matter. Consider cluster randomization (geo-based), switchback tests, or accept that user-level randomization estimates a biased treatment effect (often acceptable in practice with a large user base).

**Q: When would you use a multi-armed bandit instead of an A/B test?**
A: When the cost of showing a losing arm is high and you don't need precise effect-size estimates. Bandits are about minimizing regret; A/B is about hypothesis testing.

**Q: What's a guardrail metric?**
A: A metric that mustn't regress even if the OEC improves (latency, error rate, retention). Protects against winning a battle, losing the war.

**Q: Test shows p = 0.04, but lift is only 0.1%. Ship?**
A: Probably not. Significant ≠ practical. Compare effect size to ship cost, downstream risk, and opportunity cost of more impactful experiments.

**Q: Why is power 0.8 by convention?**
A: Cohen's recommendation. Trade-off between sample size and Type II error rate. There's nothing magic about it; you can pick 0.9 or 0.95 if stakes are high.

**Q: A test has p = 0.50. What does that mean?**
A: Effect could be anywhere; data isn't ruling out H₀. If the test was well-powered, real effect is probably small. Underpowered tests are uninformative.

---

## Interview-Ready Cheat Sheet

### The framework to recite

1. Clarify the business goal.
2. Define OEC + guardrails.
3. Pick MDE, α, power.
4. Compute sample size; convert to duration.
5. Choose randomization unit; verify SRM at runtime.
6. Pre-commit to n; no peeking.
7. Analyze with two-proportion z (or t for continuous); report lift, CI, p, power achieved.
8. Check guardrails; consider novelty / network effects; ship with holdout.

### Quick numbers

- α = 0.05, power = 0.80 → (z_{α/2} + z_β)² = 7.85 ≈ 8.
- Halving MDE → 4× sample size.
- Variance reduction via CUPED: typically 30–50% (2× effective sample).

### Trade-off pairs

- **A/B vs bandit:** rigor + effect size vs faster convergence + less regret.
- **Daily peeking vs commit:** false positives vs occasional missed wins.
- **User-level vs cluster randomization:** power vs network-effect safety.
- **More metrics vs FWER:** more learning vs higher false-positive rate; use BH-FDR on secondaries.
- **Frequentist vs Bayesian:** standardized p-values vs sequential-friendly posteriors.

---

## Resources & Links

- [*Trustworthy Online Controlled Experiments* (Kohavi, Tang, Xu)](https://experimentguide.com/) — the bible.
- [Microsoft Experimentation Platform papers](https://www.microsoft.com/en-us/research/group/experimentation-platform-exp/publications/)
- [CUPED paper (Microsoft, 2013)](http://www.exp-platform.com/Documents/2013-02-CUPED-ImprovingSensitivityOfControlledExperiments.pdf)
- [Booking.com — A/B testing lessons](https://booking.ai/the-7-essential-things-you-must-know-about-the-overall-evaluation-criterion-78a6b9fef3a0)
- [Always-Valid Inference (Howard et al., 2021)](https://arxiv.org/abs/2103.06476)
- [Netflix experimentation blog](https://netflixtechblog.com/tagged/experimentation)
- [Statsig docs — modern experimentation](https://www.statsig.com/blog)
- [Eppo / GrowthBook docs](https://docs.getgrowthbook.io/)

### Companion files
- [Hypothesis Testing](./03-hypothesis-testing.md) — the underlying machinery.
- [Bayesian & Causal](./07-bayesian-and-causal.md) — Bayesian A/B, causal inference.
- [Probability Foundations](./01-probability-foundations.md) — CLT, distributions.

---

*Next: [Regression Fundamentals](./05-regression.md) — coefficient p-values are hypothesis tests too.*
