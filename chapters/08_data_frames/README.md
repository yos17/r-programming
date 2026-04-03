# Chapter 8: Data Frames

*Subsetting, transforming, aggregating, and merging tabular data.*

---

## Where We Left Off

`analyze.R` now reads a real CSV and computes statistics for each numeric column. Run it on our employees file:

```
Rscript analyze.R employees.csv --stat=mean,sd
```

It reports the overall mean salary. But what we really want is:

```
Rscript analyze.R employees.csv --group=dept --stat=mean
```

Output:
```
--- salary by dept ---
  Engineering : mean = 83500.00
  Sales       : mean = 65500.00
  HR          : mean = 55000.00
```

That requires grouping. Grouping requires understanding data frames — R's core structure for tabular data.

---

## The Data Frame

A data frame is a list of vectors, all the same length. Each column is a vector (can be different types). Each row is one observation.

```r
df <- data.frame(
  name   = c("Alice", "Bob", "Carol", "Dave", "Eve"),
  age    = c(25, 30, 28, 35, 22),
  score  = c(92.5, 78.0, 95.5, 84.0, 88.5),
  passed = c(TRUE, TRUE, TRUE, TRUE, TRUE),
  stringsAsFactors = FALSE
)
```

---

## Subsetting

Three equivalent ways to subset rows:

```r
df[df$age > 27, ]                  # logical indexing
df[c(1, 3, 5), ]                   # by row number
subset(df, age > 27)               # subset() function
```

Subset columns:

```r
df[, c("name", "score")]           # by name
df[, 1:3]                          # by position
df$name                            # column as vector
df[["name"]]                       # same thing
```

Combine:

```r
df[df$score >= 90, c("name", "score")]
```

---

## Adding and Removing Columns

```r
# Add
df$grade <- ifelse(df$score >= 90, "A",
            ifelse(df$score >= 80, "B", "C"))

# Add with transform()
df <- transform(df, 
  score_z = (score - mean(score)) / sd(score)
)

# Remove
df$grade <- NULL
df[c("score_z")] <- NULL
```

---

## Ordering

```r
df[order(-df$score), ]                  # sort by score descending
df[order(df$age, -df$score), ]          # sort by multiple columns
```

---

## The Wrong Way to Group

The most natural approach when you first learn R: a for loop.

```r
# Loop approach — works but verbose
depts <- unique(df$dept)
for (d in depts) {
  sub <- df[df$dept == d, ]
  cat(sprintf("  %s: mean salary = %.0f\n", d, mean(sub$salary)))
}
```

This works. But R has a better way.

---

## tapply: Apply by Group

`tapply` groups a vector by a factor and applies a function:

```r
tapply(df$salary, df$dept, mean)
# Engineering      HR    Sales
#    83500.00  55000.0  65500.0

tapply(df$salary, df$dept, length)  # count per group
tapply(df$salary, df$dept, sd)      # SD per group
```

One line instead of six. Faster. Cleaner.

---

## aggregate: Multiple Columns, Multiple Stats

```r
# Mean of salary and years, by dept
aggregate(cbind(salary, years) ~ dept, data = df, FUN = mean)
```

Output:
```
          dept   salary years
1  Engineering 83500.00   2.5
2           HR 55000.00  10.0
3        Sales 65500.00   6.0
```

`cbind(salary, years) ~ dept` means "salary and years, grouped by dept." This SQL-like formula syntax is used throughout R's statistical functions.

---

## Merging Data Frames

Like SQL joins:

```r
employees <- data.frame(
  id   = 1:5,
  name = c("Alice","Bob","Carol","Dave","Eve"),
  dept_id = c(1, 2, 1, 3, 2)
)
departments <- data.frame(
  dept_id   = 1:3,
  dept_name = c("Engineering","Sales","HR")
)

merged <- merge(employees, departments, by = "dept_id")      # inner join
left   <- merge(employees, departments, by = "dept_id", all.x = TRUE)  # left join
```

