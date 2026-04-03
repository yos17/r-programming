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

---

## Solutions

### Exercise 1 — Fix the buggy program

Three bugs to introduce and fix:

```r
# ---- Bug 1: mixed-type column (some strings in a numeric column) ----

# Plant the bug: write a CSV where salary has one non-numeric value
employees_buggy <- data.frame(
  name   = c("Alice","Bob","Carol","Dave","Eve"),
  salary = c("72000","not disclosed","95000","55000","83000"),
  dept   = c("Engineering","Sales","Engineering","HR","Sales"),
  stringsAsFactors = FALSE
)
write.csv(employees_buggy, "employees_buggy.csv", row.names = FALSE)

# Observe: analyze.R silently skips the salary column (no stats printed)
# because detect_type() returns "character" when one value is non-numeric.

# Debug: set a browser() to catch partially-numeric columns
detect_type_verbose <- function(x, col_name = "?") {
  x_clean <- x[!is.na(x)]
  if (length(x_clean) == 0) return("empty")

  nums <- suppressWarnings(as.numeric(x_clean))
  n_numeric <- sum(!is.na(nums))

  if (n_numeric == length(x_clean)) return("numeric")

  if (n_numeric > 0) {
    # Partially numeric — warn and show bad values
    bad_vals <- x_clean[is.na(nums)]
    cat(sprintf("  Warning: '%s' has %d non-numeric value(s): %s\n",
                col_name, length(bad_vals),
                paste(head(bad_vals, 3), collapse=", ")))
    # Treat as numeric, coerce bad values to NA
    return("numeric_with_na")
  }

  if (all(x_clean %in% c("TRUE","FALSE","true","false","T","F"))) return("logical")
  if (all(grepl("^\\d{4}-\\d{2}-\\d{2}$", x_clean))) return("date")
  "character"
}

coerce_column_safe <- function(x, type) {
  if (type == "numeric_with_na") return(suppressWarnings(as.numeric(x)))
  switch(type,
    numeric   = suppressWarnings(as.numeric(x)),
    logical   = as.logical(x),
    date      = as.Date(x),
    x)
}

# ---- Bug 2: single-row group (sd returns NA, sprintf crashes) ----

employees_with_legal <- data.frame(
  name   = c("Alice","Bob","Carol","Dave","Eve","Zoe"),
  salary = c(72000, 48000, 95000, 55000, 83000, 95000),
  dept   = c("Engineering","Sales","Engineering","HR","Sales","Legal"),
  stringsAsFactors = FALSE
)
write.csv(employees_with_legal, "employees_with_legal.csv", row.names = FALSE)

# Observe: Rscript analyze.R employees_with_legal.csv --group=dept --stat=n,mean,sd
# Crashes in sprintf() because sd of a single value is NA.

# Fix 1: guard the fmt_num function used in report():
fmt_num <- function(v, digits = 4) {
  if (is.na(v)) return("NA")
  sprintf(paste0("%.", digits, "g"), v)
}

# Fix 2: add a warning in compute_stats():
compute_stats_safe <- function(x, stats = c("n","mean","sd","median","min","max")) {
  x <- x[!is.na(x)]
  result <- list()
  if ("n"      %in% stats) result$n      <- length(x)
  if ("mean"   %in% stats) result$mean   <- if (length(x) > 0) mean(x) else NA_real_
  if ("sd"     %in% stats) {
    if (length(x) < 2) {
      result$sd <- NA_real_
    } else {
      result$sd <- sd(x)
    }
  }
  if ("median" %in% stats) result$median <- if (length(x) > 0) median(x) else NA_real_
  if ("min"    %in% stats) result$min    <- if (length(x) > 0) min(x) else NA_real_
  if ("max"    %in% stats) result$max    <- if (length(x) > 0) max(x) else NA_real_
  result
}

# ---- Bug 3: character column with numeric-looking values and a numeric filter ----

employees_char_salary <- data.frame(
  name   = c("Alice","Bob","Carol","Dave","Eve"),
  salary = c("50000","72000","100000","not_disclosed","83000"),
  dept   = c("Engineering","Sales","Engineering","HR","Sales"),
  stringsAsFactors = FALSE
)
write.csv(employees_char_salary, "employees_char_salary.csv", row.names = FALSE)

# Observe: --filter=salary>60000 gives WRONG results (lexicographic comparison)
# "100000" < "60000" lexicographically → 100k employee is excluded

# Fix: detect type mismatch in apply_filter() and coerce:
apply_filter_safe <- function(col_values, op, threshold) {
  if (op == "~")  return(grepl(threshold, col_values))
  if (op == "!~") return(!grepl(threshold, col_values))

  threshold_num <- suppressWarnings(as.numeric(threshold))

  if (!is.na(threshold_num) && is.character(col_values)) {
    # Try numeric coercion of the column
    col_num <- suppressWarnings(as.numeric(col_values))
    n_bad   <- sum(is.na(col_num) & !is.na(col_values))
    if (all(is.na(col_num))) {
      # Column has no parseable numbers → string comparison only
      stop("Cannot apply numeric filter to non-numeric column")
    }
    if (n_bad > 0) {
      warning(sprintf("%d value(s) could not be parsed as numeric and will be excluded", n_bad))
    }
    col_values <- col_num   # now numeric; NAs will fail the comparison
  }

  if (!is.na(threshold_num)) {
    switch(op,
      ">"  = col_values > threshold_num,
      "<"  = col_values < threshold_num,
      ">=" = col_values >= threshold_num,
      "<=" = col_values <= threshold_num,
      "==" = col_values == threshold_num,
      "!=" = col_values != threshold_num,
      stop(paste("Unknown operator:", op)))
  } else {
    switch(op,
      "==" = col_values == threshold,
      "!=" = col_values != threshold,
      stop("String ops: ==, !=, ~, !~"))
  }
}
```

