# Regression Fundamentals — OLS, Assumptions, Logistic, Regularization

**Phase:** 2 (Modeling)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Hypothesis Testing](./03-hypothesis-testing.md) · [ML Metrics](./06-ml-metrics.md) · [Probability Foundations](./01-probability-foundations.md)

---

## Why This File Exists

Linear and logistic regression are the most-tested ML models in DS interviews — not because they're cutting-edge but because they're a complete diagnostic of statistical thinking. Coefficient p-values, R², assumptions, multicollinearity, regularization — every concept maps cleanly to "do you understand uncertainty in a fitted model?"

---

## BEGINNER — Linear Regression

### The Model

Linear regression assumes:

$$y = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \ldots + \beta_p x_p + \varepsilon, \quad \varepsilon \sim N(0, \sigma^2)$$

In matrix form: $y = X\beta + \varepsilon$ where X is the n × (p+1) design matrix (first column is ones for the intercept).

We fit by **Ordinary Least Squares (OLS)** — minimize sum of squared residuals:

$$\hat{\beta} = \arg\min_\beta \sum_i (y_i - x_i^\top \beta)^2$$

The closed-form solution (the formula every interviewer wants you to know):

$$\hat{\beta} = (X^\top X)^{-1} X^\top y$$

This is the normal equation. It exists when X^TX is invertible — which fails when columns are perfectly collinear.

### Why Least Squares?

Three justifications, all useful in interviews:

1. **Maximum Likelihood under Gaussian noise.** If errors are normal, OLS = MLE.
2. **Gauss-Markov theorem.** Under (linearity, independence, homoskedasticity, mean-zero errors), OLS is the **Best Linear Unbiased Estimator** (BLUE) — lowest variance among unbiased linear estimators.
3. **Geometric** — projects y onto the column space of X, minimizing Euclidean distance.

### R² (Coefficient of Determination)

$$R^2 = 1 - \frac{\text{SS}_{\text{res}}}{\text{SS}_{\text{tot}}} = 1 - \frac{\sum (y_i - \hat{y}_i)^2}{\sum (y_i - \bar{y})^2}$$

Fraction of variance in y explained by the model. Always between 0 and 1 for OLS on training data.

**Adjusted R²** penalizes adding useless predictors:

$$R^2_{\text{adj}} = 1 - (1 - R^2) \cdot \frac{n - 1}{n - p - 1}$$

Always use adjusted R² when comparing models with different numbers of predictors.

### R² Caveats

- **R² doesn't measure predictive accuracy.** A model can have high R² and still generalize poorly.
- **R² can be high for terrible models** (overfit, spurious correlations).
- **R² can be near 0 for excellent models** if the signal-to-noise ratio is low.

For prediction, **out-of-sample R², cross-validated R², RMSE, MAE** are what matter. See [ML Metrics](./06-ml-metrics.md).

### Coefficient Interpretation

For a model $y = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + ...$:

> "Holding all other predictors fixed, a 1-unit increase in $x_1$ is associated with a $\beta_1$-unit change in $y$."

The "holding others fixed" is critical and easy to forget. β₁ in a single-variable regression often differs from β₁ in a multi-variable regression — because controlling for other variables changes the partial effect.

### Coefficient p-Values and Confidence Intervals

For each β_j, test H₀: β_j = 0 using a t-test:

$$t_j = \frac{\hat{\beta}_j}{\text{SE}(\hat{\beta}_j)}, \quad \text{df} = n - p - 1$$

Small p-value → coefficient significantly different from 0. **This is the same machinery as [Hypothesis Testing](./03-hypothesis-testing.md)** — every "p-value column" in a regression output is a t-test.

The 95% CI for β_j is $\hat{\beta}_j \pm t_{0.025, n-p-1} \cdot \text{SE}(\hat{\beta}_j)$.

---

## INTERMEDIATE — The Five OLS Assumptions

Memorize these. Senior interviews always ask.

### 1. Linearity

$E[y | X] = X\beta$. The relationship is linear in the parameters (not necessarily in the inputs — you can include $x^2$ as a column).

**Check:** residual vs fitted plot. If you see a curve, the relationship isn't linear in your variables.

**Fix:** add polynomial terms, log-transform, splines, or move to a non-linear model.

### 2. Independence of Errors

$\varepsilon_i$ are mutually independent.

**Check:** Durbin-Watson statistic; residual autocorrelation plots.

**Fix:** time-series models (ARIMA, regression with autocorrelated errors), clustered SEs for hierarchical data.

### 3. Homoskedasticity

