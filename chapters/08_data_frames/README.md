# Chapter 8: Data Frames

*Subsetting, transforming, aggregating, and merging tabular data.*

---

## The Data Frame in Depth

A data frame is a list of vectors, all the same length. Each column is a vector (and can be a different type). Each row is one observation.

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
df["name"]                         # returns data frame (1 column)
df$name                            # returns vector
df[["name"]]                       # returns vector (same as $)
```

Combine:

```r
df[df$score >= 90, c("name", "score")]   # high scorers, name + score only
```

---

## Adding and Removing Columns

```r
# Add
df$grade <- ifelse(df$score >= 90, "A",
            ifelse(df$score >= 80, "B", "C"))

# Add with transform()
df <- transform(df, 
  score_z = (score - mean(score)) / sd(score),
  senior  = age > 28
)

# Remove
df$senior <- NULL

# Multiple remove
df[c("score_z", "grade")] <- NULL
```

---

## Ordering

```r
# Sort by score descending
df[order(-df$score), ]

# Sort by multiple columns
df[order(df$age, -df$score), ]

# Sort in place
df <- df[order(df$name), ]
```

---

## Aggregation: tapply and aggregate

```r
# tapply: apply function by group
tapply(df$score, df$grade, mean)

# aggregate: more powerful
aggregate(score ~ grade, data = df, FUN = mean)
aggregate(cbind(score, age) ~ grade, data = df, FUN = mean)
```

---

## Merging Data Frames

Like SQL joins:

```r
employees <- data.frame(
  id   = 1:5,
  name = c("Alice", "Bob", "Carol", "Dave", "Eve"),
  dept_id = c(1, 2, 1, 3, 2)
)

departments <- data.frame(
  dept_id   = 1:3,
  dept_name = c("Engineering", "Sales", "HR")
)

# Inner join (only matching rows)
merged <- merge(employees, departments, by = "dept_id")

# Left join (all employees, even without dept)
left <- merge(employees, departments, by = "dept_id", all.x = TRUE)

# All = outer join
outer <- merge(employees, departments, by = "dept_id", all = TRUE)
```

---

## rbind and cbind

```r
# Stack data frames vertically
df1 <- data.frame(x = 1:3, y = c("a","b","c"))
df2 <- data.frame(x = 4:6, y = c("d","e","f"))
combined <- rbind(df1, df2)

# Add columns horizontally
df_extra <- data.frame(z = c(10, 20, 30, 40, 50, 60))
full <- cbind(combined, df_extra)
```

---

## Program: Exploratory Analysis Pipeline

```r
# eda.R
# Complete exploratory data analysis pipeline

# Simulate a dataset: hospital patient data
set.seed(123)
n <- 200

generate_patients <- function(n) {
  data.frame(
    id          = sprintf("P%04d", 1:n),
    age         = round(rnorm(n, mean = 45, sd = 15)),
    gender      = sample(c("M", "F"), n, replace = TRUE),
    department  = sample(c("Cardiology", "Neurology", "Orthopedics",
                           "Oncology", "Emergency"), n, replace = TRUE),
    los_days    = round(pmax(1, rnorm(n, mean = 5, sd = 3))), # length of stay
    readmitted  = sample(c(TRUE, FALSE), n, replace = TRUE, prob = c(0.15, 0.85)),
    cost        = round(pmax(500, rnorm(n, mean = 8000, sd = 4000)), -2),
    stringsAsFactors = FALSE
  )
}

patients <- generate_patients(n)

# ---- Helper: print a labeled section ----
section <- function(title) {
  cat("\n", strrep("=", 50), "\n", title, "\n", strrep("=", 50), "\n", sep="")
}

section("DATASET OVERVIEW")
cat(sprintf("Rows: %d  |  Columns: %d\n", nrow(patients), ncol(patients)))
cat("Column types:\n")
for (col in names(patients)) {
  cat(sprintf("  %-15s : %s\n", col, class(patients[[col]])))
}

# ---- Missing values ----
section("MISSING VALUES")
na_counts <- sapply(patients, function(x) sum(is.na(x)))
if (any(na_counts > 0)) {
  print(na_counts[na_counts > 0])
} else {
  cat("No missing values.\n")
}

