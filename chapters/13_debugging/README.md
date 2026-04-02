# Chapter 13: Debugging

*Stop the program. Look inside. Fix it.*

---

## Where We Left Off

`analyze.R` works on clean data. Let's break it — deliberately — in three ways that reflect real-world bugs, then fix each one using R's debugging tools.

Here are the three bugs we'll plant:

1. **Silent wrong answer**: a column of salaries stored as strings. Filtering works, stats silently return the wrong type.
2. **Crash with unhelpful error**: a single-row group in the data. `sd()` returns `NA`, which propagates through `sprintf` and throws a cryptic error.
3. **Off-by-one in filtering**: comparing strings to numbers yields unexpected results — all rows pass a numeric filter because string comparison uses lexicographic order.

---

## The Problem with Print Debugging

Every programmer starts here:

```r
cat("x is:", x, "\n")
cat("result is:", result, "\n")
```

It works. But when you have 10 functions calling each other, `cat()` everywhere becomes noise. You want to *stop* the program at a specific point and *look around*.

That's what `browser()` is for.

---

## browser() — Stop and Look Inside

```r
f <- function(x, y) {
  z <- x + y
  browser()     # execution stops here
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
Browse[1]> z
[1] 7
Browse[1]> ls()
[1] "x" "y" "z"
Browse[1]> z * 10
[1] 70
```

### Browser navigation commands

| Command | What it does |
|---------|-------------|
| `n` | Next line (step over) |
| `s` | Step into next function call |
| `f` | Finish current function (run to return) |
| `c` | Continue to next `browser()` or end |
| `Q` | Quit debugger |
| `where` | Show call stack |
| Any expression | Evaluate in current scope |

---

## Conditional browser()

Don't stop every time — only when something goes wrong:

```r
process_record <- function(x, i) {
  browser(expr = is.na(x))   # only stop if x is NA
  x * 2
}

data <- c(1, 2, NA, 4, 5)
sapply(data, function(x) process_record(x, NULL))
```

This pauses only on the third element.

```r
# Stop on a specific iteration
for (i in 1:200) {
  result <- compute(i)
  browser(expr = i == 100)
}
```

---

## debug() — Wrap a Function

Instead of adding `browser()` to source code, `debug()` pauses at the *start* of any function:

```r
debug(my_function)       # will pause at first line next call
my_function(data)
undebug(my_function)     # turn it off

debugonce(my_function)   # debug just once, auto-undebug
my_function(data)        # pauses; next call runs normally
```

You can even debug R's own built-in functions:

```r
debug(lm)
lm(mpg ~ cyl, data = mtcars)
undebug(lm)
```

---

## traceback() — Where Did It Crash?

When you get an error, `traceback()` shows the call stack:

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

Read from bottom (where *you* called) to top (where the error happened).

---

## options(error = recover) — Walk the Stack

Most powerful: when an error occurs, drop into any frame on the call stack.

```r
options(error = recover)

f <- function(x) g(x)
g <- function(x) h(x)
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
Browse[1]> x
[1] -5
Browse[1]> Q
```

You pick which frame to inspect. You can walk up the entire call stack and see every variable at every level.

Reset when done:
```r
options(error = NULL)
```

---

## Bug 1: Silent Wrong Answer (String Salaries)

Plant the bug: some salary values got imported as strings.

```r
# employees_buggy.csv — salary column has some quoted strings
# "Alice",50000,Engineering
# "Bob","not disclosed",Sales
# "Carol",72000,Engineering
```

What happens when `analyze.R` reads this?

```r
df <- read.csv("employees_buggy.csv", stringsAsFactors = FALSE)
class(df$salary)
# [1] "character"   ← because one value is "not disclosed"
```

The whole column is now character. When `detect_type()` runs, it sees non-numeric values and labels it `"character"`. The stats are skipped. No error — just silence.

**Diagnose with browser():**

```r
detect_type <- function(x) {
  x_clean <- x[!is.na(x)]
  browser(expr = length(x_clean) > 0 && 
                 all(sapply(x_clean, function(v) !is.na(suppressWarnings(as.numeric(v))))) == FALSE &&
                 any(!is.na(suppressWarnings(as.numeric(x_clean)))))
  # stops when: column has some numbers but not all
  # ...
}
```

Simpler diagnostic:

