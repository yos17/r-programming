# Chapter 10: The Apply Family

*apply, lapply, sapply, tapply, Map, Reduce — R's functional tools.*

---

## Where We Left Off

`analyze.R` processes one CSV file with a config system. But data often comes in pieces:

```
Rscript analyze.R january.csv february.csv march.csv --stat=mean,sd
```

We could add a loop. Better: learn R's apply family, which replaces most loops with cleaner, more composable code. Then `analyze.R` becomes a function, and we call it via `lapply`.

---

## Why Apply?

In R, explicit `for` loops are:
1. Slower than vectorized operations for large data
2. More verbose
3. Less composable (harder to chain)

The apply family replaces most loops with cleaner, faster code.

---

## apply — Over Matrix Rows/Columns

```r
m <- matrix(1:12, nrow = 3)

apply(m, 1, sum)   # row sums    (MARGIN=1 = rows)
apply(m, 2, sum)   # column sums (MARGIN=2 = cols)
apply(m, 1, mean)  # row means
apply(m, 2, sd)    # column SDs
```

With a custom function:

```r
apply(m, 1, function(row) max(row) - min(row))   # range of each row
```

(For simple row/column ops, `rowSums`, `colSums`, `rowMeans`, `colMeans` are faster.)

---

## lapply / sapply — Over Lists and Vectors

```r
# lapply: always returns a list
lapply(1:5, function(x) x^2)
# list(1, 4, 9, 16, 25)

# sapply: simplifies to vector/matrix when possible
sapply(1:5, function(x) x^2)
# 1 4 9 16 25
```

`sapply` is `lapply` + simplification. Use `sapply` when you expect a clean vector result; use `lapply` when the result type is uncertain.

```r
# Practical: get file sizes for all CSVs in a directory
files <- list.files(".", pattern = "\\.csv$")
sizes <- sapply(files, file.size)
```

---

## vapply — Type-Safe Apply

`vapply` is like `sapply` but you declare the expected output type:

```r
vapply(1:5, function(x) x^2, FUN.VALUE = numeric(1))
# 1 4 9 16 25

# Crashes if result isn't the declared type — catches bugs early
vapply(c("a","b","c"), nchar, FUN.VALUE = integer(1))
```

Use `vapply` in production code over `sapply`.

---

## tapply — Apply by Group

Group a vector by a factor, then apply a function:

```r
scores <- c(85, 72, 95, 68, 90, 78)
grades <- c("B", "C", "A", "C", "A", "B")

tapply(scores, grades, mean)
# A    B    C
# 92.5 81.5 70.0
```

Equivalent to: for each unique value of `grades`, compute `mean(scores[grades == value])`.

---

## Map — Multiple Inputs

`Map` applies a function to multiple lists in parallel:

```r
names  <- c("Alice", "Bob", "Carol")
scores <- c(92, 78, 95)

Map(function(n, s) paste(n, "scored", s), names, scores)
# list("Alice scored 92", "Bob scored 78", "Carol scored 95")

# As a vector:
mapply(function(n, s) paste(n, "scored", s), names, scores)
```

---

## Reduce — Fold Over a List

`Reduce` applies a function cumulatively:

```r
Reduce("+", 1:5)          # ((((1+2)+3)+4)+5) = 15
Reduce("*", 1:5)          # 120 (5!)
Reduce(max, c(3,1,4,1,5)) # 5

# With accumulate=TRUE: intermediate results
Reduce("+", 1:5, accumulate = TRUE)
# 1 3 6 10 15  (running sum)
```

Combine multiple data frames:

```r
dfs <- list(df1, df2, df3)
Reduce(function(a, b) merge(a, b, by = "id"), dfs)
```

---

## Filter, Find, Position

```r
x <- list(1, "hello", TRUE, 3.14, "world", 42L)

Filter(is.character, x)    # list("hello", "world")
Filter(is.numeric,   x)    # list(1, 3.14)

Find(is.character, x)      # "hello" (first match)
Position(is.character, x)  # 2 (index of first match)
```

