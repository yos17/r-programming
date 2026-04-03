# Chapter 7: I/O and Files

*Reading and writing data. CSV, text files, reports.*

---

## Where We Left Off

`analyze.R` has:
- Command-line argument parsing
- Filter parsing and application (`>`, `<`, `==`, `~`, etc.)
- Statistics functions (`compute_stats`, `describe`)

But it still runs on fake hardcoded data. This chapter replaces the fake data with real CSV reading. When this chapter is done, `analyze.R` will actually work on real files.

---

## The Working Directory

R reads and writes files relative to the **working directory**.

```r
getwd()                                # show current directory
setwd("~/Projects/r-programming")      # change it
```

In RStudio: the **Files** panel shows your working directory. Set it via `Session → Set Working Directory`.

For command-line scripts (`Rscript`), the working directory is wherever you run the command. Keep that in mind.

---

## Writing Text Files

```r
# Write character vector (one element per line)
lines <- c("Line 1", "Line 2", "Line 3")
writeLines(lines, "output.txt")

# Append to file
cat("Another line\n", file = "output.txt", append = TRUE)

# Write formatted report
sink("report.txt")           # redirect cat() output to file
cat("Title\n")
cat("Data: 42\n")
sink()                       # restore output to console
```

`sink()` is useful for reports: write the same `cat()` calls, just redirect them to a file.

---

## Reading Text Files

```r
lines <- readLines("output.txt")
lines          # c("Line 1", "Line 2", "Line 3")
length(lines)  # number of lines

# Read from a URL:
lines <- readLines("https://example.com/data.txt")
```

---

## Writing CSV

```r
df <- data.frame(
  name  = c("Alice", "Bob", "Carol"),
  score = c(92, 78, 95)
)

write.csv(df, "scores.csv", row.names = FALSE)
```

`row.names = FALSE` suppresses the row number column. Almost always what you want.

---

## Reading CSV

```r
df <- read.csv("scores.csv")
head(df)     # first 6 rows
str(df)      # structure: types, dimensions
nrow(df)     # number of rows
ncol(df)     # number of columns
names(df)    # column names
```

`read.csv` options:

```r
read.csv("data.csv",
  header           = TRUE,   # first row = column names
  sep              = ",",    # separator
  na.strings       = "NA",   # what to treat as NA
  skip             = 2,      # skip first 2 lines
  nrows            = 100,    # read only 100 rows
  stringsAsFactors = FALSE   # keep strings as strings
)
```

For European-style CSV (semicolons, comma as decimal):
```r
read.csv2("data.csv")   # sep=";", dec=","
```

---

## Data Validation After Reading

Never trust raw CSV. Always validate:

```r
validate_csv <- function(df, required_cols = character(0)) {
  errors <- character(0)
  
  # Check required columns exist
  missing <- setdiff(required_cols, names(df))
  if (length(missing) > 0)
    errors <- c(errors, paste("Missing columns:", paste(missing, collapse=", ")))
  
  # Check for completely empty columns
  empty_cols <- names(df)[sapply(df, function(x) all(is.na(x)))]
  if (length(empty_cols) > 0)
    errors <- c(errors, paste("All-NA columns:", paste(empty_cols, collapse=", ")))
  
  # Report NA counts
  na_counts <- sapply(df, function(x) sum(is.na(x)))
  if (any(na_counts > 0)) {
    cat("NA counts:\n")
    for (col in names(na_counts[na_counts > 0])) {
      cat(sprintf("  %s: %d NAs (%.1f%%)\n", col, na_counts[col],
                  100 * na_counts[col] / nrow(df)))
    }
  }
  
  if (length(errors) > 0) {
    for (e in errors) cat("Error:", e, "\n")
    return(FALSE)
  }
  
  TRUE
}
```

---

## Auto-Detecting Column Types

CSV files contain only strings. Before computing, we need to detect whether each column is numeric, logical, or text.

```r
detect_type <- function(x) {
  # Remove NAs for detection
  x_clean <- x[!is.na(x)]
  if (length(x_clean) == 0) return("empty")
  
  # Try numeric
  nums <- suppressWarnings(as.numeric(x_clean))
  if (!any(is.na(nums))) return("numeric")
  
  # Try logical
  if (all(x_clean %in% c("TRUE","FALSE","true","false","T","F")))
    return("logical")
  
  # Try date (YYYY-MM-DD)
  if (all(grepl("^\\d{4}-\\d{2}-\\d{2}$", x_clean)))
    return("date")
  
  "character"
}

coerce_column <- function(x, type) {
  switch(type,
    numeric   = suppressWarnings(as.numeric(x)),
    logical   = as.logical(x),
    date      = as.Date(x),
    character = x,
    x   # fallback: leave as-is
  )
}
```

---

