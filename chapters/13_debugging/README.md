# Chapter 13: Debugging

*Stop the program. Look inside. Fix it. The R debugger, browser(), and interactive tools.*

---

## The Problem with Print Debugging

Every programmer starts here:

```r
f <- function(x) {
  cat("x is:", x, "\n")   # "print debugging"
  result <- x * 2
  cat("result is:", result, "\n")
  result
}
```

It works. But it doesn't scale. When you have 10 functions calling each other, `cat()` everywhere becomes noise. You want to *stop* the program at a specific point and *look around*.

That's what `browser()` is for.

---

## browser() — R's Pry

In Ruby, you drop `binding.pry` to pause execution and open a REPL inside the running program. R's equivalent is `browser()`.

```r
f <- function(x, y) {
  z <- x + y
  browser()     # ← execution stops here
  z * 2
}

f(3, 4)
```

When `browser()` hits, R pauses and gives you an interactive prompt:

```
Called from: f(3, 4)
Browse[1]>
```

Now you're inside the function. Every variable in scope is available:

```r
Browse[1]> x
[1] 3
Browse[1]> y
[1] 4
Browse[1]> z
[1] 7
Browse[1]> ls()     # what's in scope?
[1] "x" "y" "z"
Browse[1]> z * 10   # try expressions
[1] 70
```

### Browser navigation commands

| Command | What it does |
|---------|-------------|
| `n` | Next line (step over) |
| `s` | Step into next function call |
| `f` | Finish current function (run to return) |
| `c` | Continue (run to next browser() or end) |
| `Q` | Quit the debugger |
| `where` | Show the call stack |
| Any expression | Evaluate it in current scope |

---

## Conditional browser()

Don't stop every time — only when something goes wrong:

```r
process_record <- function(x, i) {
  browser(expr = is.na(x))   # only stop if x is NA
  x * 2
}

data <- c(1, 2, NA, 4, 5)
sapply(data, process_record)
```

This stops only on the third element, where `x` is `NA`. No wading through the happy-path iterations.

```r
# Stop on iteration 100 of a loop
for (i in 1:200) {
  result <- compute(i)
  browser(expr = i == 100)
}
```

---

## debug() — Wrap a Function

Instead of adding `browser()` to source code, `debug()` makes R pause at the *start* of any function:

```r
debug(lm)         # debug R's built-in lm()!
lm(y ~ x, data=df)  # will stop at the first line of lm()
undebug(lm)       # turn it off
```

With your own functions:

```r
my_model <- function(data, formula) {
  df <- clean_data(data)
  fit <- lm(formula, data=df)
  summary(fit)
}

debug(my_model)     # will pause at first line next time called
my_model(df, y ~ x)
undebug(my_model)
```

`debugonce()` — debug just the next call, then auto-undebug:

```r
debugonce(my_model)
my_model(df, y ~ x)   # pauses once; no need to undebug
```

---

## traceback() — Where Did It Crash?

When you get an error, `traceback()` shows the call stack — every function that was called when the error occurred:

```r
f <- function(x) g(x)
g <- function(x) h(x)
h <- function(x) stop("Something went wrong!")

f(1)
# Error in h(x) : Something went wrong!

traceback()
# 3: h(x) called from g
# 2: g(x) called from f
# 1: f(1)
```

Read from bottom (where *you* called from) to top (where the error happened).

In RStudio, the traceback shows automatically in the **Console** after an error, and you can click each level to jump to that line.

---

## options(error = recover) — Drop Into the Stack

This is the most powerful: when an error occurs, R pauses and lets you inspect *any* frame on the call stack:

```r
options(error = recover)

f <- function(x) g(x)
g <- function(x) h(x)
h <- function(x) log(x)   # log(-1) produces NaN but not an error
                           # use stop() to trigger:
h <- function(x) { if (x < 0) stop("negative!"); log(x) }

f(-5)
```

```
Error in h(x) : negative!

Enter a frame number, or 0 to exit

1: f(-5)
2: #1: g(x)
3: #2: h(x)

Selection: 3
Called from: h(x)
Browse[1]> x
[1] -5
Browse[1]> Q
```

You pick which frame to inspect. This is like Pry's `caller` — you can walk up the call stack and see every variable at each level.