$\text{Var}(\varepsilon_i)$ is constant across i.

**Check:** residual vs fitted plot; Breusch-Pagan test.

**Fix:** **heteroskedasticity-robust standard errors** (HC0/HC1/HC3), weighted least squares, or log-transform y. The coefficient estimates are still unbiased; only the SEs are wrong.

### 4. Normality of Errors

$\varepsilon_i \sim N(0, \sigma^2)$.

**Check:** Q-Q plot of residuals; Shapiro-Wilk test.

**Fix:** This matters for **small samples**. For large n, CLT gives you approximately correct inference even with non-normal errors.

### 5. No Perfect Multicollinearity

X^TX is invertible — no column is a linear combo of others.

**Check:** VIF (Variance Inflation Factor):

$$\text{VIF}_j = \frac{1}{1 - R_j^2}$$

where $R_j^2$ is from regressing $x_j$ on all other predictors. Rules of thumb: VIF > 5 (concerning), VIF > 10 (problem).

**Fix:** drop one of the correlated variables, use PCA, use regularization (Ridge handles multicollinearity gracefully).

### Why Multicollinearity Matters

Coefficient estimates become unstable — small data changes produce wildly different βs, big SEs, low t-statistics, but **predictions can still be fine**. The model "knows" something is going on but can't attribute it to a specific variable. If you care about *interpretation* (which variable matters?), multicollinearity is a problem. If you only care about *prediction*, less so.

### The Most Common Diagnostic Plots

1. **Residuals vs fitted.** Looking for: no pattern (good), curve (non-linear), funnel (heteroskedasticity).
2. **Q-Q plot of residuals.** Looking for: points on the diagonal (normal errors).
3. **Scale-location plot.** Same as #1 but with √|residuals| — easier funnel detection.
4. **Residuals vs leverage.** Looks for influential points; Cook's distance > 1 is concerning.

---

## INTERMEDIATE — Logistic Regression

### The Setup

When y is binary (0/1), linear regression is wrong (predictions outside [0,1], errors not normal). Logistic regression models:

$$P(y = 1 \mid x) = \sigma(x^\top \beta), \quad \sigma(z) = \frac{1}{1 + e^{-z}}$$

σ is the **sigmoid** (a.k.a. logistic) function. Equivalently, the **log-odds** is linear in x:

$$\log \frac{P(y=1)}{P(y=0)} = x^\top \beta$$

The left side is the **logit**. So logistic regression is a linear model on the logit scale.

### Fitting — Maximum Likelihood

No closed form. Use iteratively reweighted least squares (IRLS) or gradient descent on the log-likelihood:

$$\ell(\beta) = \sum_i \left[y_i \log p_i + (1 - y_i) \log(1 - p_i)\right]$$

where $p_i = \sigma(x_i^\top \beta)$. The negative of this is **cross-entropy loss / log loss** — the standard classification loss.

### Coefficient Interpretation

A 1-unit increase in $x_j$ multiplies the **odds** of y=1 by $e^{\beta_j}$. So:

- $e^{\beta_j} = 1$ → no effect.
- $e^{\beta_j} = 2$ → doubles the odds.
- $e^{\beta_j} = 0.5$ → halves the odds.

This is the **odds ratio** interpretation, and it's the natural way to talk about effect sizes in logistic regression.

### Multinomial / Softmax Regression

For k > 2 classes:

$$P(y = k \mid x) = \frac{e^{x^\top \beta_k}}{\sum_j e^{x^\top \beta_j}}$$

The softmax function. Same idea as logistic but generalized. Used as the final layer of every classification neural network.

### When NOT to Use Logistic Regression

- Strong non-linear interactions — tree-based models or NNs win.
- High-dimensional sparse data (text) — linear is fine but use Lasso for selection.
- When you don't need calibrated probabilities — pick the model that maximizes the metric you care about. See [ML Metrics](./06-ml-metrics.md).

---

## ADVANCED — Regularization

### Why Regularize

OLS minimizes training error and can overfit when:
- p (number of features) approaches n (sample size).
- Features are collinear.
- You want feature selection.

**Regularization** adds a penalty for large coefficients, trading some bias for less variance — usually a big win in generalization.

### Ridge Regression (L2)

$$\hat{\beta}_{\text{ridge}} = \arg\min_\beta \sum_i (y_i - x_i^\top \beta)^2 + \lambda \sum_j \beta_j^2$$

Closed form: $\hat{\beta} = (X^\top X + \lambda I)^{-1} X^\top y$. The $\lambda I$ term makes the matrix invertible even with perfect collinearity — Ridge **handles multicollinearity gracefully**.

