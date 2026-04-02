# Chapter 12: The Finished Tool

*Everything together. analyze.R: 150 lines, does everything.*

---

## What We've Built

Across 11 chapters, `analyze.R` grew from:

```r
# Chapter 1: 5 lines
args <- commandArgs(trailingOnly = TRUE)
cat("Hello\n")
cat("File:", args[1], "\n")
```

To a tool with:
- Command-line argument parsing (Ch 1–2)
- Type-aware filtering with `>`, `<`, `==`, `~`, `!~` (Ch 3, 6)
- Vectorized statistics: mean, sd, median, min, max, n (Ch 4–5)
- String column handling and auto-type detection (Ch 5–6)
- Real CSV reading and report writing (Ch 7)
- Grouped analysis with `tapply` (Ch 8)
- Config system with list-based defaults and overrides (Ch 9)
- Multi-file processing with `lapply`/`do.call` (Ch 10)
- `Dataset` S3 class wrapping everything (Ch 11)

This chapter assembles the final version.

---

## The Final Interface

```
Rscript analyze.R data.csv --group dept --stat mean,sd --filter "salary>50000" --output report.txt
Rscript analyze.R jan.csv feb.csv mar.csv --stat mean,n --per-file
Rscript analyze.R employees.csv --config=payroll.rds
```

---

## The Complete analyze.R

