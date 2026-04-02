# Chapter 5: Vectors and Arrays

*R's native data structure. Everything is a vector. Vectorized thinking.*

---

## Where We Left Off

`analyze.R` has a `compute_stats()` function from Chapter 4. It works, but internally it uses a loop to process rows. We have this:

```r
total <- 0
for (i in seq_along(values)) {
  total <- total + values[i]
}
mean_val <- total / length(values)
```

The R way:

```r
mean_val <- mean(values)
```

Not just shorter — fundamentally different. In R, operations on vectors are implemented in compiled C code. They're not loops at the R level; they're single operations on contiguous memory. This chapter explains why — and rewires how you think about data.

---

## Everything Is a Vector

In R, there are no scalars. A "single number" is a vector of length 1.

```r
x <- 42
length(x)    # 1
is.vector(x) # TRUE
```

This is why `print(42)` shows `[1] 42` — the `[1]` is the index of the first element displayed on that line.

---

## Creating Vectors

```r
# c() — combine
x <- c(1, 2, 3, 4, 5)

# Sequences
1:10                     # 1 2 3 4 5 6 7 8 9 10
seq(1, 10, by = 2)       # 1 3 5 7 9
seq(0, 1, length.out = 5) # 0.00 0.25 0.50 0.75 1.00

# Repetition
rep(0, 5)                # 0 0 0 0 0
rep(c(1,2,3), 3)         # 1 2 3 1 2 3 1 2 3
rep(c(1,2,3), each = 3)  # 1 1 1 2 2 2 3 3 3

# Character
fruits <- c("apple", "banana", "cherry")

# Logical
flags <- c(TRUE, FALSE, TRUE, TRUE, FALSE)
```

Vectors are homogeneous — all elements must be the same type. Mixed types are coerced:

```r
c(1, "hello", TRUE)   # "1" "hello" "TRUE" — all character
c(1, TRUE, FALSE)     # 1 1 0 — all numeric
```

---

## Indexing

R uses 1-based indexing (not 0-based like most languages).

```r
x <- c(10, 20, 30, 40, 50)

x[1]          # 10
x[5]          # 50
x[c(1,3,5)]   # 10 30 50
x[2:4]        # 20 30 40
x[-1]         # 20 30 40 50 — drop first element
x[-c(1,5)]    # 20 30 40
```

**Logical indexing** — select elements where condition is TRUE:

```r
x <- c(3, 1, 4, 1, 5, 9, 2, 6)

x[x > 3]            # 4 5 9 6
x[x %% 2 == 0]      # 4 2 6
x[x > mean(x)]      # 4 5 9 6
```

This replaces most filtering loops. Compare:

```r
# Loop way (slow, verbose)
result <- c()
for (v in x) {
  if (v > mean(x)) result <- c(result, v)
}

# Vector way (fast, direct)
result <- x[x > mean(x)]
```

**Named indexing**:

```r
scores <- c(Alice = 92, Bob = 78, Carol = 95, Dave = 84)
scores["Alice"]          # 92
scores[c("Bob","Carol")] # Bob 78, Carol 95
```

---

## Vectorized Operations

R's superpower: operations apply to all elements at once.

```r
x <- c(1, 2, 3, 4, 5)

x + 10        # 11 12 13 14 15
x * 2         # 2 4 6 8 10
x ^ 2         # 1 4 9 16 25
sqrt(x)       # 1.000 1.414 1.732 2.000 2.236
log(x)        # natural log of each element
```

Operations between two vectors: element-by-element:

```r
a <- c(1, 2, 3)
b <- c(10, 20, 30)
a + b    # 11 22 33
a * b    # 10 40 90
```

**Recycling** — if lengths don't match, the shorter vector repeats:

```r
x <- c(1, 2, 3, 4, 6)
x + c(10, 100)   # 11 102 13 104 16 — recycled: 10,100,10,100,10
```

Recycling is powerful but can cause silent bugs. R warns when lengths aren't multiples.

---

## Vector Functions

```r
x <- c(3, 1, 4, 1, 5, 9, 2, 6, 5, 3)

length(x)         # 10
sum(x)            # 39
prod(x)           # product of all elements
cumsum(x)         # running sum
diff(x)           # differences: x[i+1] - x[i]

sort(x)           # sort ascending
order(x)          # indices that would sort x
rev(x)            # reverse

which(x > 4)      # indices where condition is TRUE
which.min(x)      # index of minimum
which.max(x)      # index of maximum

any(x > 8)        # TRUE if any element satisfies condition
all(x > 0)        # TRUE if all elements satisfy condition
```

---

## Replacing the Loop-Based Stats

In Chapter 4 we built `compute_mean`, `compute_sd`, etc. with loops. Now replace them:

```r
# Chapter 4: loop-based (educational but slow)
compute_mean <- function(x) {
  total <- 0
  for (v in x) total <- total + v
  total / length(x)
}

# Chapter 5: vectorized (R's way)
compute_mean <- function(x, na.rm = FALSE) {
  if (na.rm) x <- x[!is.na(x)]
  sum(x) / length(x)
}
```

The loop version is fine for learning. The vectorized version is what you'd actually write.

`na.rm = FALSE` is a default argument: by default, NAs are not removed. Pass `na.rm = TRUE` to ignore them. This matches R's built-in `mean()` interface.

Now update `describe()`:

```r
describe <- function(x, name = "column", na.rm = TRUE) {
  if (na.rm) x <- x[!is.na(x)]
  n_total <- length(x) + sum(is.na(x))  # count before removal
  n_valid <- length(x)
  n_na    <- n_total - n_valid
  
  cat(sprintf("  %s:\n", name))
  cat(sprintf("    n = %d", n_valid))
  if (n_na > 0) cat(sprintf(" (%d NAs omitted)", n_na))
  cat("\n")
  cat(sprintf("    mean   = %.4f\n", mean(x)))
  cat(sprintf("    sd     = %.4f\n", sd(x)))
  cat(sprintf("    median = %.4f\n", median(x)))
  cat(sprintf("    min    = %.4f\n", min(x)))
  cat(sprintf("    max    = %.4f\n", max(x)))
}
```