---

## rbind and cbind

```r
df1 <- data.frame(x = 1:3, y = c("a","b","c"))
df2 <- data.frame(x = 4:6, y = c("d","e","f"))
combined <- rbind(df1, df2)          # stack vertically (must have same columns)
```

---

## Exploratory Analysis Pipeline

```r
# eda.R — quick EDA on any data frame

eda <- function(df, name = "dataset") {
  cat(sprintf("=== %s ===\n", name))
  cat(sprintf("Dimensions: %d rows × %d columns\n\n", nrow(df), ncol(df)))
  
  # Column types and NAs
  cat("Columns:\n")
  for (col in names(df)) {
    x <- df[[col]]
    na_pct <- 100 * sum(is.na(x)) / length(x)
    type_info <- if (is.numeric(x)) {
      sprintf("numeric  [%.1f, %.1f]", min(x, na.rm=T), max(x, na.rm=T))
    } else {
      sprintf("character %d unique", length(unique(x[!is.na(x)])))
    }
    cat(sprintf("  %-15s : %-30s NAs: %.0f%%\n", col, type_info, na_pct))
  }
}

# Patient dataset example
set.seed(123)
n <- 200
patients <- data.frame(
  id         = sprintf("P%04d", 1:n),
  age        = round(rnorm(n, 45, 15)),
  gender     = sample(c("M","F"), n, replace = TRUE),
  department = sample(c("Cardiology","Neurology","Orthopedics","Emergency"), n, replace=TRUE),
  los_days   = round(pmax(1, rnorm(n, 5, 3))),
  cost       = round(pmax(500, rnorm(n, 8000, 4000)), -2),
  stringsAsFactors = FALSE
)

eda(patients, "Hospital Patients")

# Group analysis
cat("\n--- Cost by Department ---\n")
dept_stats <- aggregate(cbind(age, los_days, cost) ~ department,
                        data = patients, FUN = mean)
dept_stats <- dept_stats[order(-dept_stats$cost), ]
print(round(dept_stats, 1))

# Cost outliers
q3  <- quantile(patients$cost, 0.75)
iqr <- IQR(patients$cost)
outliers <- patients[patients$cost > q3 + 1.5 * iqr,
                     c("id","department","los_days","cost")]
cat(sprintf("\n%d cost outliers (> Q3 + 1.5×IQR = $%.0f):\n",
            nrow(outliers), q3 + 1.5 * iqr))
print(head(outliers[order(-outliers$cost), ], 5))
```

---

## analyze.R: Chapter 8 Version

What changed: added `--group` argument. When present, compute stats per group using `tapply`.

```r
# analyze.R — Chapter 8 (grouping logic shown)

# --- Parse --group argument ---
group_col <- get_arg(args, "--group")

# --- Compute stats ---
if (!is.null(group_col) && group_col %in% names(df)) {
  cat(sprintf("\nGrouped by: %s\n", group_col))
  groups <- sort(unique(df[[group_col]]))
  
  for (col in names(df)) {
    if (!is.numeric(df[[col]])) next
    
    cat(sprintf("\n--- %s by %s ---\n", col, group_col))
    
    # tapply computes the stat per group
    group_means <- tapply(df[[col]], df[[group_col]], mean, na.rm = TRUE)
    group_sds   <- tapply(df[[col]], df[[group_col]], sd,   na.rm = TRUE)
    group_ns    <- tapply(df[[col]], df[[group_col]], length)
    
    for (g in sort(names(group_means))) {
      cat(sprintf("  %-20s : n=%-3d  mean=%.2f  sd=%.2f\n",
                  g, group_ns[g], group_means[g],
                  ifelse(is.na(group_sds[g]), 0, group_sds[g])))
    }
  }
} else {
  # Overall stats (existing code from Chapter 7)
  for (col in names(df)) {
    if (!is.numeric(df[[col]])) next
    stats <- compute_stats(df[[col]][!is.na(df[[col]])], requested_stats)
    cat(sprintf("\n--- %s ---\n", col))
    for (nm in names(stats)) cat(sprintf("  %-8s = %.4f\n", nm, stats[[nm]]))
  }
}
```

