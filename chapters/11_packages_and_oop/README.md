# Chapter 11: Packages and OOP

*Installing and using packages. S3 classes. Wrapping analyze.R into a Dataset.*

---

## Where We Left Off

`analyze.R` now has:
- Multi-file input (`lapply` over files)
- Config system (nested list)
- Filtering, grouping, stats

The main weakness: results are scattered. After running, you have `df`, `stats_results`, `config` — three separate objects you have to keep track of. When you call a follow-up function, you pass all three.

Better: one object that holds data, config, and results together. That's what an S3 class gives us.

```r
ds <- Dataset("employees.csv", config)
ds <- filter(ds, salary > 60000)
ds <- compute(ds)
print(ds)
```

---

## Packages

R's strength is its ecosystem. CRAN has 20,000+ packages.

```r
# Install once
install.packages("ggplot2")
install.packages(c("dplyr", "tidyr", "readr"))

# Load every session
library(ggplot2)
library(dplyr)

# Use without loading the whole package:
dplyr::filter(df, age > 30)

# Check if installed:
"ggplot2" %in% installed.packages()[, "Package"]
```

---

## Essential Packages

**Data manipulation:**
- `dplyr` — `filter`, `select`, `mutate`, `group_by`, `summarise`
- `tidyr` — `pivot_longer`, `pivot_wider`
- `data.table` — fast for large data

**Reading data:**
- `readr` — fast CSV reading
- `readxl` — Excel files
- `jsonlite` — JSON

**Visualization:**
- `ggplot2` — the grammar of graphics

**Dates/Times:**
- `lubridate` — painless date handling

For now we stay in base R — the concepts apply to all packages.

---

## S3: R's Main OOP System

R has several OOP systems. S3 is the most common — used throughout base R.

S3 is based on **generic functions** and **methods**. A generic like `print()` looks at the class of its argument and dispatches to the right method.

**Create an S3 class:**

```r
new_dog <- function(name, breed, age) {
  structure(
    list(name = name, breed = breed, age = age),
    class = "Dog"
  )
}
```

**Methods** — define `generic.ClassName`:

```r
print.Dog <- function(x, ...) {
  cat(sprintf("Dog: %s (%s), age %d\n", x$name, x$breed, x$age))
  invisible(x)
}

summary.Dog <- function(object, ...) {
  cat(sprintf("Name: %s\nBreed: %s\nAge: %d years\n",
              object$name, object$breed, object$age))
}

# Custom generic
speak <- function(x, ...) UseMethod("speak")
speak.Dog <- function(x, ...) cat(x$name, "says: Woof!\n")
speak.default <- function(x, ...) cat("...\n")
```

**Use it:**

```r
rex <- new_dog("Rex", "German Shepherd", 5)
print(rex)        # calls print.Dog
speak(rex)        # calls speak.Dog
class(rex)        # "Dog"
inherits(rex, "Dog")  # TRUE
```

---

## S3 Inheritance

```r
new_guide_dog <- function(name, breed, age, owner) {
  obj <- new_dog(name, breed, age)
  obj$owner <- owner
  class(obj) <- c("GuideDog", "Dog")  # inherits from Dog
  obj
}

print.GuideDog <- function(x, ...) {
  NextMethod()   # calls print.Dog
  cat(sprintf("  Guide dog for: %s\n", x$owner))
  invisible(x)
}

buddy <- new_guide_dog("Buddy", "Golden Retriever", 4, "John")
print(buddy)   # print.GuideDog → NextMethod() → print.Dog
```

---

## Building the Dataset Class

Now apply this to `analyze.R`. The `Dataset` class holds everything together:

```r
# Constructor
new_Dataset <- function(data, name = "unnamed", config = default_config) {
  if (!is.data.frame(data)) stop("data must be a data.frame")
  
  structure(
    list(
      data        = data,
      name        = name,
      config      = config,
      results     = list(),    # stats go here after compute()
      created     = Sys.time(),
      filtered    = FALSE,
      n_original  = nrow(data)
    ),
    class = "Dataset"
  )
}
```

Now the methods:

```r
# print.Dataset: concise one-liner
print.Dataset <- function(x, ...) {
  cat(sprintf("Dataset '%s': %d rows × %d columns",
              x$name, nrow(x$data), ncol(x$data)))
  if (x$filtered)
    cat(sprintf(" (filtered from %d)", x$n_original))
  cat("\n")
  invisible(x)
}

# summary.Dataset: full breakdown
summary.Dataset <- function(object, ...) {
  cat(sprintf("=== Dataset: %s ===\n", object$name))
  cat(sprintf("Dimensions: %d × %d\n", nrow(object$data), ncol(object$data)))
  cat(sprintf("Created:    %s\n\n", format(object$created, "%Y-%m-%d %H:%M")))
  
  df <- object$data
  for (col in names(df)) {
    x <- df[[col]]
    if (is.numeric(x)) {
      cat(sprintf("  %-15s [numeric]   min=%.2f  mean=%.2f  max=%.2f  NAs=%d\n",
                  col, min(x,na.rm=T), mean(x,na.rm=T), max(x,na.rm=T), sum(is.na(x))))
    } else {
      cat(sprintf("  %-15s [character] %d unique  NAs=%d\n",
                  col, length(unique(x[!is.na(x)])), sum(is.na(x))))
    }
  }
  invisible(object)
}
```