**Reset when done:**
```r
options(error = NULL)
```

---

## tryCatch() — Handle Errors Gracefully

Sometimes you don't want to crash — you want to catch the error and do something:

```r
safe_log <- function(x) {
  tryCatch(
    expr    = log(x),
    error   = function(e) {
      cat("Error:", conditionMessage(e), "\n")
      NA
    },
    warning = function(w) {
      cat("Warning:", conditionMessage(w), "\n")
      NaN
    }
  )
}

safe_log(10)     # 2.302585
safe_log(-1)     # Warning: NaNs produced → NaN
safe_log("hi")   # Error: non-numeric argument → NA
```

### withCallingHandlers() — Log but Continue

`tryCatch` stops the function at the error. `withCallingHandlers` lets the function keep going:

```r
withCallingHandlers(
  expr = {
    log(-1)   # produces warning
    log(10)   # still runs
  },
  warning = function(w) {
    cat("Caught warning:", conditionMessage(w), "\n")
    invokeRestart("muffleWarning")   # suppress the warning after logging
  }
)
```

---

## Program: A Debugging Session

Let's debug a real broken program step by step:

```r
# buggy_stats.R — this program has bugs. Let's find them.

calculate_cv <- function(x) {
  # Coefficient of variation = SD/mean
  sd(x) / mean(x) * 100
}

normalize_scores <- function(scores, group) {
  # Normalize scores within each group
  result <- numeric(length(scores))
  for (grp in unique(group)) {
    idx    <- group == grp
    subset <- scores[idx]
    cv     <- calculate_cv(subset)
    result[idx] <- (subset - mean(subset)) / sd(subset)
  }
  result
}

summarize_groups <- function(df) {
  groups <- unique(df$group)
  out    <- data.frame()
  for (g in groups) {
    sub <- df[df$group == g, ]
    out <- rbind(out, data.frame(
      group    = g,
      n        = nrow(sub),
      mean     = mean(sub$score),
      norm_avg = mean(sub$norm_score)
    ))
  }
  out
}

# Generate test data — includes a group with 1 member (will cause sd = NA)
set.seed(42)
df <- data.frame(
  id    = 1:20,
  score = c(rnorm(8, 70, 10), rnorm(8, 80, 8), 95, rnorm(3, 60, 5)),
  group = c(rep("A", 8), rep("B", 8), "C", rep("D", 3))
)

# This will silently produce NA for group C (single element, sd=NA)
df$norm_score <- normalize_scores(df$score, df$group)
result        <- summarize_groups(df)
print(result)
```

**Bug 1: Group C has only 1 member → `sd(subset)` = NA → all norms in C = NA.**

Diagnose:

```r
# Step 1: traceback / print to find where NA appears
which(is.na(df$norm_score))   # → rows with group C

# Step 2: add browser() to normalize_scores
normalize_scores <- function(scores, group) {
  result <- numeric(length(scores))
  for (grp in unique(group)) {
    idx    <- group == grp
    subset <- scores[idx]
    
    browser(expr = length(subset) < 2)   # stop on singleton groups
    
    cv     <- calculate_cv(subset)
    result[idx] <- (subset - mean(subset)) / sd(subset)
  }
  result
}

normalize_scores(df$score, df$group)
# Pauses on group C:
# Browse[1]> grp
# [1] "C"
# Browse[1]> subset
# [1] 95
# Browse[1]> sd(subset)
# [1] NA
```

**Fix:**

```r
normalize_scores <- function(scores, group) {
  result <- numeric(length(scores))
  for (grp in unique(group)) {
    idx    <- group == grp
    subset <- scores[idx]
    
    if (length(subset) < 2) {
      warning(sprintf("Group '%s' has only %d observation(s) — cannot normalize", 
                      grp, length(subset)))
      result[idx] <- 0   # or NA, depending on convention
      next
    }
    
    result[idx] <- (subset - mean(subset)) / sd(subset)
  }
  result
}
```

---

## RStudio Debugger

RStudio gives you a visual debugger on top of the same underlying tools.

**Setting a breakpoint:**
- Click in the grey margin to the left of a line number → a red dot appears
- Run the function → R pauses at that line automatically (same as `browser()`)

**The debug toolbar (appears when paused):**