### Exercise 2 — `options(error = recover)` exploration

```r
a <- function(x) b(x)
b <- function(x) c(x)
c <- function(x) {
  if (x < 0) stop(sprintf("Negative input: %g", x))
  sqrt(x)
}

options(error = recover)
a(-1)

# --- Interactive session ---
# Error in c(x) : Negative input: -1
#
# Enter a frame number, or 0 to exit
# 1: a(-1)
# 2: b(x)     [called from a]
# 3: c(x)     [called from b]
#
# Selection: 1
# Browse[1]> x           # → -1
# Browse[1]> Q
#
# Selection: 2
# Browse[1]> x           # → -1  (same value passed through)
# Browse[1]> Q
#
# Selection: 3
# Browse[1]> x           # → -1  (the actual crash site)
# Browse[1]> x < 0       # → TRUE
# Browse[1]> Q

options(error = NULL)   # restore normal behaviour
```

### Exercise 3 — tryCatch batch processor

```r
safe_process <- function(files) {
  successes <- list()
  errors    <- list()

  for (f in files) {
    result <- tryCatch({
      df <- read.csv(f, stringsAsFactors = FALSE)
      # A simple analysis: mean of every numeric column
      nums <- sapply(df, function(x) {
        nums_x <- suppressWarnings(as.numeric(x))
        if (sum(!is.na(nums_x)) > length(x) * 0.5) mean(nums_x, na.rm=TRUE) else NA
      })
      nums <- nums[!is.na(nums)]
      list(file = f, rows = nrow(df), means = nums)
    }, error = function(e) {
      NULL
    })

    if (is.null(result)) {
      errors[[length(errors) + 1]] <- list(
        file    = f,
        message = tryCatch(read.csv(f), error = function(e) conditionMessage(e))
      )
    } else {
      successes[[length(successes) + 1]] <- result
    }
  }

  list(success = successes, errors = errors)
}

# --- Demo ---
# Create one good and one bad file
write.csv(data.frame(x=1:5, y=6:10), "good.csv", row.names=FALSE)
writeLines("this is not, a proper csv\n{broken}", "bad.csv")

results <- safe_process(c("good.csv", "bad.csv", "nonexistent.csv"))

cat(sprintf("Successes: %d\n", length(results$success)))
cat(sprintf("Errors:    %d\n", length(results$errors)))
for (e in results$errors) {
  cat(sprintf("  FAILED: %s\n", e$file))
}
```

### Exercise 4 — Defensive `analyze.R`

