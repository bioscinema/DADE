# DADE  
**Distributional Analysis of Diversity Effects**

## Overview

**DADE** (*Distributional Analysis of Diversity Effects*) is an R package that provides a diagnostic framework for testing **distributional shifts** in alpha diversity across biological groups. It focuses on detecting shifts in:

- **Center** (e.g., median)
- **Dispersion** (spread)
- **Asymmetry** (skewness)

DADE supports both **permutation-based** and **quantile-regression-based** testing strategies, with or without covariate adjustment.

---

## Installation

Install the development version directly from GitHub:
```r
# install.packages("devtools")
devtools::install_github("bioscinema/DADE")
```

## Functions

| Function               | Description                                         | Best Used When                          |
|------------------------|-----------------------------------------------------|-----------------------------------------|
| `dade_perm_pair()`     | Permutation test for **2 groups**                   | Exploratory tests (center, spread, skew) |
| `dade_perm_multi()`    | Permutation test for **>2 groups**                  | Center, dispersion or skew         |
| `dade_wald_contrast()` | Quantile-regression-based **contrast test**         | Covariate-adjusted center, dispersion, or skew    |


## 1. Permutation-Based Testing

### 1.1 `dade_perm_pair()`

Permutation-based comparison for **two groups** (e.g., group A vs group B).

```r
dade_perm_pair(response ~ group,
               data        = df,
               mode        = "center",     # or "dispersion", "skewness"
               alternative = "two.sided",  # only for "center"
               perm        = 999)
```

- `"center"`: Median difference test  
- `"dispersion"`: Interquartile range (IQR)-based permutation test (Brown-Forsythe variant)
- `"skewness"`: Quantile-based asymmetry score permutation test

---

### 1.2 `dade_perm_multi()`

Permutation-based comparison for **three or more groups** (center, dispersion, or skewness).

```r
dade_perm_multi(response ~ group,
                data = df,
                mode = "center",   # or "dispersion", "skewness"
                perm = 999)
```

- `"center"`: Tests for median shift using permutation ANOVA  
- `"dispersion"`: Interquartile range (IQR)-based permutation test (Brown-Forsythe variant)
- `"skewness"`: Quantile-based asymmetry permutation test across groups

---

## 2. Quantile Regression Contrast Test

### 2.1 `dade_wald_contrast()`

Wald-type test for linear contrasts of quantile regression coefficients across multiple quantile levels. Allows **confounder adjustment** and supports tests of `center (median)`, `dispersion`, and `tail asymmetry`.

```r
dade_wald_contrast(formula, data,
                   taus,           # e.g., c(0.25, 0.75)
                   contrast,       # numeric contrast vector
                   alternative = "two.sided",
                   kernel = "gaussian")
```

- `formula`: any valid R formula (e.g., `Shannon ~ group + Depth`)
- `taus`: quantiles of interest (e.g., `c(0.1, 0.5, 0.9)`)
- `contrast`: linear contrast vector applied to stacked coefficients
- `alternative`: `"two.sided"`, `"greater"`, or `"less"`

---

### 2.2 How to Construct the Contrast Vector  

The goal of `dade_wald_contrast()` is to test **linear contrasts** of quantile–regression coefficients.  
In practice this means answering questions like:

* *Is the group effect at the 75-th percentile larger than at the 25-th percentile?*  (**dispersion**)  
* *Are the lower and upper tails symmetric around the median?*  (**asymmetry**)  
* *Does the diversity difference persist after adjusting for sequencing depth or cohort?*

Below is a **step-by-step recipe** that anyone can follow and replicate.

---

#### Step 1  Build the quantile-regression design

Suppose you fit

```
Shannon ~ group + covariate1 + covariate2
```

and you want the 25-th and 75-th quantiles (τ = 0.25, 0.75).  
For each τ the model has *p = 4* coefficients:

1. **Intercept**  
2. **group** (e.g., A vs B)  
3. **covariate1**  
4. **covariate2**

`dade_wald_contrast()` **stacks** the coefficients τ-by-τ:

```
[Intercept τ0.25, group τ0.25, cov1 τ0.25, cov2 τ0.25,
 Intercept τ0.75, group τ0.75, cov1 τ0.75, cov2 τ0.75]
```