Run it:

```
Rscript analyze.R employees.csv --group=dept --stat=mean,sd
```

Output:
```
Read 5 rows × 4 columns from employees.csv
Grouped by: dept

--- salary by dept ---
  Engineering          : n=2   mean=83500.00  sd=16263.46
  HR                   : n=1   mean=55000.00  sd=0.00
  Sales                : n=2   mean=65500.00  sd=24748.74
```

---

## Exercises

**1. Subset and transform**

With the `patients` dataset:
- Find all patients over 60 in Cardiology
- Add a `cost_per_day` column (cost / los_days)
- Find the 5 patients with highest cost_per_day

**2. Merge practice**

Create a `treatments` data frame with columns `id` and `treatment`. Merge with `patients`. Find the most common treatment per department.

**3. Group statistics**

Using `aggregate`, compute for each department:
- Mean, median, min, max of `cost`
- Standard deviation of `los_days`

**4. Two-way table**

Create a readmission rate table by department and gender:
```r
tapply(patients$readmitted, list(patients$department, patients$gender), mean)
```

**5. The growing program (do this one)**

Add `--group` to `analyze.R`. When grouping, compute stats for *all* numeric columns, not just one. The output should look like:

```
--- salary by dept ---
  Engineering : n=2  mean=83500  sd=16263
  HR          : n=1  mean=55000  sd=NA
  Sales       : n=2  mean=65500  sd=24749

--- years by dept ---
  Engineering : n=2  mean=2.5    sd=0.71
  ...
```

In Chapter 9, we'll add a config system to control which statistics are computed and how the output is formatted, without changing the main program logic.

---

*Next: Chapter 9 — Lists and Environments: a config system*

---

## Solutions

### Exercise 1 — Subset and transform

```r
# Build the patients dataset (from the chapter)
set.seed(123)
n <- 200
patients <- data.frame(
  id         = sprintf("P%04d", 1:n),
  age        = round(rnorm(n, 45, 15)),
  gender     = sample(c("M","F"), n, replace = TRUE),
  department = sample(c("Cardiology","Neurology","Orthopedics","Emergency"), n, replace=TRUE),
  los_days   = round(pmax(1, rnorm(n, 5, 3))),
  cost       = round(pmax(500, rnorm(n, 8000, 4000)), -2),
  stringsAsFactors = FALSE
)

# Patients over 60 in Cardiology
cardio_seniors <- patients[patients$age > 60 & patients$department == "Cardiology", ]
cat(sprintf("Cardiology patients over 60: %d\n", nrow(cardio_seniors)))

# Add cost_per_day column
patients$cost_per_day <- patients$cost / patients$los_days

# Top 5 by cost_per_day
top5 <- head(patients[order(-patients$cost_per_day), c("id","department","los_days","cost","cost_per_day")], 5)
cat("\nTop 5 patients by cost per day:\n")
print(top5, row.names = FALSE)
```

### Exercise 2 — Merge practice

```r
# Add a treatments column to the existing patients dataset
set.seed(7)
treatments <- data.frame(
  id        = patients$id[sample(n, n, replace = FALSE)],
  treatment = sample(c("Surgery","Medication","Physical Therapy","Observation"),
                     n, replace = TRUE),
  stringsAsFactors = FALSE
)

# Inner join on id
merged <- merge(patients, treatments, by = "id")

# Most common treatment per department
cat("Most common treatment by department:\n")
for (dept in sort(unique(merged$department))) {
  sub  <- merged[merged$department == dept, ]
  freq <- sort(table(sub$treatment), decreasing = TRUE)
  cat(sprintf("  %-14s → %s (%d)\n", dept, names(freq)[1], freq[1]))
}
```

