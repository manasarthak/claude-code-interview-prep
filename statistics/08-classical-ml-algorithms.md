# Classical ML Algorithms — Concept Cheat Sheet

**Phase:** 2 (Modeling)
**Difficulty:** Beginner → Intermediate
**Last updated:** July 3, 2026
**Related:** [Regression](./05-regression.md) · [ML Metrics](./06-ml-metrics.md) · [Probability Foundations](./01-probability-foundations.md)

---

## Why This File Exists

For questions like "which algorithm is *not* linear?", "when should you use random forest vs XGBoost?", "PCA does what?" — you need recognition + one-line justification, not the full math. This file is the algorithm zoo, cheat-sheet-style.

---

## The One-Line Summary Table (memorize this)

| Algorithm | Type | Kernel idea | Handles non-linear? | Handles missing? | Interpretable? | Needs scaling? |
|---|---|---|---|---|---|---|
| Linear Regression | Regression | y = Xβ + ε | No (unless features engineered) | No | Yes | Only if regularized |
| Logistic Regression | Classification | σ(Xβ) | No | No | Yes | Only if regularized |
| Ridge (L2) | Regression | OLS + λ‖β‖² | No | No | Yes | **Yes** |
| Lasso (L1) | Regression | OLS + λ‖β‖₁ (sparse) | No | No | Yes | **Yes** |
| Elastic Net | Regression | L1 + L2 | No | No | Yes | **Yes** |
| k-Nearest Neighbors (kNN) | Both | Vote/avg over k nearest | Yes (implicit) | No | Somewhat | **Yes** (distance-based) |
| Naive Bayes | Classification | Bayes + feature-independence | Weak nonlin. via kernel | Yes | Yes | No |
| Decision Tree | Both | Recursive greedy split | **Yes** | Some | Yes | No |
| Random Forest | Both | Bagging of trees | Yes | Some | Somewhat | No |
| Gradient Boosting (XGBoost / LightGBM / CatBoost) | Both | Sequentially fit residuals | Yes | Yes (built-in) | Somewhat (SHAP) | No |
| SVM | Both | Max-margin + kernel | Yes (RBF/poly) | No | No | **Yes** |
| k-Means | Clustering | Iterative centroids | Only convex clusters | No | Somewhat | **Yes** |
| DBSCAN | Clustering | Density-based | Yes | Handles noise | No | **Yes** |
| PCA | Dim reduction | Eigenvectors of cov(X) | No (linear) | No | Somewhat | **Yes** |
| t-SNE / UMAP | Viz / dim reduction | Preserves local structure | Yes | No | No | **Yes** |
| Neural Net / MLP | Both | Stacked linear + activation | Yes | No | No | **Yes** |

**Rule of thumb for scaling:** if the algorithm uses *distances* or *dot products* (kNN, SVM, k-means, PCA, regularized regression, NNs) → **scale**. If it uses *splits* (trees, random forest, gradient boosting) → **no need to scale**.

---

## LINEAR & LOGISTIC REGRESSION (30 sec review — deeper in [file 05](./05-regression.md))

- **Linear reg:** OLS closed form $\hat\beta = (X^\top X)^{-1}X^\top y$.
- **Logistic reg:** models $P(y=1) = \sigma(x^\top\beta)$. Coefficient $\beta_j$ = change in **log-odds** per unit x_j; $e^{\beta_j}$ = odds ratio.
- **Loss functions:** MSE for linear, cross-entropy (log-loss) for logistic.
- **Assumptions (LINE-M):** Linearity, Independence, Normality of errors, Equal variance (homoskedasticity), no perfect Multicollinearity.
- **Regularization:** L1 (Lasso) → sparsity. L2 (Ridge) → shrinkage, handles multicollinearity. Elastic Net = both.
- **Common trap:** logistic regression is a *linear* model — the decision boundary is linear in x; the sigmoid is on top.

---

## k-NEAREST NEIGHBORS

- **Idea:** predict from the k closest training points (majority vote for classification, average for regression).
- **Distance:** Euclidean default; can use Manhattan, Minkowski, cosine.
- **Training time:** O(1) (just store data). **Prediction time:** O(nd) per query naïvely; with KD-tree or Ball-tree, O(log n) in low dim.
- **Curse of dimensionality:** in high-d, "distances" become uninformative — everything is far from everything. kNN degrades badly above ~10-20 dimensions.
- **Bias-variance:** small k = low bias, high variance (overfits). Large k = high bias, low variance (smooths).
- **Scale features!** Distance is dominated by the largest-magnitude feature otherwise.
- **When to use:** small data, low dim, non-linear boundary, no time to train.

---

