# Chapter 3: Control Flow

*if, for, while — making programs that decide and repeat.*

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
    "unknown"    # default (no name = default)
  )
}

cat(day_type("Monday"),   "\n")  # weekday
cat(day_type("Saturday"), "\n")  # weekend
cat(day_type("Holiday"),  "\n")  # unknown
```

Empty cases fall through to the next non-empty case (like the Monday–Friday chain above).

---

## Program: Number Guessing Game

```r
# guess.R
# A number guessing game

set.seed(NULL)  # use random seed
secret <- sample(1:100, 1)
attempts <- 0
max_attempts <- 7

cat("I'm thinking of a number between 1 and 100.\n")
cat("You have", max_attempts, "attempts.\n\n")

for (attempt in 1:max_attempts) {
  guess <- as.integer(readline("Your guess: "))
  
  if (is.na(guess)) {
    cat("Please enter a number.\n")
    next
  }
  
  attempts <- attempts + 1
  
  if (guess == secret) {
    cat("\nCorrect! You got it in", attempts, "attempt(s).\n")
    break
  } else if (guess < secret) {
    cat("Too low! ")
  } else {
    cat("Too high! ")
  }
  
  remaining <- max_attempts - attempt
  if (remaining > 0) {
    cat(remaining, "attempt(s) remaining.\n")
  } else {
    cat("\nOut of attempts! The answer was", secret, "\n")
  }
}
```

Run this in the Console (`source("guess.R")`), not with `Ctrl+Shift+Enter` — you need interactive input.

`readline()` reads a line from the user. `sample(1:100, 1)` picks one random number from 1 to 100.

---

## Nested Loops: Multiplication Table

```r
# times_table.R

n <- 10

# Header row
cat(sprintf("%4s", ""))
for (j in 1:n) cat(sprintf("%4d", j))
cat("\n")

# Separator
cat(strrep("-", 4 * (n + 1)), "\n")

# Table body
for (i in 1:n) {
  cat(sprintf("%3d|", i))
  for (j in 1:n) {
    cat(sprintf("%4d", i * j))
  }
  cat("\n")
}
```

Output (first few rows):
```
        1   2   3   4   5   6   7   8   9  10
----------------------------------------
   1|   1   2   3   4   5   6   7   8   9  10
   2|   2   4   6   8  10  12  14  16  18  20
   3|   3   6   9  12  15  18  21  24  27  30
```

`strrep(x, n)` repeats string `x` n times.

---

## A Note on Loops in R

R has a reputation for slow loops. The R way is often to avoid explicit loops and use vectorized operations instead:

```r
# Slow loop (R style — avoid for large data)
total <- 0
for (i in 1:1000000) total <- total + i

# Fast vectorized
total <- sum(1:1000000)
```

Both give the same result. The vectorized version is ~100x faster.

You'll see more of this in Chapters 5 and 10. For now, loops are fine for learning — just know there's usually a vectorized alternative.

---

## Program: FizzBuzz

The classic:

```r
# fizzbuzz.R

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

Both produce the same output. The second version runs as a single vectorized operation.

---

## Program: Prime Sieve (Sieve of Eratosthenes)

```r
# primes.R
# Find all primes up to n using the Sieve of Eratosthenes

sieve <- function(n) {
  if (n < 2) return(integer(0))
  
  is_prime <- rep(TRUE, n)
  is_prime[1] <- FALSE
  
  i <- 2
  while (i * i <= n) {
    if (is_prime[i]) {
      # Mark all multiples of i as not prime
      multiples <- seq(i * i, n, by = i)
      is_prime[multiples] <- FALSE
    }
    i <- i + 1
  }
  
  return(which(is_prime))
}

primes <- sieve(100)
cat("Primes up to 100:\n")
cat(primes, "\n")
cat("\nCount:", length(primes), "\n")
```

Output:
```
Primes up to 100:
2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97

Count: 25
```

This introduces:
- `rep(TRUE, n)` — creates a vector of n TRUEs
- `seq(from, to, by)` — generates a sequence with step
- `which(is_prime)` — returns indices where condition is TRUE

---

## Exercises

**1. Grade classifier**

Write a function `grade(score)` that returns:
- "A" for 90–100
- "B" for 80–89
- "C" for 70–79
- "D" for 60–69
- "F" for below 60

Test it: `grade(95)`, `grade(82)`, `grade(70)`, `grade(55)`

**2. Collatz sequence**

The Collatz conjecture: starting from any positive integer n:
- if n is even: n = n / 2
- if n is odd: n = 3n + 1
- repeat until n = 1

Write a `collatz(n)` function that returns the full sequence. How long is the sequence starting from 27?

**3. Sum of digits**

Write a function `digit_sum(n)` that computes the sum of digits of a positive integer. (Hint: `%% 10` gives the last digit, `%/% 10` removes the last digit.) `digit_sum(12345)` should return 15.

**4. Pattern printing**

Use nested loops to print:
```
*
**
***
****
*****
****
***
**
*
```

**5. Prime factorization**

Write `factorize(n)` that returns the prime factors of n.
`factorize(360)` should return `2 2 2 3 3 5`.

**6. Vectorized fizzbuzz**

The vectorized FizzBuzz above uses nested `ifelse`. Write a different version using R's string operations: build a character vector where you start with `as.character(1:100)`, then replace positions divisible by 3 with "Fizz", divisible by 5 with "Buzz", divisible by 15 with "FizzBuzz".

---

*Next: Chapter 4 — Functions: writing your own*