---

## Matrices

A matrix is a 2D vector with dimensions:

```r
m <- matrix(1:12, nrow = 3, ncol = 4)
m
```

```
     [,1] [,2] [,3] [,4]
[1,]    1    4    7   10
[2,]    2    5    8   11
[3,]    3    6    9   12
```

R fills by column (column-major order). Use `byrow = TRUE` to fill by row.

**Indexing**:

```r
m[2, 3]     # row 2, column 3
m[1, ]      # entire row 1
m[, 2]      # entire column 2
```

**Fast row/column operations**:

```r
rowSums(m)    # faster than apply(m, 1, sum)
colMeans(m)   # faster than apply(m, 2, mean)
```

---

## Grade Analyzer

```r
# grades.R
students <- c("Alice","Bob","Carol","Dave","Eve","Frank","Grace","Henry","Iris","Jack")

scores <- matrix(c(
  85, 92, 78, 88,
  72, 68, 75, 80,
  95, 98, 92, 96,
  60, 55, 62, 58,
  88, 85, 90, 87,
  78, 82, 79, 84,
  93, 91, 95, 89,
  65, 70, 68, 72,
  82, 87, 84, 90,
  75, 79, 77, 81
), nrow = 10, byrow = TRUE,
dimnames = list(students, paste("Test", 1:4)))

avg <- rowMeans(scores)
letter_grade <- function(s) ifelse(s >= 90, "A", ifelse(s >= 80, "B",
                              ifelse(s >= 70, "C", ifelse(s >= 60, "D", "F"))))
grades <- letter_grade(avg)

cat(sprintf("%-10s  %5s  %5s  %5s  %5s  %6s  %s\n",
            "Student", "T1", "T2", "T3", "T4", "Avg", "Grade"))
cat(strrep("-", 48), "\n")
for (i in seq_along(students)) {
  cat(sprintf("%-10s  %5.1f  %5.1f  %5.1f  %5.1f  %6.2f  %s\n",
              students[i], scores[i,1], scores[i,2],
              scores[i,3], scores[i,4], avg[i], grades[i]))
}
cat(sprintf("\nHighest: %s (%.2f)\n", students[which.max(avg)], max(avg)))
cat(sprintf("Lowest:  %s (%.2f)\n", students[which.min(avg)], min(avg)))
```

Every computation here is vectorized: `rowMeans`, `ifelse`, `which.max`. No explicit loop needed for the calculations — only for the formatted output.

---

## analyze.R: Chapter 5 Version

What changed: `describe()` and `compute_stats()` now use vectorized operations. NA handling added.

```r
# analyze.R — Chapter 5 (key changes shown)

# --- Statistics functions (now vectorized) ---

compute_stats <- function(x, stats = c("mean","sd","median","min","max"),
                          na.rm = TRUE) {
  if (na.rm) x <- x[!is.na(x)]
  result <- list()
  if ("mean"   %in% stats) result$mean   <- mean(x)
  if ("sd"     %in% stats) result$sd     <- sd(x)
  if ("median" %in% stats) result$median <- median(x)
  if ("min"    %in% stats) result$min    <- min(x)
  if ("max"    %in% stats) result$max    <- max(x)
  if ("n"      %in% stats) result$n      <- length(x)
  result
}

print_stats <- function(stats_list, name) {
  cat(sprintf("  %s:\n", name))
  for (nm in names(stats_list)) {
    cat(sprintf("    %-8s = %.4f\n", nm, stats_list[[nm]]))
  }
}
```

---

## Exercises

**1. Vector operations**

Given `x <- c(4, 7, 2, 9, 1, 5, 8, 3, 6)`:
- Extract elements greater than 5
- Extract every other element (indices 1, 3, 5, ...)
- Replace all elements less than 3 with 0
- Find the 3rd largest element (use `sort`, index from the end)
- Compute the running mean: `cumsum(x) / seq_along(x)`

**2. Matrix operations**

Create a 5×5 matrix filled with 1–25, by row. Then:
- Extract the diagonal (use `diag()`)
- Compute row sums with `rowSums()`
- Extract the 3×3 submatrix from rows 2–4, columns 2–4
- Replace all values greater than 15 with NA

**3. Moving average**

Write `moving_avg(x, k)` that computes the k-period moving average. For `x <- c(1, 2, 3, 4, 5, 6, 7)` and `k = 3`, the result should be `NA NA 2 3 4 5 6`. No loop needed — use `cumsum` and shifted indices.

**4. Grade analysis extension**

Extend the grade analyzer to:
- Weight the tests: T1=20%, T2=20%, T3=25%, T4=35%
- Find students who improved (T4 > T1)
- Compute the correlation between T1 and T4: `cor(scores[,1], scores[,4])`

**5. The growing program (do this one)**

`analyze.R` currently gets passed column data as a hardcoded vector. Change `compute_stats()` to also handle character vectors — for strings, report: n (count), n_unique (unique values), top value (most common). Use `table()` and `which.max()`.

```r
compute_stats_char <- function(x, na.rm = TRUE) {
  if (na.rm) x <- x[!is.na(x)]
  freq <- table(x)
  list(
    n        = length(x),
    n_unique = length(unique(x)),
    top      = names(which.max(freq))
  )
}
```

In Chapter 6, this character-column handling gets extended with regex and string parsing.

---

*Next: Chapter 6 — Strings: parse string columns, add regex filters*