## NAIVE BAYES

- **Idea:** apply Bayes' theorem assuming feature independence given the class:

$$P(y \mid x_1, \dots, x_p) \propto P(y) \prod_j P(x_j \mid y)$$

- **Variants:** Gaussian NB (continuous features), Multinomial NB (word counts — text classification), Bernoulli NB (binary features).
- **Strengths:** very fast, works well on text (spam, sentiment), robust to irrelevant features.
- **Weakness:** the independence assumption is almost never true — but the classification often works anyway (probabilities are miscalibrated but the *argmax* is often right).
- **Laplace smoothing:** add α (usually 1) to counts to avoid zero probabilities for unseen features.

---

## DECISION TREES

- **Idea:** recursively split the data on the feature/threshold that maximally reduces impurity.
- **Impurity measures:**
  - **Gini** = 1 − Σ p²ᵢ (default in sklearn).
  - **Entropy** = −Σ pᵢ log pᵢ (used in ID3, C4.5).
  - **MSE / variance** for regression.
- **Split rule:** pick (feature, threshold) that maximizes impurity reduction (a.k.a. **information gain** for entropy).
- **Overfitting:** trees are notorious for it. Control via `max_depth`, `min_samples_leaf`, `ccp_alpha` (cost-complexity pruning).
- **Pros:** no scaling needed, handles mixed data types, interpretable, can capture interactions natively.
- **Cons:** high variance, unstable (small data change → very different tree), axis-aligned splits only.
- **Interview note:** "greedy" is the operative word — trees never backtrack on splits.

---

## RANDOM FOREST (Bagging)

- **Idea:** build many decision trees on **bootstrap** samples with **random feature subsets** at each split; predictions averaged (regression) or voted (classification).
- **Why it works:** decorrelates the base trees → averaging kills variance without adding bias.
- **Hyperparams that matter:** `n_estimators` (more = better, marginal returns), `max_features` (√p for classification, p/3 for regression is default), `max_depth`, `min_samples_leaf`.
- **OOB error:** each tree only sees ~63% of samples; the ~37% "out-of-bag" gives a free CV-like estimate.
- **Feature importance:** mean decrease in impurity (biased toward high-cardinality features) or permutation importance (more reliable).
- **Interview note:** "Random forest handles multicollinearity by randomly subsetting features" ✓.

---

## GRADIENT BOOSTING (XGBoost / LightGBM / CatBoost)

- **Idea:** fit trees **sequentially**, each new tree predicting the residuals (gradients) of the previous ensemble.
- **Contrast with RF:** RF is *bagging* (parallel, high-variance trees averaged). GB is *boosting* (sequential, weak trees combined to reduce bias).
- **Loss + gradient:** minimizes any differentiable loss (log-loss, MSE, custom). Each tree fits the negative gradient.
- **Key hyperparams:** `n_estimators`, `learning_rate` (a.k.a. shrinkage, ~0.01–0.1), `max_depth` (usually shallow, 3–8), `subsample`, `colsample_bytree`.
- **Regularization:** early stopping (stop adding trees when validation loss plateaus), L1/L2 on leaf weights (XGBoost `reg_alpha` / `reg_lambda`).
- **Library differences:**
  - **XGBoost:** first mainstream, level-wise growth, most features.
  - **LightGBM:** leaf-wise growth (faster, but riskier overfit), best on large data.
  - **CatBoost:** handles categoricals natively (ordered target encoding), robust to hyperparams.
- **When it wins:** structured/tabular data. **Still the SOTA for tabular ML in 2026** — deep learning hasn't beaten it broadly.

---

## SVM (Support Vector Machines)

- **Idea:** find the hyperplane that **maximizes the margin** between classes.
- **Soft margin (C parameter):** larger C = less tolerance for misclassification = more overfit-prone.
- **Kernel trick:** implicitly map to higher-dim feature space via a kernel function K(x, x′) without explicit computation.
  - **Linear:** K(x, x′) = x · x′.
  - **Polynomial:** K = (x · x′ + c)^d.
  - **RBF (Gaussian):** K = exp(−γ‖x − x′‖²). Most common non-linear kernel.
- **Support vectors:** only the points on / inside the margin matter.
- **Weakness:** O(n²) - O(n³) training; doesn't scale beyond ~100K samples. **Scale features.**
- **When to use:** medium-sized, non-linear, structured data. Falls out of fashion vs boosting/NNs but still shows up in interview questions.

---

## K-MEANS (Unsupervised)