- Shrinks all coefficients toward 0, never exactly to 0.
- Doesn't perform feature selection.
- Tunable via λ (cross-validation).

### Lasso (L1)

$$\hat{\beta}_{\text{lasso}} = \arg\min_\beta \sum_i (y_i - x_i^\top \beta)^2 + \lambda \sum_j |\beta_j|$$

Same idea but L1 penalty. No closed form — solve with coordinate descent or LARS.

- **Performs feature selection** — sends some coefficients exactly to 0.
- Useful when you expect a sparse solution (only a few features matter).
- Unstable when features are highly correlated (picks one of them, somewhat arbitrarily).

### Elastic Net

$$\lambda_1 \sum |\beta_j| + \lambda_2 \sum \beta_j^2$$

Hybrid of L1 and L2. Performs selection (L1) while still handling correlated groups (L2). Used by glmnet, scikit-learn `ElasticNet`.

### Geometric Intuition

L2 penalty is a sphere centered at origin; L1 is a diamond. The constrained-optimization geometry: the unconstrained OLS solution gets pulled toward the constraint region.

- L2's sphere has no corners → solution lands at a non-zero point on the sphere.
- L1's diamond has corners on the axes → solution often lands on an axis (coefficient = 0).

That's why L1 gives feature selection and L2 doesn't.

### Choosing λ — Cross-Validation

Don't pick λ by eye. Use k-fold CV: for each candidate λ, fit on training folds, evaluate on held-out fold, average. Pick the λ that minimizes CV error (or the largest λ within 1 SE of the minimum for sparsity — "1-SE rule").

```python
from sklearn.linear_model import RidgeCV, LassoCV, ElasticNetCV

model = RidgeCV(alphas=[0.01, 0.1, 1, 10, 100], cv=5)
model.fit(X_train, y_train)
print(model.alpha_)   # selected
```

### Don't Forget to Standardize

Regularization penalizes coefficients on the scale of the features. If `age` is in years (0–100) and `income` in dollars (0–500,000), the income coefficient is tiny and the penalty barely touches it. **Standardize features first** (`StandardScaler`) — see [z-score](./02-descriptive-stats.md). Not needed for OLS, mandatory for regularized models.

---

## ADVANCED — Generalized Linear Models (GLM)

OLS is a special case of GLMs:

$$E[y] = g^{-1}(X\beta)$$

where g is the **link function**. Examples:

| Distribution | Link | What it models |
|---|---|---|
| Normal | identity | Continuous y, OLS |
| Bernoulli | logit | Binary y, logistic |
| Binomial | logit | Counts of successes |
| Poisson | log | Count data |
| Gamma | inverse / log | Positive continuous (latency, $) |
| Negative Binomial | log | Overdispersed counts |

For count data with overdispersion (variance > mean), Poisson is wrong — use Negative Binomial. Common in DE/ML problems involving events.

```python
import statsmodels.api as sm
model = sm.GLM(y, sm.add_constant(X), family=sm.families.Poisson()).fit()
```

---

## ADVANCED — Beyond Linear: When and Why

You should know when a linear model is right and when it isn't:

| Setting | Better than linear |
|---|---|
| Non-linear interactions, tabular | Gradient boosting (XGBoost, LightGBM, CatBoost) |
| High-dim sparse text | Linear + L1, or LLM embeddings + linear head |
| Vision | CNN |
| Sequence (language, time-series) | Transformer / RNN |
| Causal estimation under confounding | Double ML, causal forests |
| Small data, interpretability needed | Stick with linear / logistic |

**Linear models remain undefeated** for: small data, when interpretability matters, when calibrated probabilities matter, as the baseline for any ML problem, and as the final layer of many deep models.

### The "Two Cultures" Bridge

Breiman's classic 2001 paper. Statistics traditionally focuses on *inference* (β estimates, p-values, CIs). ML focuses on *prediction* (CV R², test accuracy). The two cultures use overlapping math but different success criteria. Modern DS roles require both. Worth name-dropping.

---

## Common Interview Gotchas

**Q: What are the assumptions of linear regression?**
A: Linearity, independence of errors, homoskedasticity, normality of errors, no perfect multicollinearity. (LINE-M)

**Q: What does R² of 0.85 mean?**
A: 85% of variance in y is explained by the model. But it doesn't measure predictive accuracy on new data, and high R² can come from overfitting.

**Q: When is OLS the BLUE?**
A: Under the Gauss-Markov conditions: linearity, independence, homoskedasticity, mean-zero errors. (Doesn't require normality for BLUE — that's only needed for inference.)

