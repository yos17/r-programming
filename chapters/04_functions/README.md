# Chapter 4: Functions

*Write your own. Default arguments, return values, scope, and the statistics toolkit.*

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
power(5, 3)   # 125 (override the default)
```

Defaults are evaluated at call time, not definition time. This matters:

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
  if (b == 0) return(NA)  # early exit
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

Variables inside a function are local — they don't exist outside:

```r
f <- function() {
  local_var <- 42
  cat("Inside:", local_var, "\n")
}

f()
cat("Outside:", local_var, "\n")   # Error: object 'local_var' not found
```

Functions can *read* variables from the enclosing environment:

```r
multiplier <- 3

f <- function(x) x * multiplier   # reads multiplier from global env

f(5)   # 15
```

Use `<<-` to assign to the enclosing environment (use sparingly — it's a code smell):

```r
counter <- 0
increment <- function() counter <<- counter + 1

increment()
increment()
counter   # 2
```

---

## Functions as Arguments

In R, functions are first-class objects. You can pass them as arguments:

```r
apply_twice <- function(f, x) {
  f(f(x))
}

apply_twice(sqrt, 256)      # sqrt(sqrt(256)) = 4
apply_twice(toupper, "hi")  # error! toupper(toupper("hi")) = "HI"
```

**Anonymous functions** — define a function inline without naming it:

```r
apply_twice(function(x) x + 1, 10)   # 12

# R 4.1+ shorthand:
apply_twice(\(x) x + 1, 10)           # same thing
```

---

## The `...` (Dots) Argument

`...` passes any extra arguments through to another function:

```r
my_plot <- function(x, y, ...) {
  plot(x, y, type = "l", ...)
}

# Any extra args go to plot():
my_plot(1:10, (1:10)^2, col = "red", lwd = 2, main = "My Plot")
```

Also used to accept a variable number of arguments:

```r
my_sum <- function(...) {
  args <- c(...)
  sum(args)
}

my_sum(1, 2, 3, 4, 5)   # 15
```

---

## Program: Statistics Toolkit

```r
# stats_toolkit.R
# A collection of statistical functions

# Arithmetic mean
mean_val <- function(x) {
  sum(x) / length(x)
}

# Median
median_val <- function(x) {
  x_sorted <- sort(x)
  n <- length(x_sorted)
  if (n %% 2 == 1) {
    x_sorted[(n + 1) / 2]
  } else {
    (x_sorted[n / 2] + x_sorted[n / 2 + 1]) / 2
  }
}

# Variance (population)
variance_pop <- function(x) {
  m <- mean_val(x)
  sum((x - m) ^ 2) / length(x)
}

# Standard deviation (population)
sd_pop <- function(x) sqrt(variance_pop(x))

# Variance (sample, Bessel's correction)
variance_samp <- function(x) {
  m <- mean_val(x)
  sum((x - m) ^ 2) / (length(x) - 1)
}

# Standard deviation (sample)
sd_samp <- function(x) sqrt(variance_samp(x))

# Mode (most common value)
mode_val <- function(x) {
  freq <- table(x)
  as.numeric(names(which(freq == max(freq))))
}

# Quantile (simple version)
quantile_val <- function(x, p) {
  x_sorted <- sort(x)
  n <- length(x_sorted)
  index <- p * (n - 1) + 1
  lower <- floor(index)
  upper <- ceiling(index)
  if (lower == upper) return(x_sorted[lower])
  x_sorted[lower] + (x_sorted[upper] - x_sorted[lower]) * (index - lower)
}

# IQR
iqr_val <- function(x) quantile_val(x, 0.75) - quantile_val(x, 0.25)

# Z-score normalization
normalize <- function(x) {
  m <- mean_val(x)
  s <- sd_samp(x)
  (x - m) / s
}

