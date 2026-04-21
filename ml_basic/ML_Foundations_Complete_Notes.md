# 📄 Machine Learning Foundations — Complete Study Notes

**Tags:** #academic #machine-learning #scikit-learn #classification #regression #clustering #supervised #unsupervised
**Links:** [[Deep Learning]], [[Statistics & Probability]], [[Feature Engineering]], [[Model Evaluation]]

---

# PART 1 — The ML Taxonomy

## 🎯 The "Elevator Pitch"
> Machine learning is teaching a computer to make decisions from data rather than from explicit rules. You feed it examples, it finds patterns, and then it generalizes to new, unseen inputs.

## 🧠 The Three Paradigms

| Paradigm | What you give the model | What it learns | Example |
|---|---|---|---|
| **Supervised** | Labeled data `(X, y)` | A mapping `f(X) → y` | Customer churn prediction |
| **Unsupervised** | Unlabeled data `(X)` | Hidden structure in `X` | Customer segmentation |
| **Reinforcement** | Reward signals | A policy to maximize reward | Game-playing agents |

### Supervised → Two Sub-tasks

| Task | Target `y` | Example |
|---|---|---|
| **Classification** | Discrete categories | Churn risk: High / Med / Low |
| **Regression** | Continuous values | Predict house price: $430,000 |

---

# PART 2 — The Universal ML Pipeline

## 🎯 The "Elevator Pitch"
> Every ML project follows the same pipeline, regardless of algorithm. Think of it as a production line: raw data enters one end, a trained model exits the other.

## 🧠 Pipeline Stages

```
Raw Data
   │
   ▼
[1] Data Exploration       ← Understand distributions, missing values, class imbalance
   │
   ▼
[2] Data Preprocessing     ← Impute, encode, scale, drop irrelevant features
   │
   ▼
[3] Train / Test Split     ← Preserve evaluation integrity
   │
   ▼
[4] Model Selection        ← Pick algorithm(s) appropriate for the task
   │
   ▼
[5] Training               ← model.fit(X_train, y_train)
   │
   ▼
[6] Prediction             ← model.predict(X_test)
   │
   ▼
[7] Evaluation             ← Compare predictions vs. ground truth
   │
   ▼
[8] Tune Hyperparameters   ← Loop back to step 2 until satisfied
```

## 💻 Skeleton Code — Universal ML Template

```python
from sklearn.pipeline import Pipeline
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, classification_report

# ── Step 1: Load & Explore ────────────────────────────────
import pandas as pd
df = pd.read_csv("data.csv")
print(df.info())             # dtypes, non-null counts
print(df.describe())         # stats
print(df["target"].value_counts())  # class distribution

# ── Step 2: Preprocessing ─────────────────────────────────
num_cols = ["age", "balance", "transactions"]
cat_cols = ["gender", "region"]
drop_cols = ["id", "timestamp"]          # no predictive value

df.drop(columns=drop_cols, inplace=True)

num_pipeline = Pipeline([
    ("imputer", SimpleImputer(strategy="most_frequent")),
    ("scaler", StandardScaler()),          # (x - mean) / std
])

cat_pipeline = Pipeline([
    ("imputer", SimpleImputer(strategy="most_frequent")),
    ("encoder", OneHotEncoder(handle_unknown="ignore")),  # binary columns per category
])

preprocessor = ColumnTransformer([
    ("num", num_pipeline, num_cols),
    ("cat", cat_pipeline, cat_cols),
])

# ── Step 3: Label Encoding (classification target) ────────
from sklearn.preprocessing import LabelEncoder
le = LabelEncoder()
y = le.fit_transform(df["CHURNRISK"])    # "High"→0, "Low"→1, "Medium"→2

X = df.drop(columns=["CHURNRISK"])

# ── Step 4: Split ─────────────────────────────────────────
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.02, random_state=42   # 98% train / 2% test
)

# ── Step 5–6: Build Full Pipeline + Train + Predict ───────
model_pipeline = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", RandomForestClassifier(n_estimators=100, random_state=42)),
])

model_pipeline.fit(X_train, y_train)
y_pred = model_pipeline.predict(X_test)

# ── Step 7: Evaluate ──────────────────────────────────────
print(accuracy_score(y_test, y_pred))
print(classification_report(y_test, y_pred))
```