- **Idea:** partition n points into k clusters, each point assigned to nearest centroid, centroids updated to means.
- **Objective:** minimize within-cluster sum of squares (WCSS = inertia).
- **Convergence:** guaranteed to a *local* minimum, not global. Run multiple times (`n_init > 1`) and take best.
- **Init:** k-means++ picks initial centroids far apart (default in sklearn); dramatically better than random.
- **Choosing k:** elbow method (plot WCSS vs k, pick the "kink"), silhouette score, gap statistic.
- **Assumes:** roughly spherical, similar-size clusters. Fails on non-convex shapes.
- **Alternative for non-convex:** DBSCAN, hierarchical, Gaussian Mixture Models (soft clustering).
- **Scale features!** Euclidean distance dominated by large-scale features.

---

## PCA (Principal Component Analysis)

- **Idea:** find orthogonal directions of maximum variance; project data onto the top-k components.
- **Math:** compute covariance matrix Σ = X^T X / (n − 1) after centering; eigendecomposition of Σ; sort eigenvectors by eigenvalue; keep top k. Equivalent to SVD of centered X.
- **Explained variance:** eigenvalue λᵢ / Σλⱼ = fraction of variance in the i-th component.
- **Choose k:** by cumulative explained variance (e.g., "keep components summing to 95%"), by scree plot, or by CV downstream.
- **Assumptions:** linear (only linear projections). For non-linear structure use t-SNE, UMAP, kernel PCA, or autoencoders.
- **Scale features!** Otherwise the largest-magnitude feature dominates every component.
- **Interview note:** PCA is **unsupervised** — doesn't use y at all. For supervised dim reduction, use LDA.
- **Related:** LDA (Linear Discriminant Analysis) maximizes between-class separation; supervised alternative to PCA.

---

## FEATURE ENGINEERING (interview-favorite topic)

### Encoding categoricals

- **One-hot encoding:** k categories → k binary columns. Explodes dim for high-cardinality. Drop one to avoid multicollinearity in regression (`drop_first=True`).
- **Label encoding:** map categories to integers (0, 1, 2, ...). **Never for linear models / distance-based models** — it imposes false ordering. Fine for tree-based.
- **Target encoding:** replace category with mean of y within category. Leaks the target — must fit on train only or use CV encoding.
- **CatBoost's ordered target encoding:** target encoding without leakage. Built-in.
- **Frequency encoding:** replace category with its count in the data.
- **Embedding layers:** for high-cardinality categoricals in neural nets; learned low-dim vector per category.

### Scaling

- **StandardScaler (z-score):** (x − μ)/σ. Sensitive to outliers.
- **MinMaxScaler:** (x − min)/(max − min) → [0, 1]. Preserves shape, sensitive to outliers.
- **RobustScaler:** uses median and IQR. Robust to outliers.
- **PowerTransformer / QuantileTransformer:** make data more Gaussian-like.

**Common trap:** always fit scaler on **train only**, transform on both. Fitting on train+test = leakage.

### Handling missing values

- **Drop rows / columns:** loses info; only OK if MCAR.
- **Imputation:** mean/median/mode (simple), KNN imputer (nearest neighbors), IterativeImputer / MICE (chained regression).
- **Missingness as feature:** add binary `x_is_missing` column — often useful signal on its own.
- **Tree-based:** XGBoost/LightGBM/CatBoost handle NaN natively via learned split direction.

### Feature interactions / derived features

- **Polynomial features:** x, x², x·y, etc.
- **Log transform** for right-skewed positive features (income, latency, counts).
- **Binning** for non-linear relationships.
- **Datetime features:** hour-of-day, day-of-week, is_weekend, seconds-since-epoch.
- **Text:** TF-IDF, word/token counts, character n-grams, embeddings.

---

## HYPERPARAMETER TUNING

- **Grid search:** exhaustive over discrete grid. Slow, deterministic.
- **Random search:** random sample of hyperparam combinations. Usually beats grid at same budget for high-dim spaces (Bergstra & Bengio 2012).
- **Bayesian optimization:** builds a probabilistic model of val-loss vs hyperparams. Tools: Optuna, Hyperopt, scikit-optimize.
- **Halving (Successive Halving / HyperBand):** race many configs on small data, promote survivors to bigger data. `HalvingGridSearchCV` in sklearn.

**Always tune with CV**, and use **nested CV** if you're both tuning and reporting a metric — otherwise the metric is optimistically biased.

---

## ENSEMBLES

- **Bagging** (bootstrap aggregating): parallel base models on bootstrap samples, average / vote. Reduces **variance**. Example: Random Forest.
- **Boosting**: sequential base models, each correcting the previous. Reduces **bias**. Example: XGBoost / LightGBM / AdaBoost.
- **Stacking**: train several diverse base models, use a meta-learner on their predictions. Often blends bagged + boosted + linear + NN.
- **Voting**: hard (majority class) or soft (average predicted probability) voting across models.