```
▶ Next    ↓ Step Into    ↑ Step Out    ▶▶ Continue    ■ Stop
```

**The Environment panel** shows all variables in the current frame. Click through different frames in the **Traceback** panel.

**Source editor highlights** the current line in yellow.

This is equivalent to what PyCharm/RubyMine give you — all the browser() power, with a visual interface.

---

## Debugging in the pharmaverse Context

Clinical programming common pitfalls:

```r
# 1. Unexpected NAs after a join — always check
adsl_joined <- adsl %>% left_join(ex_derived, by="USUBJID")
na_check <- adsl_joined %>% filter(is.na(TRTSDT))
if (nrow(na_check) > 0) {
  cat(sprintf("WARNING: %d subjects with NA TRTSDT after join\n", nrow(na_check)))
  # browser() here to inspect
}

# 2. Unexpected row multiplication after join
before <- nrow(adsl)
adsl_joined <- adsl %>% left_join(ex_derived, by="USUBJID")
after  <- nrow(adsl_joined)
if (after != before) {
  cat(sprintf("Row count changed: %d → %d. Check for one-to-many!\n", before, after))
  # ex_derived probably has multiple rows per USUBJID
  ex_derived %>% count(USUBJID) %>% filter(n > 1) %>% print()
}

# 3. Silent date coercion — always verify
adsl <- adsl %>% derive_vars_dt(new_vars_prefix="TRTS", dtc=RFXSTDTC)
stopifnot(inherits(adsl$TRTSDT, "Date"))   # crash loudly if wrong

# 4. Wrap risky derivations in tryCatch
safe_derive <- function(df, var, expr) {
  tryCatch(
    dplyr::mutate(df, !!var := !!rlang::enexpr(expr)),
    error = function(e) {
      cat(sprintf("Error deriving %s: %s\n", var, conditionMessage(e)))
      dplyr::mutate(df, !!var := NA)
    }
  )
}
```

---

## Defensive Programming: stopifnot() and stop()

Don't let bugs propagate silently. Assert your assumptions:

```r
build_adsl <- function(dm, ex, ds) {
  # Assert inputs
  stopifnot(
    is.data.frame(dm),
    is.data.frame(ex),
    "USUBJID" %in% names(dm),
    "RFXSTDTC" %in% names(dm),
    nrow(dm) > 0
  )
  
  # Check uniqueness
  dupes <- dm %>% count(USUBJID) %>% filter(n > 1)
  if (nrow(dupes) > 0)
    stop(sprintf("DM has %d duplicate USUBJIDs: %s",
                 nrow(dupes), paste(head(dupes$USUBJID,3), collapse=", ")))
  
  # ... derivation ...
}
```

`stopifnot()` crashes with a clear message when any condition is FALSE. Better to crash loud and early than produce wrong results silently.

---

## Exercises

**1. Fix the buggy program**

Copy `buggy_stats.R`. Add a second bug: make `calculate_cv` crash when the mean is zero. Use `browser()` inside `normalize_scores` to find it without looking at `calculate_cv` directly. Then fix both bugs.

**2. options(error = recover)**

Write a chain: `a()` calls `b()` which calls `c()`. Make `c()` crash when its argument is negative. Set `options(error = recover)`. Call `a(-1)`. Explore frames 1, 2, and 3. What's in scope at each level?

**3. tryCatch batch processor**

Write a function `safe_process(files)` that reads a list of CSV files, processes each, and logs errors without stopping. Returns a list with `success` (list of results) and `errors` (list of filenames + error messages).

**4. Pharma defensive check**

Write `validate_adsl(adsl)` that checks:
- One row per USUBJID (`stopifnot`)
- TRTSDT ≤ TRTEDT for all subjects (`stop()` with the offending USUBJIDs)
- SAFFL is only "Y" or NA
- TRTDURD equals TRTEDT − TRTSDT + 1 (within tolerance of 0)

Returns invisibly TRUE if valid, crashes with a useful error message if not.

---

*That's debugging in R: `browser()` to stop and look, `traceback()` to find where it crashed, `options(error = recover)` to explore the whole stack, and `tryCatch()` to handle errors gracefully. The concepts are identical to Pry in Ruby — just different syntax.*