### Why use `Pipeline` instead of manual steps?
- **Prevents data leakage**: The scaler is fit only on training data, then applied to test data. Without Pipeline, beginners often accidentally scale the whole dataset before splitting.
- **Reproducibility**: The entire transformation + model is one serializable object (`pickle`/`joblib`).
- **Production-ready**: You deploy the pipeline, not just the model — same transforms apply to live data automatically.

---

# PART 3 — Preprocessing Deep Dive

## 🎯 The "Elevator Pitch"
> Preprocessing is 80% of the real work. Garbage in, garbage out — even a perfect algorithm will fail on dirty data.

## 🧠 Core Techniques

### 3.1 Handling Missing Values — `SimpleImputer`

```python
from sklearn.impute import SimpleImputer

imputer = SimpleImputer(strategy="most_frequent")
# Other strategies: "mean", "median", "constant"
X_filled = imputer.fit_transform(X)
```

**Why `most_frequent` for categorical data?**
Using `mean` on a string column makes no sense. `most_frequent` is a safe, non-distorting default for categories.

### 3.2 Encoding Categorical Variables — `OneHotEncoder`

```python
from sklearn.preprocessing import OneHotEncoder
import numpy as np

# Before encoding:  gender = ["Male", "Female", "Female", "Male"]
# After encoding:   gender_Male=[1,0,0,1], gender_Female=[0,1,1,0]

enc = OneHotEncoder(sparse_output=False, handle_unknown="ignore")
encoded = enc.fit_transform(X[["gender", "region"]])
```

**Why One-Hot instead of Label Encoding for categories?**
Label encoding assigns integers (Male=0, Female=1), which implies an *ordinal* relationship — the model thinks Female > Male numerically. One-Hot avoids this by creating independent binary columns, removing any false ordering.

**Trade-off:** One-Hot explodes dimensionality on high-cardinality columns (e.g., 1000 zip codes → 1000 columns). Use target encoding or embeddings in that case.

### 3.3 Feature Scaling — `StandardScaler`

```python
from sklearn.preprocessing import StandardScaler

# Formula: z = (x - μ) / σ
# Result: each column has mean=0, std=1

scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_numeric)
```

**Why scale?**
- Distance-based algorithms (KNN, SVM, K-Means) measure distances between points. A column with values in [0, 1,000,000] will dominate a column in [0, 1] unless scaled.
- Tree-based models (Random Forest, Decision Trees) are **scale-invariant** — they split on thresholds, not distances. Scaling has zero effect on them.

| Algorithm | Needs Scaling? |
|---|---|
| KNN | ✅ Yes |
| SVM | ✅ Yes |
| Logistic Regression | ✅ Yes (for convergence) |
| Linear Regression | ✅ Yes (coefficient interpretation) |
| Decision Tree | ❌ No |
| Random Forest | ❌ No |
| K-Means | ✅ Yes |

### 3.4 `ColumnTransformer` — Apply Different Transforms to Different Columns

```python
from sklearn.compose import ColumnTransformer

preprocessor = ColumnTransformer(transformers=[
    ("num", StandardScaler(), ["age", "balance"]),
    ("cat", OneHotEncoder(), ["gender", "region"]),
    # Columns not listed here are DROPPED by default
], remainder="passthrough")   # "passthrough" keeps unlisted cols
```

---

# PART 4 — Classification Algorithms

## 🎯 The "Elevator Pitch"
> Classification = "Which bucket does this data point belong to?" Every algorithm answers this question using a different geometric or probabilistic strategy.

## 4.1 Naive Bayes