**Rule of thumb:** if the question says "reduces variance" → bagging. "Reduces bias" → boosting.

---

## BIAS-VARIANCE DECOMPOSITION

$$\mathbb{E}[(y - \hat f)^2] = \underbrace{(\text{Bias})^2}_{\text{underfit}} + \underbrace{\text{Variance}}_{\text{overfit}} + \underbrace{\sigma^2}_{\text{irreducible}}$$

| Change | Bias | Variance |
|---|---|---|
| Bigger model / more depth / more features | ↓ | ↑ |
| More regularization | ↑ | ↓ |
| More training data | ≈ same | ↓ |
| Bagging (RF) | ≈ same | ↓ |
| Boosting | ↓ | ↑ (mildly) |

---

## COMMON TRAPS (read these once)

1. **"Logistic regression is a classification algorithm."** True, but it's a *linear* model with a sigmoid on the output.
2. **"kNN is a parametric model."** False — non-parametric (no fixed number of parameters, model complexity grows with n).
3. **"Random forest requires feature scaling."** False — tree-based models never need scaling.
4. **"L1 always outperforms L2."** False — L1 for sparse solutions, L2 when all features weakly matter or are highly correlated.
5. **"PCA uses labels."** False — PCA is *unsupervised*. LDA uses labels.
6. **"Boosting is a bagging method."** False. Bagging = parallel; boosting = sequential.
7. **"Naive Bayes assumes features are independent."** True — conditional independence *given the class*.
8. **"k-means always finds the global optimum."** False — local minimum only; use `n_init > 1`.
9. **"Gradient boosting handles missing values automatically."** True for XGBoost / LightGBM / CatBoost (learned default direction at each split).
10. **"Fewer neighbors in kNN → lower bias, higher variance."** True.
11. **"Deep learning always beats gradient boosting on tabular data."** False — XGBoost / LightGBM still lead on most tabular benchmarks in 2026.
12. **"Cross-entropy loss is the same as log loss."** True.
13. **"SVM's C parameter: larger C = more regularization."** **False** — larger C = *less* regularization (more penalty for margin violations = tighter fit).
14. **"Elbow method for k in k-means uses silhouette score."** False — elbow uses inertia (WCSS). Silhouette is a separate method.
15. **"PCA components are orthogonal."** True — that's the definition.

---

## The 20-Question Self-Test (do this in 10 minutes)

1. Which loss does logistic regression minimize?
2. When would you use Lasso over Ridge?
3. What's the difference between bagging and boosting?
4. Which algorithms need feature scaling?
5. Explain the kernel trick in SVM.
6. What does PCA maximize?
7. Why does k-means converge to a local optimum?
8. When would you choose kNN over logistic regression?
9. What's the difference between Gini and entropy?
10. What's OOB error in Random Forest?
11. Why does XGBoost handle missing values while sklearn's DecisionTreeClassifier doesn't?
12. When is Naive Bayes' independence assumption most problematic?
13. What is the elbow method?
14. How would you tune the learning rate in gradient boosting?
15. What's the difference between one-hot and target encoding, and which leaks the target?
16. What's a support vector?
17. How does DBSCAN differ from k-means?
18. Why is bagging good at reducing variance?
19. What's the difference between L1 and L2 geometrically?
20. When would you pick UMAP over PCA?

Answers throughout this file. If you can't answer one, that's the section to skim now.

---

## Resources

- *Elements of Statistical Learning* (Hastie, Tibshirani, Friedman) — [free PDF](https://hastie.su.domains/ElemStatLearn/).
- [scikit-learn User Guide](https://scikit-learn.org/stable/user_guide.html) — the canonical reference for every algorithm above.
- [XGBoost docs](https://xgboost.readthedocs.io/) · [LightGBM docs](https://lightgbm.readthedocs.io/) · [CatBoost docs](https://catboost.ai/docs/).
- [StatQuest YouTube](https://www.youtube.com/c/joshstarmer) — best intuition videos.
- [Chip Huyen — Machine Learning Interviews Book](https://huyenchip.com/ml-interviews-book/) — free online.

### Companion files
- [Regression](./05-regression.md) — deep dive on linear / logistic / regularized.
- [ML Metrics](./06-ml-metrics.md) — how to evaluate these.
- [Probability Foundations](./01-probability-foundations.md) — Bayes for Naive Bayes.
- [Descriptive Stats](./02-descriptive-stats.md) — z-score / standardization.

---

*Skim this once before an interview. Algorithm-name-recognition + one-line-justification wins the quick-fire conceptual questions.*