Hence the full vector has length *p × K = 4 × 2 = 8*.  
If you add more covariates or quantiles, the length grows automatically.

---
#### Step 2  Write down the scientific question as a contrast

A **contrast vector** is mostly 0’s with small non-zero weights (−1, +1, −2, …) in
the slots you wish to compare.

| What you want to test | How to place the weights |
|-----------------------|--------------------------|
| **Dispersion** – is *group* at τ0.75 larger than τ0.25? | Put −1 on the *group* slot of τ0.25 and +1 on the *group* slot of τ0.75 |
| **Asymmetry** – are tails symmetric? (`group_τ0.25 − 2×group_τ0.5 + group_τ0.75 = 0`) | Put +1, −2, +1 on successive *group* slots |

**Example A : dispersion (two quantiles)**  
```
length = 8
index 2 = group τ0.25 -> −1
index 6 = group τ0.75 -> +1
contrast = c(0, -1, 0, 0, 0, 1, 0, 0)
```
**Example B : asymmetry (three quantiles)**  

If K = 3 (τ = 0.1, 0.5, 0.9) and p = 4 ⇒ length = 12.
```
index 2 = group τ0.10 -> +1
index 6 = group τ0.50 -> -2
index 10 = group τ0.90 -> +1
contrast = c(0,1,0,0, 0,-2,0,0, 0,1,0,0)
```
---

#### Step 3  Check your work

1. *Length* of `contrast` must equal `p × K`.  
2. Non-zero elements must align with the same covariate across blocks.  
3. Sum of weights reflects the hypothesis (e.g., +1 −1 tests dispersion, +1 −2 +1 tests symmetry).
Center (median) shift – `dade_wald_contrast()` can also test a difference in medians.
Simply set `taus = 0.5` and choose a contrast that selects the group coefficient at that quantile (for a four-parameter model this is `contrast = c(0, 1, 0, 0))`.
In practice, however, we recommend fitting a single-quantile regression at τ = 0.5 and inspecting the Wald statistic via `summary()`, because it is quicker and yields the same inference.

---

### 2.3 Examples (Illustrative Only – Do Not Copy)

#### Dispersion difference (adjusting nothing)

```r
dade_wald_contrast(Shannon ~ group_prefix,
                   data     = plot_df,
                   taus     = c(0.25, 0.75),
                   contrast = c(0, -1, 0, 1),
                   alternative = "greater")
```

#### Dispersion difference (adjusting for sequencing depth)

```r
dade_wald_contrast(Shannon ~ group_prefix + Depth,
                   data     = plot_df,
                   taus     = c(0.25, 0.75),
                   contrast = c(0, -1, 0, 0, 1, 0),
                   alternative = "greater")
```

#### Dispersion difference (adjusting for cohort and depth)

```r
dade_wald_contrast(Shannon ~ group_prefix + Cohort + Depth,
                   data     = plot_df,
                   taus     = c(0.25, 0.75),
                   contrast = c(0, -1, 0, 0, 0, 1, 0, 0),
                   alternative = "greater")
```

#### Asymmetry difference (3 quantiles, unadjusted)

```r
dade_wald_contrast(Shannon ~ group_prefix,
                   data     = plot_df,
                   taus     = c(0.1, 0.5, 0.9),
                   contrast = c(0, 1, 0, -2, 0, 1),
                   alternative = "greater")
```

#### Asymmetry difference (adjusting for cohort and depth)

```r
dade_wald_contrast(Shannon ~ group_prefix + Cohort + Depth,
                   data     = plot_df,
                   taus     = c(0.1, 0.5, 0.9),
                   contrast = c(0, 1, 0, 0, 0, -2, 0, 0, 0, 1, 0, 0),
                   alternative = "greater")
```

---

## Workflow Summary

1. Use `dade_perm_pair()` or `dade_perm_multi()` for unadjusted, distribution-based testing.
2. Use `dade_wald_contrast()` when:
   - You want to adjust for covariates (e.g., sequencing depth).
   - You want formal tests of dispersion or tail asymmetry, or—if you prefer—a median (center) test; although for center a simple quantile regression followed by `summary()` is usually simpler.
3. Build your `contrast` vector carefully by indexing the relevant coefficient positions across quantile blocks.

---