**Q: What's the difference between L1 and L2 regularization?**
A: L1 (Lasso) performs feature selection (zeros out coefficients); L2 (Ridge) shrinks all coefficients toward zero but rarely to exactly zero. Geometric reason: L1's constraint has corners on axes.

**Q: When would Ridge outperform Lasso?**
A: When all features are weakly predictive (no sparse solution exists), or when features are highly correlated (Lasso picks one arbitrarily, Ridge averages).

**Q: How do you interpret a logistic regression coefficient?**
A: e^β is the multiplicative change in the odds of y=1 for a 1-unit increase in x.

**Q: How do you detect multicollinearity?**
A: VIF > 5 or 10. Or look at the condition number of X^TX. Or notice that small data perturbations cause big coefficient changes.

**Q: Why standardize features before Lasso/Ridge?**
A: The penalty is on the scale of the coefficients; features on bigger scales get effectively unpenalized. Standardization makes the penalty fair.

**Q: How do you handle non-linear relationships in OLS?**
A: Add transformed columns (log, polynomial, splines), use interaction terms, or switch to a non-linear model.

**Q: Coefficient is significant but the effect is tiny. Should I include it?**
A: Statistical significance ≠ practical importance. With huge n, tiny effects clear significance. Use effect size + domain judgment.

**Q: My residuals show a clear pattern. What do I do?**
A: Model is misspecified — wrong functional form, missing variables, or interactions. Add terms or switch model class.

**Q: Should I include all features, even insignificant ones?**
A: For prediction, regularize and let CV decide. For inference (explaining mechanisms), drop only with theoretical justification — stepwise selection is broken (inflates Type I error).

---

## Interview-Ready Cheat Sheet

### Key formulas

- OLS: $\hat{\beta} = (X^\top X)^{-1} X^\top y$
- R² = 1 - SS_res/SS_tot
- Adj R² = 1 - (1 - R²)(n − 1)/(n − p − 1)
- Coef t-stat: $t = \hat{\beta}/\text{SE}(\hat{\beta})$
- Logistic: $P(y=1) = 1/(1 + e^{-x^\top \beta})$
- Ridge: $\hat{\beta} = (X^\top X + \lambda I)^{-1} X^\top y$
- Lasso: no closed form; coordinate descent
- VIF_j = 1/(1 − $R_j^2$)

### The "LINE-M" assumptions

- **L**inearity
- **I**ndependence
- **N**ormality
- **E**qual variance (homoskedasticity)
- **M**ulticollinearity-free

### Trade-off pairs

- **OLS vs Ridge vs Lasso:** unbiased + high var vs shrunk + lower var vs sparse + interpretable.
- **Linear vs nonlinear models:** interpretable + low data vs flexible + needs more data.
- **High R² vs high CV-R²:** in-sample fit vs out-of-sample generalization.
- **Statistical significance vs effect size:** p-value vs practical impact.
- **L1 vs L2:** selection vs shrinkage.

### The four diagnostic plots

1. Residuals vs fitted (linearity, heteroskedasticity).
2. Q-Q plot (normality).
3. Scale-location (heteroskedasticity).
4. Cook's distance (influential points).

---

## Resources & Links

- *Elements of Statistical Learning* (Hastie, Tibshirani, Friedman) — ch. 3–4. [Free PDF.](https://hastie.su.domains/ElemStatLearn/)
- *An Introduction to Statistical Learning* (James, Witten, Hastie, Tibshirani) — the ESL "intro" version. [Free.](https://www.statlearning.com/)
- [scikit-learn linear models](https://scikit-learn.org/stable/modules/linear_model.html)
- [statsmodels regression docs](https://www.statsmodels.org/stable/regression.html) — with proper inference output.
- [Breiman — *Statistical Modeling: The Two Cultures* (2001)](https://projecteuclid.org/journals/statistical-science/volume-16/issue-3/Statistical-Modeling-The-Two-Cultures-with-comments-and-a-rejoinder/10.1214/ss/1009213726.full)
- [StatQuest — Linear Models playlist](https://www.youtube.com/playlist?list=PLblh5JKOoLUIzaEkCLIUxQFjPIlapw8nU)

### Companion files
- [Hypothesis Testing](./03-hypothesis-testing.md) — coefficient p-values.
- [ML Metrics](./06-ml-metrics.md) — evaluating regression and classification models.
- [Descriptive Stats](./02-descriptive-stats.md) — standardization for regularized models.

---

*Next: [ML Metrics & Evaluation](./06-ml-metrics.md) — how to score these models.*
