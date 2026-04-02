# Chapter 3: Control Flow

*if, for, while — making programs that decide and repeat.*

---

## Where We Left Off

`analyze.R` can read a filename argument and do unit conversions. It still can't read the file or filter data. Let's fix the filtering part — teach it to include or exclude rows based on a condition.

But first: to filter, we need to decide. To decide, we need `if`.

---

## if / else if / else

```r
x <- 15

if (x > 10) {
  cat("x is greater than 10\n")
} else if (x == 10) {
  cat("x is exactly 10\n")
} else {
  cat("x is less than 10\n")
}
```

Output: `x is greater than 10`

The condition must be a **single** logical value. `if (c(TRUE, FALSE))` gives a warning and uses only the first element.

**Inline if** — for short expressions:

```r
sign <- if (x > 0) "positive" else if (x < 0) "negative" else "zero"
```

**ifelse()** — vectorized if (operates on whole vectors):

```r
x <- c(-3, 0, 5, -1, 8)
ifelse(x > 0, "positive", "non-positive")
# [1] "non-positive" "non-positive" "positive" "non-positive" "positive"
```

---

## for Loops

```r
for (i in 1:5) {
  cat("i =", i, "\n")
}
```

Output:
```
i = 1
i = 2
i = 3
i = 4
i = 5
```

`1:5` creates the sequence `1, 2, 3, 4, 5`. You can loop over any vector:

```r
fruits <- c("apple", "banana", "cherry")
for (fruit in fruits) {
  cat("I like", fruit, "\n")
}
```

Loop over indices (when you need the index):

```r
fruits <- c("apple", "banana", "cherry")
for (i in seq_along(fruits)) {
  cat(i, ":", fruits[i], "\n")
}
```

`seq_along(x)` generates `1, 2, ..., length(x)`. Safer than `1:length(x)` because it handles empty vectors correctly.

---

## while Loops

```r
n <- 1
while (n <= 5) {
  cat(n, "")
  n <- n + 1
}
cat("\n")
```

Output: `1 2 3 4 5`

**repeat** — loop until explicit `break`:

```r
n <- 1
repeat {
  cat(n, "")
  n <- n + 1
  if (n > 5) break
}
cat("\n")
```

---

## break and next

`break` exits the loop. `next` skips to the next iteration (like `continue` in other languages).

```r
# Print only odd numbers, stop at 10
for (i in 1:20) {
  if (i > 10) break
  if (i %% 2 == 0) next
  cat(i, "")
}
cat("\n")
```

Output: `1 3 5 7 9`

---

## switch

For multiple alternatives, `switch` is cleaner than chains of `if/else`:

```r
day_type <- function(day) {
  switch(day,
    Monday    = ,
    Tuesday   = ,
    Wednesday = ,
    Thursday  = ,
    Friday    = "weekday",
    Saturday  = ,
    Sunday    = "weekend",
    "unknown"    # default
  )
}

cat(day_type("Monday"),   "\n")  # weekday
cat(day_type("Saturday"), "\n")  # weekend
cat(day_type("Holiday"),  "\n")  # unknown
```

Empty cases fall through to the next non-empty case.

---

## The Filtering Problem

We want to filter rows of data. Here's some fake data as parallel vectors (we'll use real data frames in Chapter 8):

```r
names   <- c("Alice", "Bob", "Carol", "Dave", "Eve")
salaries <- c(72000, 48000, 95000, 55000, 83000)
depts   <- c("Engineering", "Sales", "Engineering", "HR", "Sales")
```