```r
# After reading, check each column
for (col in names(df)) {
  x <- df[[col]]
  nums <- suppressWarnings(as.numeric(x))
  n_parseable  <- sum(!is.na(nums))
  n_total      <- sum(!is.na(x))
  if (n_parseable > 0 && n_parseable < n_total) {
    cat(sprintf("Warning: column '%s' is %d/%d numeric — treating as character\n",
                col, n_parseable, n_total))
    cat(sprintf("  Non-numeric values: %s\n",
                paste(x[is.na(nums) & !is.na(x)][1:3], collapse=", ")))
  }
}
```

**Fix:** report the non-numeric values. Ask whether to coerce (setting non-parseable to NA) or skip the column.

```r
detect_type_verbose <- function(x, col_name = "?") {
  x_clean <- x[!is.na(x)]
  nums <- suppressWarnings(as.numeric(x_clean))
  n_numeric <- sum(!is.na(nums))
  
  if (n_numeric == length(x_clean)) return("numeric")
  if (n_numeric > 0) {
    # Partially numeric
    bad_vals <- x_clean[is.na(nums)]
    cat(sprintf("  Note: '%s' has %d non-numeric values (e.g. '%s') — column treated as character\n",
                col_name, length(bad_vals), bad_vals[1]))
    return("character")
  }
  # ... rest of type detection
  "character"
}
```

---

## Bug 2: Single-Row Group (sd = NA)

Plant the bug: add one employee to a new department "Legal" with only one person.

```r
df <- rbind(df, data.frame(name="Zoe", dept="Legal", salary=95000, years=1))
```

When grouped by `dept`, Legal has n=1. `sd()` of one value is `NA`.

The bug surfaces in `report()`:

```r
# In report(), when formatting grouped results:
sprintf("%.4g", NA_real_)
# Error in sprintf("%.4g", NA_real_) : 
#   argument 2 (type 'double') cannot be handled by '%g'
```

Wait — that's a different error. Let's trace it:

```r
options(error = recover)
Rscript analyze.R employees_with_legal.csv --group=dept --stat=n,mean,sd
```

```
Error in sprintf("%.4g", NA_real_) ...

Enter a frame number, or 0 to exit
1: report.Dataset(ds, ...)
2: sprintf("%.4g", NA_real_)

Selection: 1
Browse[1]> col
[1] "salary"
Browse[1]> grp
[1] "Legal"
Browse[1]> res[["Legal"]]
$n
[1] 1
$mean
[1] 95000
$sd
[1] NA
```

Found it. The SD is NA for a single-row group, and `sprintf("%.4g", NA)` crashes.

**Fix:** handle NA in the formatter:

```r
fmt_num <- function(v) {
  if (is.na(v)) return("NA")
  sprintf(paste0("%.", digits, "g"), v)
}
```

And add a warning in `compute_stats()`:

```r
if ("sd" %in% stats) {
  result$sd <- if (length(x) > 1) sd(x) else {
    warning(sprintf("sd: only %d observation(s)", length(x)))
    NA_real_
  }
}
```

---

## Bug 3: String vs Numeric Comparison

Plant the bug: the salary column gets read as character (see Bug 1). Now apply a numeric filter:

```
--filter=salary>60000
```

`apply_filter()` detects that `"60000"` is numeric, so it tries:

```r
col_values > 60000
```

But `col_values` is a character vector like `c("50000","72000","not disclosed")`. R coerces it to numeric — `"not disclosed"` becomes NA. The filter keeps rows where `salary > 60000`, but the comparison is between characters, not numbers.

Actually let's check:

```r
c("50000","72000","100000") > 60000
# [1] FALSE FALSE FALSE   ← ALL FALSE!
```

Because `60000` (numeric) gets coerced to `"60000"` (character), and the comparison is lexicographic: `"50000" > "60000"` is FALSE, `"72000" > "60000"` is TRUE... actually:

```r
c("50000","72000","100000") > "60000"
# [1] FALSE  TRUE FALSE
```

`"100000" > "60000"` is FALSE because "1" < "6" lexicographically. So the 100k employee gets excluded even though 100000 > 60000 numerically.

This is the silent wrong answer bug. No error. Wrong results.

**Diagnose with browser():**

```r
apply_filter <- function(col_values, op, threshold) {
  browser(expr = is.character(col_values) && !is.na(suppressWarnings(as.numeric(threshold))))
  # stops when filtering a string column with a numeric threshold
  # ...
}
```

**Fix:** detect the mismatch and warn:

