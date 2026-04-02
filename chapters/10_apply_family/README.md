# Chapter 10: The Apply Family

*apply, lapply, sapply, tapply, Map, Reduce — R's functional tools.*

---

## Why Apply?

In R, explicit `for` loops are:
1. Slower than vectorized operations
2. More verbose
3. Less idiomatic

The apply family replaces most loops with cleaner, faster, more composable code.

---

## apply — Over Matrix Rows/Columns

```r
m <- matrix(1:12, nrow = 3)

apply(m, 1, sum)   # row sums:    6 22 38  (MARGIN=1 = rows)
apply(m, 2, sum)   # column sums: 6  9 12 15  (MARGIN=2 = cols)
apply(m, 1, mean)  # row means
apply(m, 2, sd)    # column standard deviations
```

`MARGIN = 1` applies over rows. `MARGIN = 2` applies over columns.

With a custom function:

```r
apply(m, 1, function(row) max(row) - min(row))   # range of each row
```

(For simple row/column operations, `rowSums`, `colSums`, `rowMeans`, `colMeans` are faster.)

---

## lapply / sapply — Over Lists and Vectors

```r
# lapply: always returns a list
lapply(1:5, function(x) x^2)
# list(1, 4, 9, 16, 25)

# sapply: simplifies to vector/matrix when possible
sapply(1:5, function(x) x^2)
# 1 4 9 16 25

# Both work on any vector, not just numbers
files <- list.files(".", pattern = "*.R")
sizes <- sapply(files, file.size)
```

`sapply` is `lapply` + simplification. Use `sapply` when you want a clean result; use `lapply` when you need a list or the result type is uncertain.

---

## vapply — Type-Safe Apply

`vapply` is like `sapply` but you specify the expected output type:

```r
vapply(1:5, function(x) x^2, FUN.VALUE = numeric(1))
# 1 4 9 16 25

# Crashes if result isn't the declared type:
vapply(1:5, function(x) as.character(x), FUN.VALUE = character(1))
```

Use `vapply` in production code — it's safer than `sapply` because it catches type mismatches.

---

## tapply — Apply by Group

Group a vector by a factor, then apply a function:

```r
scores  <- c(85, 72, 95, 68, 90, 78)
grades  <- c("B", "C", "A", "C", "A", "B")

tapply(scores, grades, mean)
# A    B    C
# 92.5 81.5 70.0
```

Equivalent to: for each unique value of `grades`, compute `mean(scores[grades == value])`.

---

## Map — Multiple Inputs

`Map` applies a function to multiple lists in parallel (like `mapply`):

```r
names  <- c("Alice", "Bob", "Carol")
scores <- c(92, 78, 95)

Map(function(n, s) paste(n, "scored", s), names, scores)
# list("Alice scored 92", "Bob scored 78", "Carol scored 95")
```

As a vector:

```r
mapply(function(n, s) paste(n, "scored", s), names, scores)
# "Alice scored 92" "Bob scored 78" "Carol scored 95"
```

---

## Reduce — Fold Over a List

`Reduce` applies a function cumulatively:

```r
Reduce("+", 1:5)          # ((((1+2)+3)+4)+5) = 15
Reduce("*", 1:5)          # 120 (5!)
Reduce(max, c(3,1,4,1,5)) # 5

# With accumulate=TRUE: return intermediate results
Reduce("+", 1:5, accumulate = TRUE)
# 1 3 6 10 15  (running sum)
```

Custom reduce — e.g., merge multiple data frames:

```r
dfs <- list(df1, df2, df3)
Reduce(function(a, b) merge(a, b, by = "id"), dfs)
```

---

## Filter, Find, Position

```r
x <- list(1, "hello", TRUE, 3.14, "world", 42L)

Filter(is.character, x)    # list("hello", "world")
Filter(is.numeric,   x)    # list(1, 3.14)

Find(is.character, x)      # "hello" (first match)
Position(is.character, x)  # 2 (index of first match)
```

---

## Program: Batch Data Cleaner