# ---- Numeric summaries ----
section("NUMERIC SUMMARIES")
num_cols <- c("age", "los_days", "cost")
for (col in num_cols) {
  x <- patients[[col]]
  cat(sprintf("\n%s:\n", col))
  cat(sprintf("  Mean ± SD : %.1f ± %.1f\n", mean(x), sd(x)))
  cat(sprintf("  Median    : %.1f\n", median(x)))
  cat(sprintf("  Range     : [%.0f, %.0f]\n", min(x), max(x)))
  cat(sprintf("  IQR       : %.1f\n", IQR(x)))
}

# ---- Categorical summaries ----
section("CATEGORICAL SUMMARIES")
cat_cols <- c("gender", "department", "readmitted")
for (col in cat_cols) {
  cat(sprintf("\n%s:\n", col))
  freq <- table(patients[[col]])
  pct  <- round(100 * prop.table(freq), 1)
  for (nm in names(freq)) {
    cat(sprintf("  %-15s : %3d (%5.1f%%)\n", nm, freq[[nm]], pct[[nm]]))
  }
}

# ---- Department analysis ----
section("DEPARTMENT ANALYSIS")
dept_stats <- aggregate(
  cbind(age, los_days, cost) ~ department,
  data = patients,
  FUN  = mean
)
dept_stats <- round(dept_stats[, -1], 1)
dept_stats <- cbind(
  department = unique(patients$department)[order(unique(patients$department))],
  dept_stats
)
dept_stats <- dept_stats[order(dept_stats$cost, decreasing = TRUE), ]

cat(sprintf("%-15s  %6s  %8s  %10s\n", "Department", "Avg Age", "Avg LOS", "Avg Cost"))
cat(strrep("-", 45), "\n")
for (i in seq_len(nrow(dept_stats))) {
  cat(sprintf("%-15s  %6.1f  %8.1f  %10.0f\n",
              dept_stats$department[i],
              dept_stats$age[i],
              dept_stats$los_days[i],
              dept_stats$cost[i]))
}

# ---- Readmission analysis ----
section("READMISSION ANALYSIS")
readmit_rate <- mean(patients$readmitted) * 100
cat(sprintf("Overall readmission rate: %.1f%%\n", readmit_rate))

dept_readmit <- tapply(patients$readmitted, patients$department, mean) * 100
dept_readmit <- sort(dept_readmit, decreasing = TRUE)
cat("\nBy department:\n")
for (dept in names(dept_readmit)) {
  bar <- strrep("█", round(dept_readmit[dept] / 2))
  cat(sprintf("  %-15s : %5.1f%%  %s\n", dept, dept_readmit[dept], bar))
}

# ---- Cost outliers ----
section("COST OUTLIERS")
q1 <- quantile(patients$cost, 0.25)
q3 <- quantile(patients$cost, 0.75)
iqr <- q3 - q1
outlier_threshold <- q3 + 1.5 * iqr

outliers <- patients[patients$cost > outlier_threshold, 
                     c("id", "department", "los_days", "cost")]
outliers <- outliers[order(-outliers$cost), ]
cat(sprintf("Patients with cost > $%s (Q3 + 1.5×IQR):\n",
            formatC(outlier_threshold, format="d", big.mark=",")))
for (i in seq_len(min(5, nrow(outliers)))) {
  cat(sprintf("  %s  %-15s  %d days  $%s\n",
              outliers$id[i],
              outliers$department[i],
              outliers$los_days[i],
              formatC(outliers$cost[i], format="d", big.mark=",")))
}
```

---

## Exercises

**1. Subset and transform**

With the `patients` dataset:
- Find all patients over 60 in Cardiology
- Add a `cost_per_day` column (cost / los_days)
- Find the 5 patients with the highest cost_per_day

**2. Merge practice**

Create a `treatments` data frame with columns `id` and `treatment` (random). Merge it with `patients`. Find the most common treatment for each department.

**3. Group statistics**

Using `aggregate`, compute for each department:
- Mean, median, min, max of `cost`
- Standard deviation of `los_days`

**4. Reshape for reporting**

Create a 2-way table showing readmission rate by department and gender. (Hint: `tapply(readmitted, list(department, gender), mean)`)

---

*Next: Chapter 9 — Lists and Environments: flexible containers*