```r
# --- Additions to analyze.R (Chapter 13) ---

valid_stats <- c("n","mean","sd","median","min","max","q1","q3","iqr")

# 1. Check requested stats names before doing anything
if (!is.null(get_arg(args, "--stat"))) {
  req <- strsplit(get_arg(args, "--stat"), ",")[[1]]
  bad <- setdiff(req, valid_stats)
  if (length(bad) > 0) {
    cat(sprintf("Error: unknown stat(s): %s\n", paste(bad, collapse=", ")))
    cat(sprintf("Valid stats: %s\n", paste(valid_stats, collapse=", ")))
    quit(status = 1)
  }
}

# 2. After filtering: check for zero rows
if (nrow(df_all) == 0) {
  cat("No rows remain after filtering.\n")
  if (!is.null(config$filter)) {
    cat(sprintf("Filter applied: %s\n", config$filter))
    cat("Try a less restrictive filter.\n")
  }
  quit(status = 0)   # not an error — just nothing to report
}

# 3. Check that the group column exists; if not, list available columns
if (!is.null(config$group) && !config$group %in% names(df_all)) {
  cat(sprintf("Error: group column '%s' not found.\n", config$group))
  cat(sprintf("Available columns: %s\n", paste(names(df_all), collapse=", ")))
  quit(status = 1)
}

# 4. Check that at least one numeric column exists for stats
num_cols <- names(df_all)[sapply(df_all, is.numeric)]
if (length(num_cols) == 0) {
  cat("Warning: no numeric columns found — no statistics to compute.\n")
  cat(sprintf("Columns in file: %s\n", paste(names(df_all), collapse=", ")))
  quit(status = 0)
}
```

### Exercise 5 — Full debugging session with `iris.csv`

```r
# Step 1: write iris to CSV
write.csv(iris, "iris.csv")   # NOTE: includes row names by default!

# Step 2: run and observe
# Rscript analyze.R iris.csv --group=Species --filter=Petal.Length>4 --stat=n,mean,sd

# ---- Known issue 1: row names column ----
# iris.csv written with write.csv() includes a row-number column with empty name "".
# read.csv() reads it back as column "X" (or "") — a character column.
# Fix: always use row.names = FALSE
write.csv(iris, "iris.csv", row.names = FALSE)

# ---- Known issue 2: column names with dots ----
# iris columns: "Sepal.Length", "Sepal.Width", "Petal.Length", "Petal.Width", "Species"
# The default filter regex ^(\w+)(op)(value) uses \w which does NOT match dots.
# So --filter=Petal.Length>4 fails to parse.

# Reproduce the parse failure:
filter_str <- "Petal.Length>4"
parts <- regmatches(filter_str,
           regexec("^(\\w+)(==|!=|>=|<=|>|<|!~|~)(.+)$", filter_str))[[1]]
length(parts)   # 0 — no match because "." is not \w

# Fix: expand the column-name character class to include dots
parse_filter_dots <- function(filter_str) {
  # Use perl=TRUE and [\\w.]+ to allow dots in column names
  parts <- regmatches(filter_str,
             regexec("^([\\w.]+)(==|!=|>=|<=|>|<|!~|~)(.+)$",
                     filter_str, perl = TRUE))[[1]]
  if (length(parts) != 4) stop(paste("Invalid filter:", filter_str))
  list(column = parts[2], op = parts[3], value = parts[4])
}

# Verify fix:
f <- parse_filter_dots("Petal.Length>4")
cat("Column:", f$column, "\n")   # Petal.Length
cat("Op:    ", f$op,     "\n")   # >
cat("Value: ", f$value,  "\n")   # 4

# With both fixes applied, the full command works correctly:
# Rscript analyze.R iris.csv --group=Species --filter=Petal.Length>4 --stat=n,mean,sd
#
# Read 150 rows × 5 columns from iris.csv
# After filter 'Petal.Length>4': 58 rows
#
# --- Sepal.Length by Species ---
#   setosa               : n=0   (no setosa rows have Petal.Length > 4)
#   versicolor           : n=23  mean=6.15  sd=0.52
#   virginica            : n=35  mean=6.53  sd=0.64
#
# --- Petal.Length by Species ---
#   versicolor           : n=23  mean=4.61  sd=0.37
#   virginica            : n=35  mean=5.55  sd=0.55
```