---

## do.call

`do.call` calls a function with a list as arguments. Essential for combining apply results:

```r
# Combine a list of data frames into one
combined <- do.call(rbind, list_of_dfs)

# Pass variable arguments
do.call(paste, list("a", "b", "c", sep = "-"))  # "a-b-c"
```

---

## Processing Multiple Files

Here's the pattern for batch processing:

```r
# The wrong way (loop)
results <- list()
for (f in files) {
  df <- read.csv(f)
  results[[f]] <- mean(df$salary, na.rm = TRUE)
}

# The R way (lapply)
results <- lapply(files, function(f) {
  df <- read.csv(f)
  mean(df$salary, na.rm = TRUE)
})
names(results) <- files
```

Same result. The second version is more concise and the function passed to `lapply` can be extracted, named, and tested independently.

---

## Batch Data Cleaner

```r
# batch_cleaner.R

make_messy_data <- function(n, noise = 0.1) {
  df <- data.frame(
    id    = 1:n,
    value = round(rnorm(n, 100, 20), 2),
    label = sample(c("A","B","C", NA), n, replace=TRUE, prob=c(.3,.3,.3,.1)),
    stringsAsFactors = FALSE
  )
  # Introduce outliers
  outlier_idx <- sample(n, max(1, round(n * noise)))
  df$value[outlier_idx] <- df$value[outlier_idx] * 5
  df
}

datasets <- lapply(1:5, function(i) make_messy_data(sample(50:100, 1)))
names(datasets) <- paste0("dataset_", 1:5)

clean_dataset <- function(df) {
  df <- df[!is.na(df$label), ]                        # drop NA labels
  m <- mean(df$value, na.rm = TRUE)
  s <- sd(df$value, na.rm = TRUE)
  df <- df[abs(df$value - m) < 3 * s, ]               # remove outliers
  df$value_norm <- (df$value - min(df$value)) /
                   (max(df$value) - min(df$value))     # normalize 0-1
  df
}

cleaned <- lapply(datasets, clean_dataset)

# Summary per dataset
summary_list <- Map(function(orig, clean, name) {
  data.frame(
    dataset    = name,
    n_orig     = nrow(orig),
    n_clean    = nrow(clean),
    pct_kept   = round(100 * nrow(clean) / nrow(orig), 1),
    mean_value = round(mean(clean$value), 2)
  )
}, datasets, cleaned, names(datasets))

summary_df <- do.call(rbind, summary_list)
print(summary_df, row.names = FALSE)

# Combine all cleaned data
all_clean   <- do.call(rbind, cleaned)
group_means <- tapply(all_clean$value, all_clean$label, mean)
cat("\nMean value by label (all datasets):\n")
for (lbl in names(group_means)) cat(sprintf("  %s: %.2f\n", lbl, group_means[[lbl]]))
```

---

## analyze.R: Chapter 10 Version

What changed: `analyze.R` can now accept multiple CSV files. `lapply` processes each one; `do.call(rbind, ...)` combines results.

```r
# analyze.R — Chapter 10 (multi-file support shown)

# Collect all non-flag arguments as filenames
filenames <- args[!grepl("^--", args)]

if (length(filenames) == 0) {
  cat("Error: no input files specified\n")
  quit(status = 1)
}

# Check they all exist
missing_files <- filenames[!file.exists(filenames)]
if (length(missing_files) > 0) {
  cat("Error: file(s) not found:", paste(missing_files, collapse=", "), "\n")
  quit(status = 1)
}

# Read all files
read_file <- function(f, config) {
  df <- read.csv(f,
    sep              = config$input$sep,
    header           = config$input$header,
    na.strings       = config$input$na_strings,
    stringsAsFactors = FALSE
  )
  df$.source <- f   # track which file each row came from
  df
}

dfs <- lapply(filenames, read_file, config = config)

# Check columns are compatible
col_sets  <- lapply(dfs, names)
common_cols <- Reduce(intersect, col_sets)
if (length(common_cols) < ncol(dfs[[1]])) {
  cat(sprintf("Note: combining on %d common columns (of %d)\n",
              length(common_cols), ncol(dfs[[1]])))
}

# Combine
df_all <- do.call(rbind, lapply(dfs, function(d) d[, common_cols, drop=FALSE]))

cat(sprintf("Combined %d files: %d rows × %d columns\n",
            length(filenames), nrow(df_all), ncol(df_all)))
```