Generic functions for Dataset operations:

```r
# filter generic
filter_data <- function(x, ...) UseMethod("filter_data")
filter_data.Dataset <- function(x, filter_str, ...) {
  parts <- regmatches(filter_str,
             regexec("^(\\w+)(==|!=|>=|<=|>|<|!~|~)(.+)$", filter_str))[[1]]
  if (length(parts) != 4) stop(paste("Invalid filter:", filter_str))
  
  col_name  <- parts[2]
  op        <- parts[3]
  threshold <- parts[4]
  
  if (!col_name %in% names(x$data)) stop(paste("Column not found:", col_name))
  
  keep <- apply_filter(x$data[[col_name]], op, threshold)
  keep[is.na(keep)] <- FALSE
  
  x$data     <- x$data[keep, ]
  x$filtered <- TRUE
  x$name     <- paste0(x$name, " [filtered]")
  x
}

# compute generic
compute <- function(x, ...) UseMethod("compute")
compute.Dataset <- function(x, ...) {
  cfg     <- x$config
  stats   <- cfg$analysis$stats
  group   <- cfg$group
  na.rm   <- cfg$analysis$na_rm
  
  df      <- x$data
  num_cols <- names(df)[sapply(df, is.numeric)]
  
  if (!is.null(group) && group %in% names(df)) {
    x$results <- lapply(num_cols, function(col) {
      tapply(df[[col]], df[[group]], function(v) {
        compute_stats(v[!is.na(v)], stats)
      })
    })
    names(x$results) <- num_cols
  } else {
    x$results <- lapply(num_cols, function(col) {
      v <- df[[col]]
      compute_stats(v[!is.na(v)], stats)
    })
    names(x$results) <- num_cols
  }
  
  x
}
```

Report generation as a method:

```r
report <- function(x, ...) UseMethod("report")
report.Dataset <- function(x, output_file = NULL, ...) {
  lines <- c(
    sprintf("=== Analysis Report: %s ===", x$name),
    sprintf("Generated: %s", format(Sys.time(), "%Y-%m-%d %H:%M:%S")),
    sprintf("Rows: %d | Columns: %d", nrow(x$data), ncol(x$data)),
    ""
  )
  
  for (col in names(x$results)) {
    res <- x$results[[col]]
    lines <- c(lines, sprintf("--- %s ---", col))
    
    if (is.list(res) && !is.null(names(res[[1]]))) {
      # Grouped results
      for (grp in names(res)) {
        stat_str <- paste(names(res[[grp]]),
                         round(unlist(res[[grp]]), 4), sep="=", collapse="  ")
        lines <- c(lines, sprintf("  %-20s : %s", grp, stat_str))
      }
    } else {
      for (nm in names(res)) {
        lines <- c(lines, sprintf("  %-8s = %.4f", nm, res[[nm]]))
      }
    }
    lines <- c(lines, "")
  }
  
  cat(paste(lines, collapse="\n"), "\n")
  
  if (!is.null(output_file)) {
    writeLines(lines, output_file)
    cat(sprintf("Report written to %s\n", output_file))
  }
  
  invisible(x)
}
```

---

## analyze.R: Chapter 11 Version

What changed: everything goes through the `Dataset` class. The main program is now just:

```r
# analyze.R — Chapter 11
# Usage: Rscript analyze.R <file.csv> [options]

# ... (source helper functions: apply_filter, compute_stats, default_config, etc.)

args     <- commandArgs(trailingOnly = TRUE)
filename <- args[!grepl("^--", args)][1]

if (is.na(filename) || !file.exists(filename)) {
  cat("Usage: Rscript analyze.R <file.csv> [--filter=] [--group=] [--stat=] [--output=]\n")
  quit(status = 1)
}

config <- args_to_config(args)
validate_config(config)

df  <- read.csv(filename, stringsAsFactors = FALSE,
                sep = config$input$sep, na.strings = config$input$na_strings)
ds  <- new_Dataset(df, basename(filename), config)

# Apply filter if specified
if (!is.null(config$filter)) {
  ds <- filter_data(ds, config$filter)
}

# Compute stats
ds <- compute(ds)

# Report
output_file <- config$output$file
report(ds, output_file = output_file)
```

