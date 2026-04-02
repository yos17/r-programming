# Chapter 7: I/O and Files

*Reading and writing data. CSV, text files, and a data pipeline.*

---

## The Working Directory

R reads and writes files relative to the **working directory**.

```r
getwd()            # show current working directory
setwd("~/Projects/r-programming")   # change it
```

In RStudio: the **Files** panel shows your working directory. You can also set it via `Session → Set Working Directory`.

**Best practice**: at the top of each project script, set the working directory explicitly, or use RStudio Projects (`.Rproj` files) which set it automatically.

---

## Writing Text Files

```r
# Write a character vector to a file (one element per line)
lines <- c("Line 1", "Line 2", "Line 3")
writeLines(lines, "output.txt")

# Append to a file
cat("Another line\n", file = "output.txt", append = TRUE)

# Write with cat
cat("Name:", "Alice", "\n", file = "report.txt")
cat("Score:", 95, "\n", file = "report.txt", append = TRUE)
```

---

## Reading Text Files

```r
# Read all lines into a character vector
lines <- readLines("output.txt")
lines          # c("Line 1", "Line 2", "Line 3")
length(lines)  # number of lines

# Read a URL directly
lines <- readLines("https://gutenberg.org/files/1342/1342-0.txt")
```

---

## Writing CSV

```r
# Create some data
df <- data.frame(
  name  = c("Alice", "Bob", "Carol"),
  score = c(92, 78, 95),
  grade = c("A", "C", "A")
)

write.csv(df, "scores.csv", row.names = FALSE)
```

Open `scores.csv` in a text editor — it's just comma-separated text:
```
"name","score","grade"
"Alice",92,"A"
"Bob",78,"C"
"Carol",95,"A"
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
  header     = TRUE,    # first row = column names (default)
  sep        = ",",     # separator (use ";" for European CSV)
  dec        = ".",     # decimal point
  na.strings = "NA",    # what to treat as NA
  skip       = 2,       # skip first 2 lines
  nrows      = 100,     # read only 100 rows
  stringsAsFactors = FALSE  # don't convert strings to factors
)
```

For European-style CSV (semicolon-separated, comma as decimal):
```r
read.csv2("data.csv")   # sep=";", dec=","
```

---

## The Data Frame

A data frame is R's core data structure for tabular data — a list of equal-length vectors.

```r
df <- data.frame(
  name   = c("Alice", "Bob", "Carol", "Dave"),
  age    = c(25, 30, 28, 35),
  salary = c(50000, 60000, 55000, 70000),
  active = c(TRUE, TRUE, FALSE, TRUE)
)

# Access columns
df$name           # column as vector
df[["name"]]      # same thing
df[, "name"]      # same thing
df[, 1]           # by position

# Access rows
df[1, ]           # first row
df[df$age > 28, ] # rows where age > 28

# Access a cell
df[2, "salary"]   # 60000
```

---

## Program: CSV Data Processor