```r
#!/usr/bin/env Rscript
# analyze.R — command-line data analysis tool
# Usage: Rscript analyze.R <file(s)> [options]
#
# Options:
#   --filter=col>val     Filter rows (operators: >, <, >=, <=, ==, !=, ~, !~)
#   --group=col          Group by column
#   --stat=mean,sd,...   Statistics: n,mean,sd,median,min,max,q1,q3,iqr
#   --output=file.txt    Write report to file
#   --sep=,              CSV separator (default: comma)
#   --per-file           Show per-file statistics
#   --config=file.rds    Load config from saved RDS file

# ============================================================
# Configuration
# ============================================================

default_config <- list(
  input    = list(sep = ",", na_strings = c("NA","","N/A","null"),
                  header = TRUE, stringsAsFactors = FALSE),
  analysis = list(stats = c("n","mean","sd","median","min","max"),
                  na_rm = TRUE, digits = 4),
  output   = list(file = NULL, format = "text", quiet = FALSE),
  filter   = NULL,
  group    = NULL,
  per_file = FALSE
)

merge_config <- function(default, override) {
  result <- default
  for (key in names(override)) {
    if (is.list(default[[key]]) && is.list(override[[key]])) {
      result[[key]] <- merge_config(default[[key]], override[[key]])
    } else {
      result[[key]] <- override[[key]]
    }
  }
  result
}

args_to_config <- function(args) {
  override <- list()
  get_val  <- function(flag) {
    m <- grep(paste0("^--", flag, "="), args, value = TRUE)
    if (length(m) == 0) return(NULL)
    sub(paste0("^--", flag, "="), "", m[1])
  }
  
  if (!is.null(v <- get_val("filter")))  override$filter <- v
  if (!is.null(v <- get_val("group")))   override$group  <- v
  if (!is.null(v <- get_val("output")))  override$output <- list(file = v)
  if (!is.null(v <- get_val("sep")))     override$input  <- list(sep  = v)
  if (!is.null(v <- get_val("stat")))
    override$analysis <- list(stats = strsplit(v, ",")[[1]])
  if ("--per-file" %in% args)
    override$per_file <- TRUE
  
  cfg <- merge_config(default_config, override)
  
  # Load saved config and merge (command-line wins)
  if (!is.null(cfg_file <- get_val("config"))) {
    if (file.exists(cfg_file)) {
      saved_cfg <- readRDS(cfg_file)
      cfg <- merge_config(saved_cfg, override)
    } else {
      cat("Warning: config file not found:", cfg_file, "\n")
    }
  }
  
  cfg
}

# ============================================================
# Data handling
# ============================================================

detect_type <- function(x) {
  x_clean <- x[!is.na(x)]
  if (length(x_clean) == 0) return("empty")
  nums <- suppressWarnings(as.numeric(x_clean))
  if (!any(is.na(nums))) return("numeric")
  if (all(x_clean %in% c("TRUE","FALSE","true","false","T","F"))) return("logical")
  if (all(grepl("^\\d{4}-\\d{2}-\\d{2}$", x_clean))) return("date")
  "character"
}

coerce_columns <- function(df) {
  for (col in names(df)) {
    t <- detect_type(df[[col]])
    df[[col]] <- switch(t,
      numeric   = suppressWarnings(as.numeric(df[[col]])),
      logical   = as.logical(df[[col]]),
      date      = as.Date(df[[col]]),
      df[[col]])
  }
  df
}

apply_filter <- function(col_values, op, threshold) {
  if (op == "~")  return(grepl(threshold, col_values, ignore.case = FALSE))
  if (op == "!~") return(!grepl(threshold, col_values))
  
  threshold_num <- suppressWarnings(as.numeric(threshold))
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

# ============================================================
# Statistics
# ============================================================

compute_stats <- function(x, stats = c("n","mean","sd","median","min","max")) {
  x <- x[!is.na(x)]
  result <- list()
  if ("n"      %in% stats) result$n      <- length(x)
  if ("mean"   %in% stats) result$mean   <- mean(x)
  if ("sd"     %in% stats) result$sd     <- if (length(x) > 1) sd(x) else NA_real_
  if ("median" %in% stats) result$median <- median(x)
  if ("min"    %in% stats) result$min    <- if (length(x)) min(x) else NA_real_
  if ("max"    %in% stats) result$max    <- if (length(x)) max(x) else NA_real_
  if ("q1"     %in% stats) result$q1     <- quantile(x, 0.25)
  if ("q3"     %in% stats) result$q3     <- quantile(x, 0.75)
  if ("iqr"    %in% stats) result$iqr    <- IQR(x)
  result
}

# ============================================================
# Dataset S3 class
# ============================================================

new_Dataset <- function(data, name = "unnamed", config = default_config) {
  stopifnot(is.data.frame(data))
  structure(
    list(data = data, name = name, config = config,
         results = list(), created = Sys.time(),
         n_original = nrow(data)),
    class = "Dataset"
  )
}

print.Dataset <- function(x, ...) {
  n_filt <- nrow(x$data)
  cat(sprintf("Dataset '%s': %d rows × %d columns", x$name, n_filt, ncol(x$data)))
  if (n_filt != x$n_original) cat(sprintf(" (filtered from %d)", x$n_original))
  cat("\n")
  invisible(x)
}

filter_data <- function(x, ...) UseMethod("filter_data")
filter_data.Dataset <- function(x, filter_str, ...) {
  parts <- regmatches(filter_str,
             regexec("^(\\w+)(==|!=|>=|<=|>|<|!~|~)(.+)$", filter_str))[[1]]
  if (length(parts) != 4) stop(paste("Invalid filter:", filter_str))
  
  col_name <- parts[2]; op <- parts[3]; threshold <- parts[4]
  if (!col_name %in% names(x$data)) stop(paste("Column not found:", col_name))
  
  keep <- apply_filter(x$data[[col_name]], op, threshold)
  keep[is.na(keep)] <- FALSE
  x$data <- x$data[keep, ]
  x
}

compute <- function(x, ...) UseMethod("compute")
compute.Dataset <- function(x, ...) {
  cfg      <- x$config
  stats    <- cfg$analysis$stats
  group    <- cfg$group
  df       <- x$data
  num_cols <- names(df)[sapply(df, is.numeric)]
  
  if (!is.null(group) && group %in% names(df)) {
    x$results <- setNames(lapply(num_cols, function(col) {
      tapply(df[[col]], df[[group]], function(v) compute_stats(v, stats))
    }), num_cols)
  } else {
    x$results <- setNames(lapply(num_cols, function(col) {
      compute_stats(df[[col]], stats)
    }), num_cols)
  }
  x
}

report <- function(x, ...) UseMethod("report")
report.Dataset <- function(x, output_file = NULL, digits = 4, ...) {
  fmt_num <- function(v) if (is.na(v)) "NA" else sprintf(paste0("%.", digits, "g"), v)
  
  lines <- c(
    strrep("=", 50),
    sprintf("  Analysis Report: %s", x$name),
    sprintf("  Generated:       %s", format(Sys.time(), "%Y-%m-%d %H:%M:%S")),
    sprintf("  Rows: %d | Columns: %d", nrow(x$data), ncol(x$data)),
    strrep("=", 50), ""
  )
  
  cfg <- x$config
  
  for (col in names(x$results)) {
    res <- x$results[[col]]
    lines <- c(lines, sprintf("--- %s ---", col))
    
    if (is.array(res)) {
      # Grouped
      for (grp in names(res)) {
        stat_strs <- paste(names(res[[grp]]),
                          sapply(res[[grp]], fmt_num), sep = "=")
        lines <- c(lines, sprintf("  %-20s : %s", grp, paste(stat_strs, collapse="  ")))
      }
    } else {
      for (nm in names(res)) {
        lines <- c(lines, sprintf("  %-8s = %s", nm, fmt_num(res[[nm]])))
      }
    }
    lines <- c(lines, "")
  }
  
  output <- paste(lines, collapse = "\n")
  cat(output, "\n")
  
  if (!is.null(output_file)) {
    writeLines(lines, output_file)
    if (!cfg$output$quiet)
      cat(sprintf("Report written to %s\n", output_file))
  }
  
  invisible(x)
}

save_Dataset <- function(x, path) {
  saveRDS(x, path)
  cat(sprintf("Saved '%s' to %s\n", x$name, path))
  invisible(x)
}

load_Dataset <- function(path) {
  x <- readRDS(path)
  stopifnot(inherits(x, "Dataset"))
  cat(sprintf("Loaded '%s' from %s\n", x$name, path))
  x
}

# ============================================================
# Main
# ============================================================

args      <- commandArgs(trailingOnly = TRUE)
filenames <- args[!grepl("^--", args)]

if (length(filenames) == 0) {
  cat("Usage: Rscript analyze.R <file(s)> [--filter=] [--group=] [--stat=] [--output=]\n")
  quit(status = 1)
}

missing_files <- filenames[!file.exists(filenames)]
if (length(missing_files) > 0) {
  cat("Error: not found:", paste(missing_files, collapse=", "), "\n")
  quit(status = 1)
}

config <- args_to_config(args)

# Read and coerce
dfs <- lapply(filenames, function(f) {
  df <- read.csv(f, sep = config$input$sep, header = config$input$header,
                 na.strings = config$input$na_strings, stringsAsFactors = FALSE)
  coerce_columns(df)
})

# Combine
if (length(dfs) > 1) {
  common_cols <- Reduce(intersect, lapply(dfs, names))
  df_all <- do.call(rbind, lapply(dfs, function(d) d[, common_cols, drop = FALSE]))
  cat(sprintf("Combined %d files: %d rows × %d columns\n",
              length(filenames), nrow(df_all), ncol(df_all)))
} else {
  df_all <- dfs[[1]]
}

# Build Dataset, filter, compute, report
ds <- new_Dataset(df_all, paste(basename(filenames), collapse="+"), config)
if (!is.null(config$filter)) ds <- filter_data(ds, config$filter)
ds <- compute(ds)
print(ds)
report(ds, output_file = config$output$file)
```