## Writing Reports

```r
write_report <- function(df, output_file, stats_list) {
  sink(output_file)
  on.exit(sink())   # always restore, even if an error occurs
  
  cat("=== Analysis Report ===\n")
  cat(sprintf("Generated: %s\n", format(Sys.time(), "%Y-%m-%d %H:%M:%S")))
  cat(sprintf("Rows: %d | Columns: %d\n\n", nrow(df), ncol(df)))
  
  for (col_name in names(stats_list)) {
    stats <- stats_list[[col_name]]
    cat(sprintf("--- %s ---\n", col_name))
    for (stat_name in names(stats)) {
      cat(sprintf("  %-8s = %s\n", stat_name,
                  if (is.numeric(stats[[stat_name]])) 
                    sprintf("%.4f", stats[[stat_name]])
                  else
                    as.character(stats[[stat_name]])))
    }
    cat("\n")
  }
}
```

---

## File and Directory Operations

```r
file.exists("data.csv")          # check if file exists
file.copy("old.csv", "new.csv")  # copy
file.rename("old.csv", "new.csv")# rename/move
file.remove("temp.csv")          # delete

dir.create("output")             # create directory
list.files(".")                  # list files
list.files(".", pattern = "\\.csv$")   # CSV files only
```

---

## analyze.R: Chapter 7 Version

This is where it becomes real. We replace the hardcoded data with actual CSV reading.

```r
# analyze.R — Chapter 7
# Usage: Rscript analyze.R <file.csv> [--filter 'col>value'] [--stat mean,sd] [--output report.txt]

args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv> [--filter 'col>value'] [--stat stats] [--output file]\n")
  quit(status = 1)
}

filename <- args[1]
if (!file.exists(filename)) {
  cat("Error: file not found:", filename, "\n")
  quit(status = 1)
}

# --- Parse arguments ---
get_arg <- function(args, flag) {
  m <- grep(paste0("^", flag, "="), args, value = TRUE)
  if (length(m) == 0) return(NULL)
  sub(paste0("^", flag, "="), "", m[1])
}

filter_str  <- get_arg(args, "--filter")
stat_str    <- get_arg(args, "--stat")
output_file <- get_arg(args, "--output")

requested_stats <- if (!is.null(stat_str)) strsplit(stat_str, ",")[[1]]
                   else c("mean", "sd", "median", "min", "max", "n")

# --- Read and detect types ---
df_raw <- read.csv(filename, stringsAsFactors = FALSE)
cat(sprintf("Read %d rows × %d columns from %s\n", nrow(df_raw), ncol(df_raw), filename))

# Detect and coerce column types
df <- df_raw
col_types <- sapply(df_raw, detect_type)
for (col in names(col_types)) {
  df[[col]] <- coerce_column(df_raw[[col]], col_types[[col]])
}

# --- Apply filter ---
if (!is.null(filter_str)) {
  parts <- regmatches(filter_str,
             regexec("^(\\w+)(==|!=|>=|<=|>|<|!~|~)(.+)$", filter_str))[[1]]
  if (length(parts) == 4) {
    col_name  <- parts[2]
    op        <- parts[3]
    threshold <- parts[4]
    if (col_name %in% names(df)) {
      keep <- apply_filter(df[[col_name]], op, threshold)
      keep[is.na(keep)] <- FALSE
      df <- df[keep, ]
      cat(sprintf("After filter '%s': %d rows\n", filter_str, nrow(df)))
    } else {
      cat(sprintf("Warning: column '%s' not found\n", col_name))
    }
  }
}

# --- Compute stats ---
stats_results <- list()
for (col in names(df)) {
  x <- df[[col]]
  if (is.numeric(x)) {
    x_clean <- x[!is.na(x)]
    stats_results[[col]] <- compute_stats(x_clean, requested_stats)
  }
}

# --- Output ---
report_lines <- c(
  sprintf("=== Analysis Report ==="),
  sprintf("File: %s", filename),
  sprintf("Rows: %d | Columns: %d", nrow(df), ncol(df)),
  ""
)
for (col in names(stats_results)) {
  report_lines <- c(report_lines, sprintf("--- %s ---", col))
  for (nm in names(stats_results[[col]])) {
    report_lines <- c(report_lines,
      sprintf("  %-8s = %.4f", nm, stats_results[[col]][[nm]]))
  }
  report_lines <- c(report_lines, "")
}

cat(paste(report_lines, collapse="\n"), "\n")

if (!is.null(output_file)) {
  writeLines(report_lines, output_file)
  cat(sprintf("Report written to %s\n", output_file))
}
```

Create a test file:

