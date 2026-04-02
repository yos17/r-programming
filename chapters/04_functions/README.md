# Chapter 4: Functions

*Write your own. Default arguments, return values, scope, and the statistics toolkit.*

---

## Where We Left Off

`analyze.R` parses a filename, validates it, and parses a filter string. The statistics logic is still missing — we've been printing what we *would* compute. Time to actually compute it.

The statistics we want:
- mean, median, standard deviation
- min, max, range
- a clean summary

We *could* write these inline. Better: extract them into functions. That way they're testable, reusable, and the main program stays short.

---

## Defining a Function

```r
square <- function(x) {
  x ^ 2
}

square(5)   # 25
square(3)   # 9
```

A function in R: `function(arguments) { body }`.

The last expression evaluated is returned. No `return()` needed (though you can use it).

```r
square <- function(x) x ^ 2   # one-liner, no braces needed
```

---

## Multiple Arguments

```r
power <- function(base, exponent) {
  base ^ exponent
}

power(2, 10)   # 1024
power(3, 3)    # 27
```

**Named arguments** — call in any order:

```r
power(exponent = 3, base = 2)   # 8
```

**Default arguments**:

```r
power <- function(base, exponent = 2) {
  base ^ exponent
}

power(5)      # 25  (exponent defaults to 2)
power(5, 3)   # 125 (override)
```

Defaults are evaluated at call time, not definition time:

```r
f <- function(x, y = x * 2) {
  cat(x, y, "\n")
}
f(5)    # 5 10  (y = 5 * 2)
f(5, 3) # 5 3
```

---

## Return Values

The last expression is returned. Use `return()` for early exits:

```r
safe_divide <- function(a, b) {
  if (b == 0) return(NA)
  a / b
}

safe_divide(10, 2)   # 5
safe_divide(10, 0)   # NA
```

Return multiple values with a **list**:

```r
min_max <- function(x) {
  list(minimum = min(x), maximum = max(x), range = max(x) - min(x))
}

result <- min_max(c(3, 1, 4, 1, 5, 9, 2, 6))
result$minimum   # 1
result$maximum   # 9
result$range     # 8
```

---

## Scope

Variables inside a function are local:

```r
f <- function() {
  local_var <- 42
  cat("Inside:", local_var, "\n")
}

f()
local_var   # Error: object 'local_var' not found
```

Functions can *read* variables from the enclosing environment:

```r
multiplier <- 3
f <- function(x) x * multiplier
f(5)   # 15
```

This is fine for constants. For everything else, pass arguments explicitly. Using `<<-` to modify external state from inside a function is a code smell — avoid it.

---

## Functions as Arguments

In R, functions are first-class objects. You can pass them as arguments:

```r
apply_twice <- function(f, x) {
  f(f(x))
}

apply_twice(sqrt, 256)      # sqrt(sqrt(256)) = 4
apply_twice(toupper, "hi")  # "HI" (toupper is idempotent)
```

**Anonymous functions** — define a function inline:

```r
apply_twice(function(x) x + 1, 10)   # 12

# R 4.1+ shorthand:
apply_twice(\(x) x + 1, 10)           # same
```

---

## The `...` (Dots) Argument

`...` passes extra arguments through to another function:

```r
my_sum <- function(...) {
  args <- c(...)
  sum(args)
}

my_sum(1, 2, 3, 4, 5)   # 15
```

---

## Building the Statistics Toolkit

Here's the right approach: write each statistic as a small function, then compose them.

Start minimal:

```r
compute_mean <- function(x) {
  sum(x) / length(x)
}
```

Test it. It works. Now add more:

```r
compute_median <- function(x) {
  x_sorted <- sort(x)
  n <- length(x_sorted)
  if (n %% 2 == 1) {
    x_sorted[(n + 1) / 2]
  } else {
    (x_sorted[n / 2] + x_sorted[n / 2 + 1]) / 2
  }
}

compute_sd <- function(x) {
  m <- compute_mean(x)
  sqrt(sum((x - m) ^ 2) / (length(x) - 1))  # sample SD
}

compute_quantile <- function(x, p) {
  x_sorted <- sort(x)
  n <- length(x_sorted)
  index <- p * (n - 1) + 1
  lower <- floor(index)
  upper <- ceiling(index)
  if (lower == upper) return(x_sorted[lower])
  x_sorted[lower] + (x_sorted[upper] - x_sorted[lower]) * (index - lower)
}
```

Now compose them into a summary:

```r
describe <- function(x, name = "x") {
  cat(sprintf("\nSummary of %s (%d values):\n", name, length(x)))
  cat(sprintf("  Min:    %8.4f\n", min(x)))
  cat(sprintf("  Q1:     %8.4f\n", compute_quantile(x, 0.25)))
  cat(sprintf("  Median: %8.4f\n", compute_median(x)))
  cat(sprintf("  Mean:   %8.4f\n", compute_mean(x)))
  cat(sprintf("  Q3:     %8.4f\n", compute_quantile(x, 0.75)))
  cat(sprintf("  Max:    %8.4f\n", max(x)))
  cat(sprintf("  SD:     %8.4f\n", compute_sd(x)))
}
```

Test it:

```r
set.seed(42)
data <- c(23, 45, 12, 67, 34, 89, 45, 23, 56, 78,
          12, 45, 67, 23, 90, 45, 34, 78, 56, 23)

describe(data, "test data")
```