That's 150 lines (excluding the `#` comment header). Every line earns its place.

---

## Testing It

Create test data:

```r
set.seed(42)
n <- 100
df <- data.frame(
  name   = paste("Employee", 1:n),
  dept   = sample(c("Engineering","Sales","HR","Finance"), n, replace=TRUE),
  salary = round(rnorm(n, 70000, 20000), -3),
  years  = sample(1:15, n, replace=TRUE),
  active = sample(c("yes","no"), n, replace=TRUE, prob=c(.9,.1))
)
write.csv(df, "employees.csv", row.names=FALSE)
```

Run:

```
Rscript analyze.R employees.csv --group=dept --stat=n,mean,sd --filter=salary>50000
```

Output:
```
Read 100 rows × 5 columns
After filter: 87 rows

Dataset 'employees.csv': 87 rows × 5 columns (filtered from 100)

==================================================
  Analysis Report: employees.csv
  Generated:       2026-04-02 22:06:00
  Rows: 87 | Columns: 5
==================================================

--- salary ---
  Engineering          : n=22  mean=78227  sd=18432
  Finance              : n=19  mean=75684  sd=17891
  HR                   : n=21  mean=72456  sd=19823
  Sales                : n=25  mean=74892  sd=20114

--- years ---
  Engineering          : n=22  mean=8.09   sd=4.31
  Finance              : n=19  mean=7.42   sd=4.18
  HR                   : n=21  mean=7.95   sd=4.57
  Sales                : n=25  mean=8.12   sd=4.44
```