**The loop way** (wrong way first — so you understand what's slow):

```r
# Loop over every row and check the condition
for (i in seq_along(names)) {
  if (salaries[i] > 60000) {
    cat(sprintf("  %s: $%d\n", names[i], salaries[i]))
  }
}
```

Output:
```
  Alice: $72000
  Carol: $95000
  Eve: $83000
```

It works. But loops are slow in R and verbose. We'll show a better way in Chapter 5.

---

## Parsing a Filter String

The tool will accept filters like `"salary>50000"` on the command line. We need to parse that string into a comparison:

```r
parse_filter <- function(filter_str) {
  # Match patterns like "salary>50000", "dept==Engineering", "age<=30"
  m <- regmatches(filter_str,
         regexpr("^(\\w+)(==|!=|>=|<=|>|<)(.+)$", filter_str))
  
  if (length(m) == 0) stop(paste("Invalid filter:", filter_str))
  
  parts <- regmatches(filter_str,
             regexec("^(\\w+)(==|!=|>=|<=|>|<)(.+)$", filter_str))[[1]]
  
  list(
    column = parts[2],
    op     = parts[3],
    value  = parts[4]
  )
}

f <- parse_filter("salary>50000")
cat("Column:", f$column, "\n")
cat("Op:    ", f$op,     "\n")
cat("Value: ", f$value,  "\n")
```

Output:
```
Column: salary
Op:     >
Value:  50000
```

---

## apply_filter(): Deciding Row by Row

Now we use `if/else` to apply the operator:

```r
apply_filter <- function(col_values, op, threshold) {
  threshold_num <- suppressWarnings(as.numeric(threshold))
  
  if (!is.na(threshold_num)) {
    # Numeric comparison
    switch(op,
      ">"  = col_values > threshold_num,
      "<"  = col_values < threshold_num,
      ">=" = col_values >= threshold_num,
      "<=" = col_values <= threshold_num,
      "==" = col_values == threshold_num,
      "!=" = col_values != threshold_num,
      stop(paste("Unknown operator:", op))
    )
  } else {
    # String comparison
    switch(op,
      "==" = col_values == threshold,
      "!=" = col_values != threshold,
      stop(paste("String comparison only supports == and !="))
    )
  }
}

# Test it
keep <- apply_filter(salaries, ">", "60000")
cat("Employees earning over $60k:\n")
for (i in which(keep)) {
  cat(sprintf("  %s ($%d)\n", names[i], salaries[i]))
}
```

---

## A Note on Loops in R

R has a reputation for slow loops. The R way is often to avoid explicit loops and use vectorized operations instead:

```r
# Slow loop
total <- 0
for (i in 1:1000000) total <- total + i

# Fast vectorized
total <- sum(1:1000000)
```

Same result. The vectorized version is ~100x faster. You'll see this in Chapter 5. For now, loops are fine for learning — just know the better way exists.

---

## FizzBuzz

The classic, two ways:

```r
# Loop version
for (i in 1:100) {
  if (i %% 15 == 0) {
    cat("FizzBuzz\n")
  } else if (i %% 3 == 0) {
    cat("Fizz\n")
  } else if (i %% 5 == 0) {
    cat("Buzz\n")
  } else {
    cat(i, "\n")
  }
}
```

Vectorized version (more R-idiomatic):

```r
x <- 1:100
result <- ifelse(x %% 15 == 0, "FizzBuzz",
          ifelse(x %% 3  == 0, "Fizz",
          ifelse(x %% 5  == 0, "Buzz",
                 as.character(x))))
cat(result, sep = "\n")
```

Same output. The second version runs as one vectorized operation — no explicit loop.

---

## Prime Sieve

```r
# Find all primes up to n using the Sieve of Eratosthenes

sieve <- function(n) {
  if (n < 2) return(integer(0))
  
  is_prime <- rep(TRUE, n)
  is_prime[1] <- FALSE
  
  i <- 2
  while (i * i <= n) {
    if (is_prime[i]) {
      multiples <- seq(i * i, n, by = i)
      is_prime[multiples] <- FALSE
    }
    i <- i + 1
  }
  
  which(is_prime)
}

primes <- sieve(100)
cat("Primes up to 100:\n")
cat(primes, "\n")
cat("Count:", length(primes), "\n")
```

New: `rep(TRUE, n)` creates a vector of n TRUEs. `seq(from, to, by)` generates a sequence with step. `which()` returns indices where condition is TRUE.

---

## analyze.R: Chapter 3 Version

What changed: added `--filter` argument parsing. The filter logic works on fake inline data for now — real CSV reading comes in Chapter 7.

```r
# analyze.R — Chapter 3
# Usage: Rscript analyze.R <file.csv> [--filter "col>value"]

args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv> [--filter 'col>value']\n")
  quit(status = 1)
}

filename <- args[1]

if (!file.exists(filename)) {
  cat("Error: file not found:", filename, "\n")
  quit(status = 1)
}

cat("analyze.R\n")
cat("File:", filename, "\n")

# Parse --filter argument
filter_arg <- grep("^--filter=", args, value = TRUE)
if (length(filter_arg) > 0) {
  filter_str <- sub("^--filter=", "", filter_arg)
  
  # Parse: column, operator, value
  parts <- regmatches(filter_str,
             regexec("^(\\w+)(==|!=|>=|<=|>|<)(.+)$", filter_str))[[1]]
  
  if (length(parts) == 4) {
    col_name  <- parts[2]
    op        <- parts[3]
    threshold <- parts[4]
    cat(sprintf("Filter: %s %s %s\n", col_name, op, threshold))
  } else {
    cat("Warning: could not parse filter:", filter_str, "\n")
  }
}
```

Run it:

```
Rscript analyze.R data.csv --filter=salary>50000
```

Output:
```
analyze.R
File: data.csv
Filter: salary > 50000
```

The filter is parsed. We'll actually apply it once we have data to filter.

---

## Exercises

**1. Grade classifier**

Write a function `grade(score)` that returns:
- "A" for 90–100, "B" for 80–89, "C" for 70–79, "D" for 60–69, "F" for below 60

Test it: `grade(95)`, `grade(82)`, `grade(70)`, `grade(55)`

**2. Collatz sequence**

The Collatz conjecture: starting from any positive integer n:
- if n is even: n = n / 2; if n is odd: n = 3n + 1; repeat until n = 1

Write `collatz(n)` that returns the full sequence. How long is the sequence starting from 27?

**3. Sum of digits**

Write `digit_sum(n)` that computes the sum of digits of a positive integer.
(`%% 10` gives the last digit, `%/% 10` removes the last digit.)
`digit_sum(12345)` should return 15.

**4. Vectorized FizzBuzz**

The vectorized FizzBuzz above uses nested `ifelse`. Write a different version: start with `as.character(1:100)`, then replace positions divisible by 3 with "Fizz", divisible by 5 with "Buzz", divisible by 15 with "FizzBuzz".

**5. Prime factorization**

Write `factorize(n)` that returns the prime factors of n.
`factorize(360)` should return `2 2 2 3 3 5`.

**6. The growing program (do this one)**

Extend `apply_filter()` so it handles multiple conditions joined by `&`:

```
--filter=salary>50000&dept==Engineering
```

Split on `&`, parse each condition separately, combine the logical vectors with `&`. This multi-filter will be essential in Chapters 7 and 8.

---

*Next: Chapter 4 — Functions: extract the statistics logic into reusable pieces*