```r
write.csv(data.frame(
  name   = c("Alice","Bob","Carol","Dave","Eve"),
  salary = c(72000, 48000, 95000, 55000, 83000),
  dept   = c("Engineering","Sales","Engineering","HR","Sales"),
  years  = c(3, 7, 2, 10, 5)
), "employees.csv", row.names = FALSE)
```

Run it:

```
Rscript analyze.R employees.csv --filter=salary>60000 --stat=mean,sd --output=report.txt
```

Output:
```
Read 5 rows × 4 columns from employees.csv
After filter 'salary>60000': 3 rows

=== Analysis Report ===
File: employees.csv
Rows: 3 | Columns: 4

--- salary ---
  mean     = 83333.3333
  sd       = 11503.6237

--- years ---
  mean     = 3.3333
  sd       = 1.5275
```

---

## Exercises

**1. Sales data pipeline**

Create a CSV with columns: `date, product, quantity, price`. Generate 100 rows. Read it back. Compute:
- Total revenue by product
- Daily total revenue
- The top 3 revenue days

**2. Log file parser**

Generate a fake web server log file:
```
2026-03-15 10:23:45 GET /index.html 200 1234
2026-03-15 10:23:47 GET /about.html 200 876
```

Write a function that reads this log and reports:
- Request count by status code
- Most requested URLs
- Number of errors (4xx status codes)

**3. CSV round-trip**

Write a data frame to CSV, read it back, verify all values are identical. Where do round-trip errors occur? (Check floating point numbers.)

**4. Multi-file batch**

Write a script that:
- Generates 5 CSV files with random monthly sales data
- Reads all of them with `list.files()` + loop
- Combines with `rbind()`
- Writes the combined file

**5. The growing program (do this one)**

Add `--output` argument support to `analyze.R` so the report goes to a file. Use `sink()` with `on.exit(sink())`. Test:

```
Rscript analyze.R employees.csv --output=report.txt
cat report.txt
```

In Chapter 8, we add `--group` to group rows by a categorical column and compute per-group statistics. The `write_report()` function will need to handle grouped output sections.

---

*Next: Chapter 8 — Data Frames: group and aggregate*

---

## Solutions

### Exercise 1 — Sales data pipeline

```r
# Generate 100 rows of sales data
set.seed(42)
n <- 100
sales <- data.frame(
  date     = seq(as.Date("2026-01-01"), by = "day", length.out = n)[sample(n)],
  product  = sample(c("Widget","Gadget","Doohickey"), n, replace = TRUE),
  quantity = sample(1:20, n, replace = TRUE),
  price    = round(runif(n, 5, 100), 2),
  stringsAsFactors = FALSE
)
sales$date    <- as.character(sales$date)   # store as string for CSV round-trip

write.csv(sales, "sales.csv", row.names = FALSE)

# Read it back
df <- read.csv("sales.csv", stringsAsFactors = FALSE)
df$revenue <- df$quantity * df$price

# Total revenue by product
cat("Revenue by product:\n")
rev_by_product <- tapply(df$revenue, df$product, sum)
for (p in sort(names(rev_by_product))) {
  cat(sprintf("  %-12s : $%.2f\n", p, rev_by_product[p]))
}

# Daily total revenue
daily_rev <- tapply(df$revenue, df$date, sum)
daily_rev <- sort(daily_rev, decreasing = TRUE)

cat("\nTop 3 revenue days:\n")
for (i in 1:3) {
  cat(sprintf("  %s : $%.2f\n", names(daily_rev)[i], daily_rev[i]))
}
```

### Exercise 2 — Log file parser