Run it:

```
Rscript analyze.R jan.csv feb.csv mar.csv --group=dept --stat=mean
```

---

## Exercises

**1. Apply practice**

Given:
```r
data <- list(a = c(1,5,3,2), b = c(10,2,8), c = c(4,4,4,4,4))
```
Using only apply-family functions (no for loops):
- Find the mean of each
- Find the range (max - min) of each
- Find the sum of all values across all lists

**2. tapply report**

Use `tapply` to compute mean cost by department, readmission rate by gender AND department (two-way), and mean age by readmitted status. Use any patient dataset.

**3. Functional pipeline**

Write a `pipeline` function:
```r
pipeline <- function(x, ...) {
  fns <- list(...)
  Reduce(function(v, f) f(v), fns, x)
}

pipeline(c(1, -2, 3, -4, 5),
  function(x) x[x > 0],
  sum,
  sqrt)
# Expected: sqrt(1 + 3 + 5) = sqrt(9) = 3
```

**4. Parallel Map**

Use `Map` to compute pairwise dot products:
```r
vecs1 <- list(c(1,2,3), c(4,5,6), c(7,8,9))
vecs2 <- list(c(1,0,0), c(0,1,0), c(0,0,1))
# Expected: 1, 5, 9
```

**5. The growing program (do this one)**

`analyze.R` currently combines all input files into one data frame and analyzes them together. Add a `--per-file` flag that instead processes each file separately and shows per-file statistics side by side:

```
Rscript analyze.R jan.csv feb.csv --stat=mean,n --per-file
```

Output:
```
Column: salary
  jan.csv : n=50   mean=72000
  feb.csv : n=48   mean=74500
```

Use `sapply` to extract the same statistic from each file's results. In Chapter 11, we'll wrap the whole analysis object (data + config + results) into an S3 class called `Dataset`.

---

*Next: Chapter 11 — Packages and OOP: wrap analyze.R into a Dataset class*

---

## Solutions

### Exercise 1 — Apply practice

```r
data <- list(a = c(1,5,3,2), b = c(10,2,8), c = c(4,4,4,4,4))

# Mean of each
sapply(data, mean)
#    a    b    c
# 2.75 6.67 4.00

# Range (max - min) of each
sapply(data, function(x) max(x) - min(x))
# a  b  c
# 4  8  0

# Sum of ALL values across all lists
Reduce("+", lapply(data, sum))   # 11 + 20 + 20 = 51
# or equivalently:
sum(unlist(data))                # 51
```

### Exercise 2 — tapply report

```r
set.seed(123)
n <- 200
patients <- data.frame(
  age        = round(rnorm(n, 45, 15)),
  gender     = sample(c("M","F"), n, replace = TRUE),
  department = sample(c("Cardiology","Neurology","Orthopedics","Emergency"), n, replace=TRUE),
  los_days   = round(pmax(1, rnorm(n, 5, 3))),
  cost       = round(pmax(500, rnorm(n, 8000, 4000)), -2),
  readmitted = sample(0:1, n, replace = TRUE, prob = c(0.8, 0.2)),
  stringsAsFactors = FALSE
)

# Mean cost by department
cat("Mean cost by department:\n")
print(round(tapply(patients$cost, patients$department, mean), 0))

# Two-way readmission rate (department × gender)
cat("\nReadmission rate by department and gender:\n")
print(round(tapply(patients$readmitted,
                   list(patients$department, patients$gender),
                   mean), 3))

# Mean age by readmitted status
cat("\nMean age by readmission status:\n")
print(round(tapply(patients$age, patients$readmitted, mean), 1))
```