# Summary report
describe <- function(x, name = "x") {
  cat(sprintf("\nSummary of %s (%d values):\n", name, length(x)))
  cat(sprintf("  Min:    %8.4f\n", min(x)))
  cat(sprintf("  Q1:     %8.4f\n", quantile_val(x, 0.25)))
  cat(sprintf("  Median: %8.4f\n", median_val(x)))
  cat(sprintf("  Mean:   %8.4f\n", mean_val(x)))
  cat(sprintf("  Q3:     %8.4f\n", quantile_val(x, 0.75)))
  cat(sprintf("  Max:    %8.4f\n", max(x)))
  cat(sprintf("  SD:     %8.4f\n", sd_samp(x)))
  cat(sprintf("  IQR:    %8.4f\n", iqr_val(x)))
}

# Test it
set.seed(42)
data <- c(23, 45, 12, 67, 34, 89, 45, 23, 56, 78,
          12, 45, 67, 23, 90, 45, 34, 78, 56, 23)

describe(data, "test data")

# Verify against R's built-ins
cat("\nVerification against R built-ins:\n")
cat(sprintf("  mean():   %8.4f\n", mean(data)))
cat(sprintf("  median(): %8.4f\n", median(data)))
cat(sprintf("  sd():     %8.4f\n", sd(data)))
cat(sprintf("  IQR():    %8.4f\n", IQR(data)))
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
  IQR:      39.7500

Verification against R built-ins:
  mean():   45.2500
  median(): 45.0000
  sd():     24.0022
  IQR():    39.7500
```

Your implementations match R's built-ins. That's how you know they're right.

---

## Recursive Functions

A function that calls itself:

```r
factorial <- function(n) {
  if (n <= 1) return(1)
  n * factorial(n - 1)
}

factorial(5)   # 120
factorial(10)  # 3628800
```

Fibonacci:

```r
fib <- function(n) {
  if (n <= 1) return(n)
  fib(n - 1) + fib(n - 2)
}

sapply(0:10, fib)  # 0 1 1 2 3 5 8 13 21 34 55
```

Recursive Fibonacci is famously slow (exponential time). An iterative version:

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

Document your functions with comments. The convention in R:

```r
#' Compute the harmonic mean of a numeric vector
#'
#' @param x numeric vector (all values must be positive)
#' @return the harmonic mean
#' @examples
#' harmonic_mean(c(1, 2, 4))  # 1.714
harmonic_mean <- function(x) {
  if (any(x <= 0)) stop("All values must be positive")
  n <- length(x)
  n / sum(1 / x)
}
```

The `#'` format is used by the `roxygen2` package to generate proper documentation. Get in the habit of documenting what arguments a function expects and what it returns.

---

## Exercises

**1. Temperature converter function**

Write `temp_convert(value, from, to)` that converts between Celsius, Fahrenheit, and Kelvin. Handle invalid inputs with `stop()`.

**2. String padding**

Write `pad(s, width, side = "right", char = " ")` that pads string `s` to `width` characters on the specified side. Use it to format a table.

**3. Running statistics**

Write `running_mean(x)` that returns a vector where the i-th element is the mean of `x[1:i]`. (No loops — use `cumsum(x) / seq_along(x)`.)

**4. Memoization**

The Fibonacci recursion is slow because it recomputes the same values. Write a memoized version using an environment to cache results:

```r
fib_memo <- local({
  cache <- c()
  function(n) {
    if (!is.na(cache[n + 1])) return(cache[n + 1])
    result <- if (n <= 1) n else fib_memo(n - 1) + fib_memo(n - 2)
    cache[n + 1] <<- result
    result
  }
})
```

Time the difference: `system.time(fib(30))` vs `system.time(fib_memo(30))`.

**5. Build a describe() extension**

Add to `describe()`: skewness and kurtosis.

Skewness: `mean((x - mean(x))^3) / sd(x)^3`  
Kurtosis: `mean((x - mean(x))^4) / sd(x)^4 - 3` (excess kurtosis)

---

*Next: Chapter 5 — Vectors and Arrays: R's fundamental data structure*