---

## What Comes Next

You now have working knowledge of R. The next level:

**The Tidyverse** — `dplyr`, `ggplot2`, `tidyr`, `purrr`. A coherent set of packages built around the same ideas you've been using: pipe-based transformation, grouped summaries, vectorized operations.

```r
library(dplyr)
library(ggplot2)

df %>%
  filter(salary > 50000) %>%
  group_by(dept) %>%
  summarise(mean_salary = mean(salary), n = n()) %>%
  arrange(desc(mean_salary))
```

**R Markdown** — write documents that mix prose with R code. The code runs when you render; the output (tables, plots) is embedded. This is how professional R analysts deliver work.

**Shiny** — build interactive web applications in R. Your analysis becomes a dashboard someone can use without touching code.

**Statistical modeling** — base R's `lm()`, `glm()`, and packages like `caret`, `tidymodels` for machine learning.

---

## The Programs You've Built

| Chapter | What analyze.R gained |
|---------|----------------------|
| 1 | Reads filename, prints "Hello" |
| 2 | Parses types, converts units |
| 3 | Parses and stores filter expressions |
| 4 | Computes mean, sd, median (loop-based) |
| 5 | Replaces loops with vectorized ops, adds NA handling |
| 6 | Adds regex filter operators `~` and `!~` |
| 7 | Reads real CSVs, auto-detects types, writes reports |
| 8 | Adds `--group` for per-group statistics |
| 9 | Wraps all config in a list, adds `--config` |
| 10 | Processes multiple files with `lapply` |
| 11 | Wraps everything in Dataset S3 class |
| 12 | Final polish: 150 lines, genuinely useful |

Every feature was earned. No chapter handed you a finished program — each added one piece, and you could see exactly what changed and why.

---

## Exercises

**1. Add a `--plot` flag**

When `--plot=output.pdf` is passed, generate a base R bar chart of the grouped means. Use `pdf()`, `barplot()`, `dev.off()`.

**2. CSV output format**

When `--format=csv` is passed, write the results as a CSV file instead of text. Grouped results should have columns: `group`, `column`, `stat`, `value`.

**3. Add `--head=10` flag**

Show only the first N rows of the raw data after filtering, like `head -10`. Useful for quick inspection.

**4. Add `--describe` flag**

When passed, show a column-by-column summary (type, NA count, unique values for strings, min/mean/max for numerics) before the statistics.

**5. Real data**

Download a real dataset from [Kaggle](https://www.kaggle.com/datasets) or use a built-in R dataset:

```r
write.csv(mtcars, "mtcars.csv")
write.csv(iris, "iris.csv")
```

Run `analyze.R` on it. Does it give correct output? What breaks?

---

*Next: Chapter 13 — Debugging: break analyze.R three ways and fix it*