### Exercise 3 — Functional pipeline

```r
pipeline <- function(x, ...) {
  fns <- list(...)
  Reduce(function(v, f) f(v), fns, x)
}

# Test 1: filter positives → sum → sqrt
pipeline(c(1, -2, 3, -4, 5),
  function(x) x[x > 0],
  sum,
  sqrt)
# sqrt(1 + 3 + 5) = sqrt(9) = 3

# Test 2: normalize → round to 2 decimals → sort
normalize <- function(x) (x - min(x)) / (max(x) - min(x))
pipeline(c(3, 1, 4, 1, 5, 9, 2, 6),
  normalize,
  function(x) round(x, 2),
  sort)
# 0.00 0.00 0.12 0.25 0.38 0.50 0.62 1.00
```

### Exercise 4 — Parallel Map (dot products)

```r
vecs1 <- list(c(1,2,3), c(4,5,6), c(7,8,9))
vecs2 <- list(c(1,0,0), c(0,1,0), c(0,0,1))

# Dot product of two vectors: sum of element-wise products
dot <- function(a, b) sum(a * b)

# Map applies dot to paired elements
results <- Map(dot, vecs1, vecs2)
unlist(results)
# 1 5 9
# Explanation:
# c(1,2,3)·c(1,0,0) = 1+0+0 = 1
# c(4,5,6)·c(0,1,0) = 0+5+0 = 5
# c(7,8,9)·c(0,0,1) = 0+0+9 = 9
```

### Exercise 5 — The growing program (`--per-file` flag)

```r
# --- Addition to analyze.R (Chapter 10) ---

# After reading and coercing all files (dfs is a named list of data frames):

per_file_flag <- "--per-file" %in% args

if (per_file_flag && length(filenames) > 1) {
  # Per-file stats using sapply
  cat("\nPer-file statistics:\n")

  # Determine common numeric columns across all files
  num_cols_per_file <- lapply(dfs, function(d) names(d)[sapply(d, is.numeric)])
  common_num_cols   <- Reduce(intersect, num_cols_per_file)

  req_stats <- if (!is.null(get_arg(args, "--stat")))
                 strsplit(get_arg(args, "--stat"), ",")[[1]]
               else c("n", "mean")

  for (col in common_num_cols) {
    cat(sprintf("\nColumn: %s\n", col))

    # sapply extracts stats for this column from each file
    per_file_stats <- sapply(seq_along(filenames), function(i) {
      x <- dfs[[i]][[col]]
      s <- compute_stats(x[!is.na(x)], req_stats)
      sapply(names(s), function(nm) s[[nm]])
    })
    colnames(per_file_stats) <- basename(filenames)

    for (nm in rownames(per_file_stats)) {
      vals <- per_file_stats[nm, ]
      val_str <- paste(sprintf("%-12s %s", basename(filenames),
                               ifelse(is.na(vals), "NA",
                                      sprintf("%.2f", vals))),
                       collapse="  |  ")
      cat(sprintf("  %-8s: %s\n", nm, val_str))
    }
  }
} else {
  # Default: combine all files and analyze together (existing behavior)
  df_all <- do.call(rbind, lapply(dfs, function(d) {
    common <- Reduce(intersect, lapply(dfs, names))
    d[, common, drop = FALSE]
  }))
  # ... continue with existing analysis
}

# Rscript analyze.R jan.csv feb.csv --stat=mean,n --per-file
#
# Per-file statistics:
#
# Column: salary
#   n       : jan.csv      50.00  |  feb.csv      48.00
#   mean    : jan.csv   72000.00  |  feb.csv   74500.00
```