```r
# csv_processor.R
# Read, clean, analyze, and write CSV data

# ---- Generate test data ----
set.seed(42)
n <- 50

generate_data <- function(n) {
  departments <- c("Engineering", "Sales", "Marketing",
                   "HR", "Finance")
  data.frame(
    id         = 1:n,
    name       = paste("Employee", 1:n),
    department = sample(departments, n, replace = TRUE),
    salary     = round(runif(n, 35000, 120000), -3),
    years      = sample(1:15, n, replace = TRUE),
    rating     = sample(1:5, n, replace = TRUE),
    active     = sample(c(TRUE, FALSE), n, replace = TRUE, prob = c(0.9, 0.1))
  )
}

employees <- generate_data(n)
write.csv(employees, "employees.csv", row.names = FALSE)
cat("Generated employees.csv with", nrow(employees), "rows\n\n")

# ---- Read and inspect ----
df <- read.csv("employees.csv", stringsAsFactors = FALSE)

cat("=== Data Overview ===\n")
cat(sprintf("Rows: %d, Columns: %d\n", nrow(df), ncol(df)))
cat("Columns:", paste(names(df), collapse = ", "), "\n")
cat("\nFirst 3 rows:\n")
print(head(df, 3))

# ---- Clean ----
# Filter active employees
df_active <- df[df$active == TRUE, ]
cat(sprintf("\nActive employees: %d (of %d)\n", nrow(df_active), nrow(df)))

# ---- Summarize by department ----
cat("\n=== Department Summary ===\n")
depts <- unique(df_active$department)

dept_summary <- data.frame(
  department = depts,
  count      = sapply(depts, function(d) sum(df_active$department == d)),
  avg_salary = sapply(depts, function(d) mean(df_active$salary[df_active$department == d])),
  avg_years  = sapply(depts, function(d) mean(df_active$years[df_active$department == d])),
  avg_rating = sapply(depts, function(d) mean(df_active$rating[df_active$department == d]))
)

# Sort by average salary
dept_summary <- dept_summary[order(-dept_summary$avg_salary), ]

cat(sprintf("%-15s  %5s  %10s  %9s  %10s\n",
            "Department", "Count", "Avg Salary", "Avg Years", "Avg Rating"))
cat(strrep("-", 58), "\n")
for (i in seq_len(nrow(dept_summary))) {
  cat(sprintf("%-15s  %5d  %10.0f  %9.1f  %10.2f\n",
              dept_summary$department[i],
              dept_summary$count[i],
              dept_summary$avg_salary[i],
              dept_summary$avg_years[i],
              dept_summary$avg_rating[i]))
}

# ---- Top earners ----
cat("\n=== Top 5 Earners ===\n")
top5 <- head(df_active[order(-df_active$salary), ], 5)
for (i in seq_len(nrow(top5))) {
  cat(sprintf("  %s (%s): $%s/yr\n",
              top5$name[i],
              top5$department[i],
              formatC(top5$salary[i], format="d", big.mark=",")))
}

# ---- Write summary ----
write.csv(dept_summary, "dept_summary.csv", row.names = FALSE)
cat("\nSummary written to dept_summary.csv\n")
```

---

## Reading Other Formats

```r
# Tab-separated
df <- read.table("data.tsv", header = TRUE, sep = "\t")

# Fixed-width
df <- read.fwf("data.txt", widths = c(10, 5, 8),
               col.names = c("name", "age", "score"))

# From clipboard (useful for pasting from Excel)
df <- read.table("clipboard", header = TRUE, sep = "\t")

# RDS — R's native binary format (fast, preserves types)
saveRDS(df, "data.rds")
df2 <- readRDS("data.rds")
```

---

## File and Directory Operations

```r
file.exists("data.csv")          # check if file exists
file.copy("old.csv", "new.csv")  # copy
file.rename("old.csv", "new.csv")# rename/move
file.remove("temp.csv")          # delete

dir.create("output")             # create directory
list.files(".")                  # list files in current dir
list.files(".", pattern = "*.csv")  # list only CSV files
list.files(".", recursive = TRUE)   # include subdirectories
```

---

## Exercises

**1. Sales data pipeline**

Create a CSV with columns: `date, product, quantity, price`. Generate 100 rows of random data. Read it back. Compute:
- Total revenue by product
- Daily total revenue
- The top 3 revenue days

**2. Log file parser**

Generate a fake web server log:
```
2026-03-15 10:23:45 GET /index.html 200 1234
2026-03-15 10:23:47 GET /about.html 200 876
2026-03-15 10:23:50 POST /login 302 0
2026-03-15 10:23:55 GET /missing.html 404 0
```

Write a function that reads this log and reports:
- Request count by status code
- Most requested URLs
- Number of errors (4xx status codes)

**3. Multi-file batch processing**

Write a script that:
- Generates 5 CSV files, each with random monthly data
- Reads all of them with a loop (use `list.files` + `read.csv`)
- Combines them into one data frame with `rbind`
- Writes the combined file

**4. CSV round-trip**

Write a data frame to CSV, read it back, verify that all values are identical. Where can round-trip errors occur? (Hint: check floating point numbers and date strings.)

---

*Next: Chapter 8 — Data Frames: the full toolkit for tabular data*