Output:
```
Summary of test data (20 values):
  Min:      12.0000
  Q1:       23.0000
  Median:   45.0000
  Mean:     45.2500
  Q3:       62.7500
  Max:      90.0000
  SD:       24.0022
```

Verify against R's built-ins:

```r
cat(sprintf("  mean():   %8.4f\n", mean(data)))    # 45.2500 ✓
cat(sprintf("  sd():     %8.4f\n", sd(data)))       # 24.0022 ✓
```

Your implementations match. That's how you know they're right.

---

## Recursive Functions

A function that calls itself:

```r
factorial <- function(n) {
  if (n <= 1) return(1)
  n * factorial(n - 1)
}

factorial(5)   # 120
```

Recursive Fibonacci is famously slow (exponential time):

```r
fib <- function(n) {
  if (n <= 1) return(n)
  fib(n - 1) + fib(n - 2)
}
```

An iterative version is much faster:

```r
fib_fast <- function(n) {
  if (n <= 1) return(n)
  a <- 0; b <- 1
  for (i in seq_len(n - 1)) {
    temp <- a + b
    a <- b
    b <- temp
  }
  b
}
```

---

## Function Documentation

Document your functions. The convention in R:

```r
#' Compute the sample standard deviation
#'
#' @param x numeric vector (length >= 2)
#' @return standard deviation (sample, Bessel's correction)
#' @examples
#' compute_sd(c(2, 4, 4, 4, 5, 5, 7, 9))  # 2
compute_sd <- function(x) {
  if (length(x) < 2) stop("need at least 2 values")
  m <- compute_mean(x)
  sqrt(sum((x - m) ^ 2) / (length(x) - 1))
}
```

The `#'` format is used by the `roxygen2` package to generate proper documentation. Even without the package, it makes your code readable.

---

## analyze.R: Chapter 4 Version

What changed: added `compute_stats()`. The main program now calls it with fake data — real data loading comes in Chapter 7.

```r
# analyze.R — Chapter 4
# Usage: Rscript analyze.R <file.csv> [--filter 'col>value'] [--stat mean,sd,median]

args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv>\n")
  quit(status = 1)
}

filename <- args[1]

if (!file.exists(filename)) {
  cat("Error: file not found:", filename, "\n")
  quit(status = 1)
}

# --- Statistics functions ---

compute_mean <- function(x) sum(x) / length(x)

compute_sd <- function(x) {
  m <- compute_mean(x)
  sqrt(sum((x - m) ^ 2) / (length(x) - 1))
}

compute_median <- function(x) {
  x_sorted <- sort(x)
  n <- length(x_sorted)
  if (n %% 2 == 1) x_sorted[(n + 1) / 2]
  else (x_sorted[n / 2] + x_sorted[n / 2 + 1]) / 2
}

describe <- function(x, name = "column") {
  cat(sprintf("  %s: n=%d, mean=%.2f, sd=%.2f, min=%.2f, max=%.2f\n",
              name, length(x),
              compute_mean(x), compute_sd(x),
              min(x), max(x)))
}

# --- Main ---

cat("analyze.R\n")
cat("File:", filename, "\n")

# Parse --stat argument
stat_arg <- grep("^--stat=", args, value = TRUE)
requested_stats <- if (length(stat_arg) > 0) {
  strsplit(sub("^--stat=", "", stat_arg), ",")[[1]]
} else {
  c("mean", "sd", "median")
}

# Fake data — replaced in Chapter 7 with real CSV reading
values <- c(72000, 48000, 95000, 55000, 83000, 61000, 78000, 90000)
cat("\nColumn: salary (fake data)\n")
describe(values, "salary")
```

Run it:

```
Rscript analyze.R data.csv --stat=mean,sd
```

Output:
```
analyze.R
File: data.csv

Column: salary (fake data)
  salary: n=8, mean=72750.00, sd=16213.12, min=48000.00, max=95000.00
```

---

## Exercises

**1. Temperature converter function**

Write `temp_convert(value, from, to)` that converts between Celsius, Fahrenheit, and Kelvin. Handle invalid inputs with `stop()`.

**2. String padding**

Write `pad(s, width, side = "right", char = " ")` that pads string `s` to `width` characters. Use it to format a table.

**3. Running statistics**

Write `running_mean(x)` that returns a vector where element i is the mean of `x[1:i]`. No loops — use `cumsum(x) / seq_along(x)`.

**4. Build a describe() extension**

Add to `describe()`: skewness and kurtosis.

Skewness: `mean((x - mean(x))^3) / sd(x)^3`  
Kurtosis: `mean((x - mean(x))^4) / sd(x)^4 - 3` (excess kurtosis)

**5. The growing program (do this one)**

Add `compute_stats(x, stats)` to `analyze.R` where `stats` is a character vector like `c("mean", "sd", "median", "min", "max")`. The function computes only what's requested and returns a named list. So `--stat=mean,sd` returns `list(mean=..., sd=...)`.

This function will be called in Chapter 5 after we learn vectorized operations — you'll replace the loop-based implementations with `mean()`, `sd()`, etc. from base R.

---

*Next: Chapter 5 — Vectors and Arrays: operate on whole columns at once*