```r
# batch_cleaner.R
# Clean and transform multiple datasets using apply family

# ---- Simulate multiple messy datasets ----
set.seed(42)

make_messy_data <- function(n, noise = 0.1) {
  df <- data.frame(
    id     = 1:n,
    value  = round(rnorm(n, 100, 20), 2),
    label  = sample(c("A","B","C", NA), n, replace=TRUE, prob=c(.3,.3,.3,.1)),
    extra  = sample(c(runif(n*0.9), rep(NA, n*0.1))),
    stringsAsFactors = FALSE
  )
  # Introduce some outliers
  outlier_idx <- sample(n, max(1, round(n * noise)))
  df$value[outlier_idx] <- df$value[outlier_idx] * 5
  df
}

datasets <- lapply(1:5, function(i) make_messy_data(sample(50:100, 1)))
names(datasets) <- paste0("dataset_", 1:5)

# ---- Cleaning pipeline ----
clean_dataset <- function(df) {
  # 1. Remove rows with NA in key columns
  df <- df[!is.na(df$label), ]
  
  # 2. Remove outliers (> 3 SD from mean)
  m <- mean(df$value, na.rm = TRUE)
  s <- sd(df$value, na.rm = TRUE)
  df <- df[abs(df$value - m) < 3 * s, ]
  
  # 3. Normalize value to 0-1
  vmin <- min(df$value)
  vmax <- max(df$value)
  df$value_norm <- (df$value - vmin) / (vmax - vmin)
  
  # 4. Fill NA in extra with median
  med_extra <- median(df$extra, na.rm = TRUE)
  df$extra[is.na(df$extra)] <- med_extra
  
  df
}

cleaned <- lapply(datasets, clean_dataset)

# ---- Summary statistics per dataset ----
summarize_dataset <- function(df, name) {
  data.frame(
    dataset     = name,
    n_rows      = nrow(df),
    mean_value  = round(mean(df$value), 2),
    sd_value    = round(sd(df$value), 2),
    label_A_pct = round(100 * mean(df$label == "A"), 1),
    na_extra    = sum(is.na(df$extra))
  )
}

summary_list <- Map(summarize_dataset, cleaned, names(cleaned))
summary_df   <- do.call(rbind, summary_list)

cat("=== Cleaning Results ===\n")

# Original sizes
orig_sizes <- sapply(datasets, nrow)
clean_sizes <- sapply(cleaned, nrow)
removed <- orig_sizes - clean_sizes
cat("\nRows removed per dataset:\n")
for (i in seq_along(orig_sizes)) {
  pct <- 100 * removed[i] / orig_sizes[i]
  cat(sprintf("  %s: %d → %d (removed %d, %.1f%%)\n",
              names(orig_sizes)[i],
              orig_sizes[i], clean_sizes[i],
              removed[i], pct))
}

cat("\nClean dataset summaries:\n")
print(summary_df, row.names = FALSE)

# ---- Aggregate across all datasets ----
all_clean <- do.call(rbind, cleaned)
cat(sprintf("\nCombined: %d rows across %d datasets\n",
            nrow(all_clean), length(cleaned)))

# Group means by label across all datasets
group_means <- tapply(all_clean$value, all_clean$label, mean)
cat("\nMean value by label (all datasets combined):\n")
for (lbl in names(group_means)) {
  cat(sprintf("  %s: %.2f\n", lbl, group_means[[lbl]]))
}

# ---- Find datasets with highest variability ----
cv <- sapply(cleaned, function(df) sd(df$value) / mean(df$value))
cat("\nCoefficient of variation (higher = more variable):\n")
cv_sorted <- sort(cv, decreasing = TRUE)
for (nm in names(cv_sorted)) {
  cat(sprintf("  %-15s : %.3f\n", nm, cv_sorted[[nm]]))
}
```

---

## do.call

`do.call` calls a function with a list as arguments. Essential for combining apply results:

```r
# Combine a list of data frames
combined <- do.call(rbind, list_of_dfs)

# Pass variable number of arguments
do.call(paste, list("a", "b", "c", sep = "-"))  # "a-b-c"
```

---

## Exercises

**1. Apply practice**

Given a list of numeric vectors of different lengths:
```r
data <- list(a = c(1,5,3,2), b = c(10,2,8), c = c(4,4,4,4,4))
```
Using only apply-family functions (no for loops):
- Find the mean of each
- Find the range (max - min) of each
- Find the length of each
- Find the sum of all values across all lists

**2. tapply report**

Use the `patients` dataset from Chapter 8. Use `tapply` to compute:
- Mean cost by department
- Readmission rate by gender AND department (two-way tapply)
- Mean age by readmitted status

**3. Functional pipeline**

Write a `pipeline` function that takes a list of functions and applies them in sequence:
```r
pipeline <- function(x, ...) {
  fns <- list(...)
  Reduce(function(v, f) f(v), fns, x)
}

pipeline(c(1, -2, 3, -4, 5),
  function(x) x[x > 0],   # keep positive
  sum,                     # sum
  sqrt)                    # square root
```

**4. Parallel Map**

Use `Map` to compute a pairwise dot product between two lists of vectors:
```r
vecs1 <- list(c(1,2,3), c(4,5,6), c(7,8,9))
vecs2 <- list(c(1,0,0), c(0,1,0), c(0,0,1))
# Expected: 1, 5, 9
```

---

*Next: Chapter 11 — Packages and OOP: using CRAN and building your own*