**Core Idea:** Use probability theory (Bayes' theorem) to assign class membership. Assumes all features are *independent* of each other — a "naive" but often effective simplification.

**Formula:**
```
P(Class | Features) ∝ P(Features | Class) × P(Class)
```

```python
from sklearn.naive_bayes import GaussianNB

model = GaussianNB()
model.fit(X_train, y_train)
y_pred = model.predict(X_test)
# Accuracy on customer churn dataset: ~69%
```

**When to use:**
- Text classification (spam filters) — words are roughly independent
- Very fast training, small datasets
- Baseline model before trying complex approaches

**When it fails:**
- When features are highly correlated (violates the independence assumption)
- Continuous numeric features with complex distributions

---

## 4.2 Logistic Regression

**Core Idea:** Fit a linear equation to the data, then squash its output through a **sigmoid function** to produce a probability between 0 and 1.

```
sigmoid(z) = 1 / (1 + e^(-z))
where z = w₀ + w₁x₁ + w₂x₂ + ... + wₙxₙ
```

```python
from sklearn.linear_model import LogisticRegression

model = LogisticRegression(max_iter=1000, C=1.0)
# C = inverse regularization strength: smaller C = more regularization
model.fit(X_train, y_train)
y_pred = model.predict(X_test)
# Accuracy on customer churn dataset: ~92%

# Get probabilities instead of hard labels:
y_proba = model.predict_proba(X_test)   # shape: (n_samples, n_classes)
```

**When to use:**
- Binary or multi-class problems with linearly separable data
- When you need **probability outputs** (e.g., "70% chance of churn")
- When interpretability matters — coefficients show feature importance directly

**When it fails:**
- Non-linear decision boundaries — will underfit complex data
- Highly correlated features need regularization (Ridge/Lasso via `penalty` param)

---

## 4.3 K-Nearest Neighbors (KNN)

**Core Idea:** No training phase — just memorize data. At prediction time, find the K most similar training examples (by Euclidean distance) and vote for the majority class.

```python
from sklearn.neighbors import KNeighborsClassifier
import numpy as np

# Euclidean distance: d = sqrt(Σ(xᵢ - yᵢ)²)

model = KNeighborsClassifier(n_neighbors=5, metric="euclidean")
model.fit(X_train, y_train)   # No actual training, just stores data
y_pred = model.predict(X_test)
# Accuracy on customer churn dataset: ~89%

# Choosing K: try multiple values
from sklearn.model_selection import cross_val_score
k_scores = [cross_val_score(KNeighborsClassifier(k), X, y, cv=5).mean()
            for k in range(1, 21)]
best_k = np.argmax(k_scores) + 1
```

**When to use:**
- Small datasets (slow at inference on large datasets)
- Non-linear decision boundaries
- When decision boundary logic is hard to specify manually

**When it fails:**
- High-dimensional data (curse of dimensionality — all points become equidistant)
- Large datasets — O(n) prediction cost per query
- Sensitive to irrelevant/noisy features and requires scaling

**Curse of Dimensionality:** As dimensions increase, the volume of space grows exponentially. Points that seem "near" in 2D become increasingly equidistant in 100D, making "nearest neighbors" meaningless.

---

## 4.4 Support Vector Machines (SVM)

**Core Idea:** Find the **hyperplane** that maximally separates classes, with the largest possible margin between the nearest points of each class (called *support vectors*). For non-linearly-separable data, use a **kernel trick** to map data to higher dimensions where it *is* separable.

```python
from sklearn.svm import SVC

# Linear kernel: fast, works for linearly separable data
model_linear = SVC(kernel="linear", C=1.0)

# RBF (Radial Basis Function) kernel: maps to infinite dimensions
# γ controls how far the influence of each training example reaches
model_rbf = SVC(kernel="rbf", C=1.0, gamma="scale")
model_rbf.fit(X_train, y_train)
y_pred = model_rbf.predict(X_test)
# Accuracy on customer churn dataset: ~95%
```

**The Kernel Trick (Why it's brilliant):**
Instead of explicitly computing coordinates in high-dimensional space (expensive), the kernel function computes the *inner product* between two points in that high-dimensional space directly. You get the benefit of high-dimensional separation without the computational cost.

**Common Kernels:**

| Kernel | Use Case |
|---|---|
| `linear` | Linearly separable, high-dim text data |
| `rbf` (Gaussian) | General purpose, non-linear |
| `poly` | Polynomial relationships |

**Hyperparameters:**
- `C` — Regularization. High C = low tolerance for misclassification (risk of overfitting). Low C = more misclassifications allowed (risk of underfitting).
- `gamma` — Influence radius of each support vector. High gamma = narrow, tight clusters → overfitting.

**When to use:**
- High-dimensional data (text classification, genomics)
- Clear margin of separation exists
- Binary or multi-class (via one-vs-rest)

**When it fails:**
- Very large datasets (quadratic training complexity)
- Noisy data with overlapping classes
- Requires careful scaling

---

## 4.5 Decision Trees

**Core Idea:** Build a tree of if-then rules by recursively splitting data on the feature that best separates classes. Each internal node is a feature test, each leaf is a class.

```python
from sklearn.tree import DecisionTreeClassifier, export_text

model = DecisionTreeClassifier(
    max_depth=5,           # Limit depth to prevent overfitting
    min_samples_split=10,  # Min samples to split a node
    criterion="gini"       # "gini" or "entropy" (information gain)
)
model.fit(X_train, y_train)

# Inspect the rules:
print(export_text(model, feature_names=list(X.columns)))
```

**Splitting Criteria:**
- **Gini Impurity:** `G = 1 - Σ(pᵢ²)` — how often a randomly chosen element would be incorrectly classified
- **Entropy / Information Gain:** `H = -Σ(pᵢ log₂ pᵢ)` — measures disorder; prefer splits that reduce entropy most

```python
# Conceptual split logic:
def best_split(data):
    best_gain = 0
    for feature in features:
        for threshold in unique_values(feature):
            left, right = split(data, feature, threshold)
            gain = information_gain(data, left, right)
            if gain > best_gain:
                best_feature, best_threshold = feature, threshold
    return best_feature, best_threshold
```

**When to use:**
- Need interpretability (can visualize the tree)
- Handles mixed data types without scaling
- Feature selection happens automatically

**When it fails:**
- Prone to **overfitting** without depth limits — trees memorize noise
- Unstable — small data changes produce very different trees
- Biased toward features with more unique values

---

## 4.6 Ensemble Learning — Random Forest

**Core Idea:** Train hundreds of decision trees on different random subsets of data and features. Combine their predictions by majority vote. Individual weak trees, collectively strong.

```python
from sklearn.ensemble import RandomForestClassifier

model = RandomForestClassifier(
    n_estimators=100,      # Number of trees
    max_features="sqrt",   # Features considered per split: sqrt(n_features)
    max_depth=None,        # Let trees grow fully (randomness prevents overfit)
    bootstrap=True,        # Train each tree on a random sample with replacement
    random_state=42
)
model.fit(X_train, y_train)
y_pred = model.predict(X_test)
# Accuracy on customer churn dataset: ~91%

# Bonus: Feature importance
import pandas as pd
importances = pd.Series(model.feature_importances_, index=X.columns)
importances.sort_values(ascending=False).plot(kind="bar")
```

**Why Random Forests Work (Bias-Variance Trade-off):**
- A single deep tree: **low bias, high variance** — overfits
- Many trees averaged together: **variance cancels out** across trees, bias stays low
- Randomness in features (`max_features="sqrt"`) ensures trees are *decorrelated* — if they all made the same mistake, averaging wouldn't help

```
Prediction = majority_vote([tree_1(x), tree_2(x), ..., tree_n(x)])
```

**Random Forest vs Decision Tree:**

| Property | Decision Tree | Random Forest |
|---|---|---|
| Overfitting | High risk | Low risk |
| Interpretability | ✅ High | ❌ Black box |
| Training speed | Fast | Slower |
| Accuracy | Lower | Higher |
| Feature importance | Yes | Yes (averaged) |

---

## 4.7 Ensemble Learning — Gradient Boosted Trees

**Core Idea:** Unlike Random Forest (parallel trees), Gradient Boosting builds trees *sequentially*. Each new tree learns to correct the *residual errors* of the previous ensemble.

```python
from sklearn.ensemble import GradientBoostingClassifier

model = GradientBoostingClassifier(
    n_estimators=100,      # Number of boosting stages
    learning_rate=0.1,     # Shrinks each tree's contribution (regularization)
    max_depth=3,           # Shallow trees preferred (weak learners)
    subsample=0.8,         # Fraction of samples per tree (stochastic GBT)
)
model.fit(X_train, y_train)

# XGBoost (faster implementation, often preferred in practice):
# pip install xgboost
from xgboost import XGBClassifier
xgb = XGBClassifier(n_estimators=100, learning_rate=0.1, use_label_encoder=False)
```

**Conceptual Training Loop:**
```python
# Pseudocode — Gradient Boosting mechanics
predictions = base_prediction(y_train)   # e.g., mean of y

for i in range(n_estimators):
    residuals = y_train - predictions                  # What did we get wrong?
    tree_i = DecisionTree(max_depth=3)
    tree_i.fit(X_train, residuals)                     # Learn to predict the errors
    predictions += learning_rate * tree_i.predict(X_train)  # Correct the ensemble
```

**Random Forest vs Gradient Boosting:**

| Property | Random Forest | Gradient Boosting |
|---|---|---|
| Tree construction | Parallel | Sequential |
| Speed | Faster | Slower (but XGBoost fixes this) |
| Overfitting risk | Lower | Higher (needs tuning) |
| Accuracy (typical) | Good | Often better |
| Key hyperparams | `n_estimators`, `max_features` | `learning_rate`, `n_estimators`, `max_depth` |

---

## Classification Algorithm Comparison

| Algorithm | Accuracy (Churn Dataset) | Interpretable | Needs Scaling | Notes |
|---|---|---|---|---|
| Naive Bayes | ~69% | ✅ | ❌ | Fast baseline |
| Logistic Regression | ~92% | ✅ | ✅ | Great first model |
| KNN | ~89% | ✅ | ✅ | No training phase |
| SVM (RBF) | ~95% | ❌ | ✅ | Highest accuracy here |
| Decision Tree | Varies | ✅ | ❌ | Prone to overfit |
| Random Forest | ~91% | ❌ | ❌ | Robust, go-to model |
| Gradient Boosting | Often highest | ❌ | ❌ | Tune carefully |

> ⚠️ **Critical insight:** No single algorithm is universally best. Performance depends on dataset characteristics, class balance, feature types, and dimensionality. Always compare multiple models empirically.

---

# PART 5 — Regression Algorithms

## 🎯 The "Elevator Pitch"
> Regression = "What's the exact number?" instead of "Which bucket?" You're predicting continuous values like price, temperature, or revenue.

## 5.1 Simple Linear Regression

**Formula:** `y = w₀ + w₁x₁`

```python
from sklearn.linear_model import LinearRegression

X_single = df[["living_area"]]   # One feature only
y = df["sale_price"]

X_train, X_test, y_train, y_test = train_test_split(X_single, y, test_size=0.2)

model = LinearRegression()
model.fit(X_train, y_train)

print(f"Intercept (w₀): {model.intercept_:.2f}")
print(f"Slope (w₁): {model.coef_[0]:.2f}")
# Interpretation: "Each extra sq ft adds $w₁ to the price"
```

**Limitation:** Real-world target variables depend on many factors. One feature is almost never enough.

---

## 5.2 Multiple Linear Regression

**Formula:** `y = w₀ + w₁x₁ + w₂x₂ + ... + wₙxₙ`

```python
X_multi = df[["living_area", "bedrooms", "age", "garage_size"]]

model = LinearRegression()
model.fit(X_train, y_train)

# Each coefficient shows the marginal effect of that feature
# holding all others constant
```

---

## 5.3 Polynomial Regression

**Core Idea:** When the relationship is curved, engineer polynomial features (x², x³) and feed them to a linear model.

```python
from sklearn.preprocessing import PolynomialFeatures

# Creates: [1, x, x², x³] for degree=3
poly = PolynomialFeatures(degree=3, include_bias=False)
X_poly = poly.fit_transform(X_single)

model = LinearRegression()
model.fit(X_poly, y_train)
```

**Trade-off: Degree vs Overfitting**

| Degree | Behavior |
|---|---|
| 1 | Same as linear (underfitting) |
| 2-3 | Good fit for many real curves |
| 10+ | Memorizes training data, fails on new data (overfitting) |

---

## 5.4 Decision Tree Regression

```python
from sklearn.tree import DecisionTreeRegressor

model = DecisionTreeRegressor(max_depth=5)
model.fit(X_train, y_train)
# Prediction = average of all training samples in the same leaf
```

**Key difference from classification trees:** Leaf output is the **mean** of training samples in that region, not a majority class.

---

## 5.5 Random Forest Regression

```python
from sklearn.ensemble import RandomForestRegressor

model = RandomForestRegressor(n_estimators=100, max_depth=10, random_state=42)
model.fit(X_train, y_train)
# Prediction = average of all individual tree predictions
```

---

## 5.6 Regression Evaluation Metrics

```python
from sklearn.metrics import mean_squared_error, r2_score
import numpy as np

y_pred = model.predict(X_test)

mse = mean_squared_error(y_test, y_pred)
rmse = np.sqrt(mse)       # Same unit as y (dollars, not dollars²)
r2 = r2_score(y_test, y_pred)

print(f"RMSE: {rmse:.2f}")
print(f"R²:   {r2:.4f}")
```

**Understanding R²:**

```
R² = 1 - (SS_residual / SS_total)
   = 1 - (Σ(yᵢ - ŷᵢ)² / Σ(yᵢ - ȳ)²)
```

| R² Value | Interpretation |
|---|---|
| 1.0 | Perfect predictions |
| 0.0 | No better than predicting the mean |
| < 0 | Worse than predicting the mean (model is broken) |

**MSE vs RMSE:**
- MSE penalizes large errors more (squaring amplifies outliers) — use when big errors are particularly costly
- RMSE is in the same units as `y`, easier to interpret

---

## Regression Algorithm Comparison

| Algorithm | Handles Non-linearity | Interpretable | Outlier Sensitive | Notes |
|---|---|---|---|---|
| Simple Linear | ❌ | ✅ | ✅ | Start here always |
| Multiple Linear | Partial | ✅ | ✅ | Standard baseline |
| Polynomial | ✅ | ✅ | ✅ | Overfits at high degree |
| Decision Tree | ✅ | ✅ | ❌ | Unstable, weak alone |
| Random Forest | ✅ | ❌ | ❌ | Robust, strong default |
| Gradient Boosting | ✅ | ❌ | ❌ | Often best accuracy |

---

# PART 6 — Clustering (Unsupervised Learning)

## 🎯 The "Elevator Pitch"
> Clustering is exploring data without a map. You don't tell the algorithm what groups to find — it discovers them by measuring similarity. Used to uncover hidden structure.

## 6.1 K-Means Clustering

**Core Idea:** Partition data into K groups by minimizing the distance between each point and its assigned cluster **centroid** (mean).

```python
from sklearn.cluster import KMeans
import matplotlib.pyplot as plt

# Step 1: Choose K (the big weakness)
# Elbow Method: plot inertia vs K
inertias = []
for k in range(1, 11):
    km = KMeans(n_clusters=k, n_init=10, random_state=42)
    km.fit(X)
    inertias.append(km.inertia_)   # Sum of squared distances to centroids

plt.plot(range(1, 11), inertias, "bo-")
plt.xlabel("K")
plt.ylabel("Inertia")
plt.title("Elbow Method")
plt.show()

# Step 2: Train with chosen K
model = KMeans(n_clusters=3, n_init=10, random_state=42)
model.fit(X)
labels = model.labels_               # Cluster ID for each point
centers = model.cluster_centers_     # Centroid coordinates
```

**Algorithm Steps:**
```python
# K-Means pseudocode
def kmeans(X, k, max_iter=300):
    centroids = smart_initialize(X, k)      # k-means++ default
    for _ in range(max_iter):
        # Assignment step
        labels = [argmin(distance(x, c) for c in centroids) for x in X]
        # Update step
        new_centroids = [mean(X[labels == i]) for i in range(k)]
        if centroids == new_centroids:      # Convergence
            break
        centroids = new_centroids
    return labels, centroids
```

**Limitations:**
- Must specify K upfront
- Assumes **spherical** clusters — fails on crescent/ring shapes
- Sensitive to outliers (centroid pulled toward them)
- Sensitive to initialization (k-means++ mitigates this)

---

## 6.2 Mean Shift

**Core Idea:** No need to specify K. Seeks areas of **high density** and places cluster centers there. Points in sparse regions become outliers (orphans).

```python
from sklearn.cluster import MeanShift, estimate_bandwidth

bandwidth = estimate_bandwidth(X, quantile=0.2)  # Controls neighborhood size
model = MeanShift(bandwidth=bandwidth, cluster_all=False)
# cluster_all=False: sparse points get label -1 (orphans/anomalies)
model.fit(X)
labels = model.labels_    # -1 = outlier
```

**K-Means vs Mean Shift:**

| Property | K-Means | Mean Shift |
|---|---|---|
| Specify K? | ✅ Required | ❌ Automatic |
| Cluster shape | Spherical only | Follows density |
| Outliers | Forced into clusters | Labeled as -1 |
| Speed | Faster | Slower |

---

## 6.3 DBSCAN — Density-Based Spatial Clustering

**Core Idea:** Clusters are dense regions of points separated by sparse regions. Non-dense points are labeled **noise/outliers**. Handles arbitrary shapes.

```python
from sklearn.cluster import DBSCAN

model = DBSCAN(
    eps=0.5,           # Maximum radius of a neighborhood
    min_samples=5      # Minimum points to form a dense region (core point)
)
model.fit(X)
labels = model.labels_    # -1 = noise/outlier
n_clusters = len(set(labels)) - (1 if -1 in labels else 0)
n_noise = list(labels).count(-1)
```

**Point Classifications:**
```
Core point:   has ≥ min_samples neighbors within eps radius
Border point: within eps of a core point, but < min_samples neighbors itself
Noise point:  not reachable from any core point → label = -1
```

**Why DBSCAN Beats K-Means on Non-Spherical Data:**
- K-Means enforces spherical shapes because it minimizes distance to centroids
- DBSCAN discovers shapes by connectivity — any two points within eps of a chain of core points belong to the same cluster

**DBSCAN Advantages:**
- No need to specify K
- Finds clusters of arbitrary shape
- Naturally identifies outliers
- Works in any dimensionality

**DBSCAN Weaknesses:**
- Two hyperparameters (`eps`, `min_samples`) require tuning
- Struggles with varying-density clusters (one `eps` doesn't fit all regions)
- Poor on very high-dimensional data

---

## 6.4 Agglomerative (Hierarchical) Clustering

**Core Idea:** Bottom-up. Start with each point as its own cluster. Repeatedly merge the two closest clusters. Produces a **dendrogram** (tree of all possible clusterings).

```python
from sklearn.cluster import AgglomerativeClustering
import scipy.cluster.hierarchy as sch

# Visualize dendrogram to choose cut level
dendrogram = sch.dendrogram(sch.linkage(X, method="ward"))
plt.show()

# Then cluster
model = AgglomerativeClustering(
    n_clusters=None,
    distance_threshold=10,     # Cut the dendrogram here
    linkage="ward"             # Minimize variance within clusters
)
model.fit(X)
labels = model.labels_
```

**Linkage Strategies (how to measure cluster-to-cluster distance):**

| Linkage | Measures | Best for |
|---|---|---|
| `ward` | Minimize within-cluster variance | Compact, equally-sized clusters |
| `complete` | Max distance between any two points | Well-separated clusters |
| `average` | Average pairwise distance | Balanced approach |
| `single` | Min distance (nearest neighbor) | Detecting chains |

---

## Clustering Algorithm Comparison

| Property | K-Means | Mean Shift | DBSCAN | Agglomerative |
|---|---|---|---|---|
| Specify K? | ✅ Required | ❌ Auto | ❌ Auto | ✅ or threshold |
| Cluster shape | Spherical | Density-based | Arbitrary | Arbitrary |
| Detects outliers | ❌ | ✅ | ✅ | ❌ |
| Speed | Fast | Slow | Medium | Slow (large data) |
| Sensitivity | Initialization | Bandwidth | eps, min_samples | Linkage, threshold |

---

# PART 7 — Model Evaluation Reference

## Classification Metrics

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score,
    f1_score, confusion_matrix, classification_report
)

# Accuracy: correct / total
acc = accuracy_score(y_test, y_pred)

# Confusion matrix
cm = confusion_matrix(y_test, y_pred)
# Rows = actual, Columns = predicted
# cm[i][j] = count of samples with true class i predicted as j

# Full report per class:
print(classification_report(y_test, y_pred))
```

**Precision vs Recall:**

```
Precision = TP / (TP + FP)   "Of all my 'positive' predictions, how many were right?"
Recall    = TP / (TP + FN)   "Of all actual positives, how many did I catch?"
F1        = 2 * (P * R) / (P + R)   Harmonic mean — use when both matter equally
```

**When class imbalance exists (e.g., 95% class A, 5% class B):**
- Accuracy is misleading — 95% accuracy just by predicting A always
- Use **F1**, **precision**, **recall** per class, or **ROC-AUC**

## Regression Metrics (Recap)

```python
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
import numpy as np

mse  = mean_squared_error(y_test, y_pred)
rmse = np.sqrt(mse)
mae  = mean_absolute_error(y_test, y_pred)   # More robust to outliers
r2   = r2_score(y_test, y_pred)
```

---

# PART 8 — Hyperparameter Tuning

```python
from sklearn.model_selection import GridSearchCV, RandomizedSearchCV

# Grid Search: exhaustive — tries every combination
param_grid = {
    "classifier__n_estimators": [50, 100, 200],
    "classifier__max_depth": [None, 5, 10],
}

grid_search = GridSearchCV(
    model_pipeline, param_grid,
    cv=5,                      # 5-fold cross-validation
    scoring="accuracy",
    n_jobs=-1                  # Use all CPU cores
)
grid_search.fit(X_train, y_train)

print(grid_search.best_params_)
print(grid_search.best_score_)

# Random Search: samples random combinations — better for large spaces
from scipy.stats import randint
param_dist = {
    "classifier__n_estimators": randint(50, 500),
    "classifier__max_depth": [None, 5, 10, 15, 20],
}
random_search = RandomizedSearchCV(
    model_pipeline, param_dist, n_iter=20, cv=5, random_state=42
)
```

---

## ⚠️ Key Edge Cases & Pitfalls

- **Data Leakage:** Never fit preprocessing (scaler, imputer) on test data. Use `Pipeline` to prevent this automatically.
- **Class Imbalance:** Accuracy is misleading. Use `class_weight="balanced"` in most sklearn classifiers, or oversample with SMOTE.
- **Overfitting vs Underfitting:** High train accuracy, low test accuracy = overfit. Low both = underfit. Use learning curves to diagnose.
- **The "best algorithm" fallacy:** No algorithm universally wins. Use cross-validation and compare multiple models empirically on your specific data.
- **Scaling before splitting:** A common beginner mistake. Scale after splitting. The scaler's parameters (mean, std) should only be derived from training data.
- **K-Means on non-spherical data:** K-Means will force spherical clusters even if the true clusters are crescents or rings. Always visualize before committing.
- **Feature importance ≠ causality:** A feature highly ranked by Random Forest predicts the target, but doesn't necessarily cause it.

---

## ❓ Active Recall Questions

- [ ] What is the difference between classification and regression? Give one real-world example of each.
- [ ] Why does OneHotEncoder prevent ordinal bias that LabelEncoder introduces?
- [ ] Why must you `fit` the StandardScaler only on training data, not the full dataset?
- [ ] Explain the kernel trick in SVM in plain language. Why is it useful?
- [ ] What is the bias-variance trade-off? How does Random Forest exploit it?
- [ ] What makes Gradient Boosting fundamentally different from Random Forest in how trees are built?
- [ ] You train a model that achieves 97% accuracy on an imbalanced dataset (95% class A). Is this a good model? What should you measure instead?
- [ ] K-Means is given crescent-shaped data. What will happen, and which algorithm should you use instead?
- [ ] What do `eps` and `min_samples` control in DBSCAN? What are "core points", "border points", and "noise points"?
- [ ] Explain what R² = -0.3 means. Is it a good or bad model?
- [ ] What is data leakage and how does `sklearn.Pipeline` prevent it?
- [ ] When would you prefer RMSE over MAE for regression evaluation?
- [ ] Why does the "curse of dimensionality" hurt KNN specifically?
- [ ] What is the elbow method and when is it used?
- [ ] Compare agglomerative clustering with K-Means: when would you choose each?