Clean. The logic lives in the class methods. The main program is just orchestration.

---

## Exercises

**1. Extend the Dog class**

Add:
- A `feed(x)` generic that updates health
- A `compare(a, b)` method comparing two dogs by age
- A `format.Dog` method so `paste(rex)` produces a readable string

**2. S3 arithmetic**

Create a `Money` class with currency and amount:
```r
m1 <- new_money(100, "USD")
m2 <- new_money(50, "USD")
m1 + m2   # Money: 150 USD
```
Implement `+.Money` and `print.Money`. Add currency checking: adding USD to EUR should error.

**3. Write a package**

Create a minimal R package:
- `File → New Project → New Directory → R Package` in RStudio
- Add `compute_stats()` and `describe()` from this course
- Document with `#'` roxygen comments
- `Build → Install and Restart`

**4. The growing program (do this one)**

Add `load_Dataset` and `save_Dataset` methods. `save_Dataset(ds, "analysis.rds")` saves the entire object. `load_Dataset("analysis.rds")` restores it.

This is the final piece before Chapter 12: the tool can save its state, so you can run analysis once and re-use the results without re-reading and re-computing the data.

---

*Next: Chapter 12 — The finished tool: 150 lines, does everything*

---

## Solutions

### Exercise 1 — Extend the Dog class

```r
# Constructor (from chapter)
new_dog <- function(name, breed, age) {
  structure(list(name=name, breed=breed, age=age, health=100),
            class = "Dog")
}

# Existing methods
print.Dog <- function(x, ...) {
  cat(sprintf("Dog: %s (%s), age %d, health %d\n",
              x$name, x$breed, x$age, x$health))
  invisible(x)
}

speak <- function(x, ...) UseMethod("speak")
speak.Dog <- function(x, ...) cat(x$name, "says: Woof!\n")

# --- New: feed generic ---
feed <- function(x, ...) UseMethod("feed")
feed.Dog <- function(x, amount = 10, ...) {
  x$health <- min(100, x$health + amount)   # cap at 100
  cat(sprintf("%s was fed. Health: %d\n", x$name, x$health))
  invisible(x)
}

# --- New: compare two dogs by age ---
compare <- function(a, b, ...) UseMethod("compare")
compare.Dog <- function(a, b, ...) {
  if (a$age > b$age)      cat(sprintf("%s is older than %s\n",  a$name, b$name))
  else if (a$age < b$age) cat(sprintf("%s is younger than %s\n", a$name, b$name))
  else                    cat(sprintf("%s and %s are the same age\n", a$name, b$name))
  invisible(list(a=a, b=b))
}

# --- New: format.Dog so paste() works ---
format.Dog <- function(x, ...) {
  sprintf("%s the %s (age %d)", x$name, x$breed, x$age)
}

# Tests
rex   <- new_dog("Rex",   "German Shepherd", 5)
buddy <- new_dog("Buddy", "Golden Retriever", 3)

print(rex)
feed(rex, 5)
compare(rex, buddy)
paste("My dog is", format(rex))   # "My dog is Rex the German Shepherd (age 5)"
```

### Exercise 2 — S3 arithmetic (Money class)

```r
new_money <- function(amount, currency = "USD") {
  if (!is.numeric(amount)) stop("amount must be numeric")
  structure(list(amount=amount, currency=toupper(currency)), class="Money")
}

print.Money <- function(x, ...) {
  cat(sprintf("Money: %.2f %s\n", x$amount, x$currency))
  invisible(x)
}

`+.Money` <- function(a, b) {
  if (!inherits(b, "Money")) stop("Can only add Money to Money")
  if (a$currency != b$currency)
    stop(sprintf("Currency mismatch: %s != %s", a$currency, b$currency))
  new_money(a$amount + b$amount, a$currency)
}

`-.Money` <- function(a, b) {
  if (!inherits(b, "Money")) stop("Can only subtract Money from Money")
  if (a$currency != b$currency)
    stop(sprintf("Currency mismatch: %s != %s", a$currency, b$currency))
  new_money(a$amount - b$amount, a$currency)
}

`==.Money` <- function(a, b) {
  inherits(b, "Money") && a$currency == b$currency && a$amount == b$amount
}

# Tests
m1 <- new_money(100, "USD")
m2 <- new_money(50,  "USD")
m1 + m2     # Money: 150.00 USD
m1 - m2     # Money: 50.00 USD

# Currency mismatch should error:
m3 <- new_money(80, "EUR")
tryCatch(m1 + m3, error = function(e) cat("Error caught:", e$message, "\n"))
# Error caught: Currency mismatch: USD != EUR
```

### Exercise 3 — Write a package (outline)