### Exercise 3 — Group statistics

```r
# Mean, median, min, max of cost, and SD of los_days — by department
cat("\n--- Cost statistics by department ---\n")
cost_stats <- aggregate(cost ~ department, data = patients,
                        FUN = function(x) c(mean=mean(x), median=median(x),
                                            min=min(x),   max=max(x)))
# aggregate returns a matrix in the second column; flatten it
cost_df <- do.call(data.frame, cost_stats)
names(cost_df) <- c("department", "mean", "median", "min", "max")
print(round(cost_df, 0), row.names = FALSE)

cat("\n--- LOS SD by department ---\n")
los_sd <- aggregate(los_days ~ department, data = patients, FUN = sd)
print(round(los_sd, 2), row.names = FALSE)
```

### Exercise 4 — Two-way table

```r
# Add a readmitted column (random, for demonstration)
set.seed(3)
patients$readmitted <- sample(c(0, 1), n, replace = TRUE, prob = c(0.8, 0.2))

# Readmission rate by department × gender
readmit_table <- tapply(patients$readmitted,
                        list(patients$department, patients$gender),
                        mean)

cat("\nReadmission rate by department and gender:\n")
print(round(readmit_table, 3))
```

### Exercise 5 — The growing program (`--group` for all numeric columns)

```r
# --- Addition to analyze.R (Chapter 8) ---
# Replace or extend the existing grouping logic with this version.

# Assumes: df is the (filtered) data frame, group_col from args,
#          compute_stats() from Chapter 4/5.

group_col   <- get_arg(args, "--group")
req_stats   <- if (!is.null(get_arg(args, "--stat")))
                 strsplit(get_arg(args, "--stat"), ",")[[1]]
               else c("n", "mean", "sd")

if (!is.null(group_col) && group_col %in% names(df)) {
  cat(sprintf("\nGrouped by: %s\n", group_col))

  # Find all numeric columns (excluding the grouping column)
  num_cols <- names(df)[sapply(df, is.numeric)]
  num_cols <- num_cols[num_cols != group_col]

  for (col in num_cols) {
    cat(sprintf("\n--- %s by %s ---\n", col, group_col))

    groups <- sort(unique(df[[group_col]]))
    for (g in groups) {
      x <- df[[col]][df[[group_col]] == g]
      s <- compute_stats(x[!is.na(x)], req_stats)

      stat_str <- paste(
        mapply(function(nm, val) {
          if (is.na(val)) sprintf("%s=NA", nm)
          else            sprintf("%s=%.2f", nm, val)
        }, names(s), unlist(s)),
        collapse="  "
      )
      cat(sprintf("  %-20s : %s\n", g, stat_str))
    }
  }
} else {
  # Overall stats for each numeric column
  num_cols <- names(df)[sapply(df, is.numeric)]
  for (col in num_cols) {
    s <- compute_stats(df[[col]][!is.na(df[[col]])], req_stats)
    cat(sprintf("\n--- %s ---\n", col))
    for (nm in names(s)) {
      val <- s[[nm]]
      cat(sprintf("  %-8s = %s\n", nm, if (is.na(val)) "NA" else sprintf("%.4f", val)))
    }
  }
}

# Rscript analyze.R employees.csv --group=dept --stat=n,mean,sd
#
# Grouped by: dept
#
# --- salary by dept ---
#   Engineering          : n=2.00  mean=83500.00  sd=16263.46
#   HR                   : n=1.00  mean=55000.00  sd=NA
#   Sales                : n=2.00  mean=65500.00  sd=24748.74
#
# --- years by dept ---
#   Engineering          : n=2.00  mean=2.50  sd=0.71
#   HR                   : n=1.00  mean=10.00  sd=NA
#   Sales                : n=2.00  mean=6.00  sd=1.41
```