```r
apply_filter <- function(col_values, op, threshold) {
  threshold_num <- suppressWarnings(as.numeric(threshold))
  
  if (!is.na(threshold_num) && is.character(col_values)) {
    # Try to coerce column
    col_num <- suppressWarnings(as.numeric(col_values))
    if (all(is.na(col_num) & !is.na(col_values))) {
      stop(sprintf("Cannot apply numeric filter to non-numeric column"))
    }
    if (any(is.na(col_num) & !is.na(col_values))) {
      warning(sprintf("Some values could not be parsed as numeric and will be excluded"))
    }
    col_values <- col_num
  }
  # ... rest of function
}
```

---

## tryCatch() — Handle Errors Gracefully

Sometimes you don't want to crash — you want to catch the error and continue:

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
safe_log(-1)     # Warning → NaN
safe_log("hi")   # Error → NA
```

In `analyze.R`, wrap per-column stats in `tryCatch` so one bad column doesn't abort the whole report:

```r
safe_compute_stats <- function(x, stats, col_name) {
  tryCatch(
    compute_stats(x, stats),
    error = function(e) {
      cat(sprintf("  Warning: could not compute stats for '%s': %s\n",
                  col_name, conditionMessage(e)))
      NULL
    }
  )
}
```

---

## Defensive Programming

Don't let bugs propagate silently. Assert your assumptions:

```r
analyze_column <- function(x, col_name, config) {
  stopifnot(
    is.numeric(x),
    length(x) > 0,
    !is.null(col_name),
    is.list(config)
  )
  # ...
}
```

`stopifnot()` crashes with a clear message when any condition is FALSE. Better to fail loud and early than produce wrong results silently.

For user-facing errors, `stop()` with a helpful message:

```r
if (!col_name %in% names(df)) {
  stop(sprintf("Column '%s' not found. Available columns: %s",
               col_name, paste(names(df), collapse=", ")))
}
```

---

## RStudio Debugger

Everything above also works in RStudio's visual debugger:

**Setting a breakpoint:**
- Click in the grey margin to the left of a line number → red dot
- Run the function → R pauses at that line (same as `browser()`)

**The debug toolbar:**
```
▶ Next    ↓ Step Into    ↑ Step Out    ▶▶ Continue    ■ Stop
```

The **Environment panel** shows all variables in the current frame. The **Traceback panel** lets you click through frames.

---

## Summary: The Debugging Toolkit

| Tool | Use when |
|------|----------|
| `browser()` | Stop inside a function and look around |
| `browser(expr=cond)` | Stop only when condition is TRUE |
| `debug(f)` | Pause at start of function f |
| `debugonce(f)` | Pause once, then auto-undebug |
| `traceback()` | Find where an error occurred |
| `options(error=recover)` | Walk the entire call stack after error |
| `tryCatch()` | Catch errors and handle gracefully |
| `stopifnot()` | Assert assumptions at function entry |
| `stop()` | Throw a clear, descriptive error |
| `warning()` | Warn but continue |

---

## Exercises

**1. Fix the buggy program**

Introduce all three bugs into `analyze.R`:
1. A CSV where one numeric column has a few string values
2. A dataset where one group has only 1 row
3. A filter applied to a character column that looks numeric

For each bug:
- Observe what happens (error message? silent wrong answer?)
- Use `browser()` or `options(error = recover)` to find the root cause
- Fix it

**2. options(error = recover)**

Write a chain: `a()` calls `b()` which calls `c()`. Make `c()` crash when its argument is negative. Set `options(error = recover)`. Call `a(-1)`. Explore frames 1, 2, and 3. What's in scope at each level?

**3. tryCatch batch processor**

Write `safe_process(files)` that reads a list of CSV files, processes each, and logs errors without stopping. Returns a list with `success` (list of results) and `errors` (list of filenames + error messages).

**4. Defensive analyze.R**

Add these checks to `analyze.R`:
- If the CSV has zero rows after filtering, print a helpful message and exit cleanly (not with an error)
- If a requested stat column doesn't exist, print which columns are available
- If `--stat` contains an unknown statistic name, list the valid ones

**5. The full debugging session**

Run `analyze.R` on `iris.csv` (write it with `write.csv(iris, "iris.csv")`):

```
Rscript analyze.R iris.csv --group=Species --filter=Petal.Length>4 --stat=n,mean,sd
```

Does it work? If not, find the bug and fix it. (Hint: column names with dots may need quoting in the filter string. How would you handle that?)

---

*That's debugging in R: `browser()` to stop and look, `traceback()` to find where it crashed, `options(error = recover)` to explore the whole stack, and `tryCatch()` to handle errors gracefully.*