Creating a minimal R package requires a few files. Here is the complete minimal structure:

```r
# Step 1 — create package skeleton (run in RStudio or R console)
# usethis::create_package("~/Projects/ranalytics")
# OR manually create the directory structure:

# ranalytics/
#   DESCRIPTION
#   NAMESPACE
#   R/
#     stats.R

# --- DESCRIPTION ---
# Package: ranalytics
# Title: Command-line data analysis helpers
# Version: 0.1.0
# Author: Your Name
# Description: compute_stats() and describe() from the R Programming course.
# License: MIT

# --- R/stats.R ---

#' Compute descriptive statistics for a numeric vector
#'
#' @param x numeric vector (NAs removed automatically)
#' @param stats character vector of statistics to compute:
#'   "n", "mean", "sd", "median", "min", "max", "q1", "q3", "iqr"
#' @return named list of computed statistics
#' @export
#' @examples
#' compute_stats(c(1,2,3,4,5), c("mean","sd"))
compute_stats <- function(x, stats = c("n","mean","sd","median","min","max")) {
  x <- x[!is.na(x)]
  result <- list()
  if ("n"      %in% stats) result$n      <- length(x)
  if ("mean"   %in% stats) result$mean   <- mean(x)
  if ("sd"     %in% stats) result$sd     <- if (length(x) > 1) sd(x) else NA_real_
  if ("median" %in% stats) result$median <- median(x)
  if ("min"    %in% stats) result$min    <- if (length(x)) min(x) else NA_real_
  if ("max"    %in% stats) result$max    <- if (length(x)) max(x) else NA_real_
  if ("q1"     %in% stats) result$q1     <- quantile(x, 0.25, names=FALSE)
  if ("q3"     %in% stats) result$q3     <- quantile(x, 0.75, names=FALSE)
  if ("iqr"    %in% stats) result$iqr    <- IQR(x)
  result
}

#' Print a descriptive summary of a numeric vector
#'
#' @param x numeric vector
#' @param name label shown in the output
#' @return invisible(x)
#' @export
describe <- function(x, name = "x") {
  s <- compute_stats(x, c("n","mean","sd","median","min","max","q1","q3"))
  cat(sprintf("\nSummary of %s (%d values):\n", name, s$n))
  cat(sprintf("  Min:    %8.4f\n", s$min))
  cat(sprintf("  Q1:     %8.4f\n", s$q1))
  cat(sprintf("  Median: %8.4f\n", s$median))
  cat(sprintf("  Mean:   %8.4f\n", s$mean))
  cat(sprintf("  Q3:     %8.4f\n", s$q3))
  cat(sprintf("  Max:    %8.4f\n", s$max))
  cat(sprintf("  SD:     %8.4f\n", s$sd))
  invisible(x)
}

# Step 2 — install (from package root directory):
# install.packages(".", repos = NULL, type = "source")
# OR in RStudio: Build → Install and Restart

# Step 3 — use:
# library(ranalytics)
# compute_stats(c(10, 20, 30), c("mean","sd"))
```

### Exercise 4 — The growing program (`save_Dataset` / `load_Dataset`)

```r
# --- Addition to analyze.R (Chapter 11) ---

# These methods serialize the entire Dataset object (data + config + results).

save_Dataset <- function(x, path) {
  if (!inherits(x, "Dataset")) stop("x must be a Dataset object")
  saveRDS(x, path)
  cat(sprintf("Dataset '%s' saved to %s\n", x$name, path))
  invisible(x)
}

load_Dataset <- function(path) {
  if (!file.exists(path)) stop(paste("File not found:", path))
  x <- readRDS(path)
  if (!inherits(x, "Dataset")) stop("File does not contain a Dataset object")
  cat(sprintf("Loaded Dataset '%s' from %s\n", x$name, path))
  x
}

# --- Usage example ---
# (after running the full analyze.R pipeline to produce 'ds')

# Save:
# ds <- compute(ds)
# save_Dataset(ds, "analysis.rds")
# # Dataset 'employees.csv' saved to analysis.rds

# Later — restore without re-reading or re-computing:
# ds_restored <- load_Dataset("analysis.rds")
# report(ds_restored)

# Demo (self-contained):
df_demo <- data.frame(
  name   = c("Alice","Bob","Carol"),
  salary = c(72000, 48000, 95000),
  stringsAsFactors = FALSE
)
ds_demo <- new_Dataset(df_demo, "demo")
ds_demo <- compute(ds_demo)
save_Dataset(ds_demo, "/tmp/demo_analysis.rds")

ds_back <- load_Dataset("/tmp/demo_analysis.rds")
print(ds_back)     # Dataset 'demo': 3 rows × 2 columns
report(ds_back)    # shows the same report as before
```