```r
# Generate a fake web server log
set.seed(1)
n_lines <- 50
urls <- c("/index.html","/about.html","/contact.html","/api/data","/missing.html")
statuses <- sample(c(200,200,200,301,404,500), n_lines, replace = TRUE)
timestamps <- format(
  as.POSIXct("2026-03-15 10:00:00") + cumsum(sample(1:10, n_lines, replace = TRUE)),
  "%Y-%m-%d %H:%M:%S"
)
log_lines <- sprintf("%s GET %s %d %d",
  timestamps,
  sample(urls, n_lines, replace = TRUE),
  statuses,
  sample(500:5000, n_lines, replace = TRUE))

writeLines(log_lines, "server.log")

# Parse the log
parse_log <- function(log_file) {
  lines <- readLines(log_file)

  # Extract fields with regex
  # Format: DATETIME METHOD URL STATUS BYTES
  pat <- "^(\\S+ \\S+) (\\w+) (\\S+) (\\d+) (\\d+)$"
  parsed <- lapply(lines, function(l) {
    m <- regmatches(l, regexec(pat, l))[[1]]
    if (length(m) == 0) return(NULL)
    list(ts = m[2], method = m[3], url = m[4],
         status = as.integer(m[5]), bytes = as.integer(m[6]))
  })
  parsed <- Filter(Negate(is.null), parsed)

  status_codes <- sapply(parsed, `[[`, "status")
  urls_req     <- sapply(parsed, `[[`, "url")

  cat(sprintf("Total requests: %d\n\n", length(parsed)))

  cat("Requests by status code:\n")
  for (sc in sort(unique(status_codes))) {
    cat(sprintf("  %d : %d\n", sc, sum(status_codes == sc)))
  }

  cat("\nMost requested URLs:\n")
  freq <- sort(table(urls_req), decreasing = TRUE)
  for (i in seq_len(min(5, length(freq)))) {
    cat(sprintf("  %-20s : %d\n", names(freq)[i], freq[i]))
  }

  n_errors <- sum(status_codes >= 400)
  cat(sprintf("\nErrors (4xx/5xx): %d\n", n_errors))
}

parse_log("server.log")
```

### Exercise 3 — CSV round-trip

```r
# Create a data frame with various types
original <- data.frame(
  integer_col = 1L:5L,
  double_col  = c(1.0, 1.23456789, pi, exp(1), sqrt(2)),
  string_col  = c("hello", "world", "foo", "bar", "baz"),
  logical_col = c(TRUE, FALSE, TRUE, TRUE, FALSE),
  stringsAsFactors = FALSE
)

write.csv(original, "roundtrip.csv", row.names = FALSE)
restored <- read.csv("roundtrip.csv", stringsAsFactors = FALSE)

# Compare
cat("Identical integers? ", identical(original$integer_col, restored$integer_col), "\n")
# FALSE: original is integer class, restored is numeric (double)

cat("Equal doubles?      ", all.equal(original$double_col,  restored$double_col),  "\n")
# TRUE: doubles survive the round-trip accurately (R uses 15 sig figures by default)

cat("Identical strings?  ", identical(original$string_col,  restored$string_col),  "\n")
# TRUE

cat("Identical logicals? ", identical(original$logical_col, restored$logical_col), "\n")
# TRUE

# Finding differences:
# - Integers become doubles after read.csv (class changes, values identical)
# - Very long doubles may lose precision if write.csv uses fewer digits
# Fix: write.csv(..., digits = 22) to preserve full precision
```

### Exercise 4 — Multi-file batch

```r
# Generate 5 monthly CSV files
set.seed(99)
for (month in 1:5) {
  n <- sample(30:50, 1)
  df <- data.frame(
    month    = month,
    product  = sample(c("A","B","C"), n, replace = TRUE),
    sales    = round(runif(n, 100, 1000), 2),
    stringsAsFactors = FALSE
  )
  write.csv(df, sprintf("month_%02d.csv", month), row.names = FALSE)
}

# Read all monthly files and combine
files <- list.files(".", pattern = "^month_\\d+\\.csv$")
cat("Found files:", paste(files, collapse=", "), "\n")

dfs <- lapply(files, read.csv, stringsAsFactors = FALSE)
combined <- do.call(rbind, dfs)

cat(sprintf("Combined: %d rows\n", nrow(combined)))

write.csv(combined, "all_months.csv", row.names = FALSE)
cat("Written to all_months.csv\n")

# Quick summary
cat("\nTotal sales by product:\n")
print(tapply(combined$sales, combined$product, sum))
```

### Exercise 5 — The growing program (`--output` support)

Add `--output` argument to `analyze.R` to write the report to a file using `sink()`:

```r
# --- Addition to analyze.R (Chapter 7) ---

# Parse --output argument
output_file <- NULL
output_arg  <- grep("^--output=", args, value = TRUE)
if (length(output_arg) > 0) {
  output_file <- sub("^--output=", "", output_arg[1])
}

# Helper: run a block, writing output to file if requested
write_report <- function(output_file, report_fn) {
  if (!is.null(output_file)) {
    sink(output_file)
    on.exit(sink(), add = TRUE)   # restore even if error occurs
  }
  report_fn()
}

# Usage: wrap the entire output section
write_report(output_file, function() {
  cat("=== Analysis Report ===\n")
  cat(sprintf("File: %s\n", filename))
  cat(sprintf("Rows: %d | Columns: %d\n\n", nrow(df), ncol(df)))

  for (col in names(df)) {
    x <- df[[col]]
    if (!is.numeric(x)) next
    s <- compute_stats(x[!is.na(x)], req_stats)
    cat(sprintf("--- %s ---\n", col))
    for (nm in names(s)) cat(sprintf("  %-8s = %.4f\n", nm, s[[nm]]))
    cat("\n")
  }
})

if (!is.null(output_file)) {
  cat(sprintf("Report written to %s\n", output_file))
}

# Rscript analyze.R employees.csv --output=report.txt
# cat report.txt
# === Analysis Report ===
# File: employees.csv
# Rows: 5 | Columns: 4
# ...
```
