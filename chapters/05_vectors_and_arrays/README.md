# Chapter 5: Vectors and Arrays

*R's native data structure. Everything is a vector. Vectorized thinking.*

---

## Everything Is a Vector

In R, there are no scalars. A "single number" is a vector of length 1.

```r
x <- 42
length(x)   # 1
is.vector(x)  # TRUE
```

This is why `print(42)` shows `[1] 42` — the `[1]` is the index of the first element displayed on that line.

---

## Creating Vectors

```r
# c() — combine
x <- c(1, 2, 3, 4, 5)

# Sequences
1:10              # 1 2 3 4 5 6 7 8 9 10
seq(1, 10, by=2)  # 1 3 5 7 9
seq(0, 1, length.out=5)  # 0.00 0.25 0.50 0.75 1.00

# Repetition
rep(0, 5)          # 0 0 0 0 0
rep(c(1,2,3), 3)   # 1 2 3 1 2 3 1 2 3
rep(c(1,2,3), each=3)  # 1 1 1 2 2 2 3 3 3

# Character
fruits <- c("apple", "banana", "cherry")

# Logical
flags <- c(TRUE, FALSE, TRUE, TRUE, FALSE)
```

Vectors are homogeneous — all elements must be the same type. Mixed types are coerced:

```r
c(1, "hello", TRUE)   # "1" "hello" "TRUE" — all character
c(1, TRUE, FALSE)     # 1 1 0 — all numeric
```

---

## Indexing

R uses 1-based indexing (not 0-based like most languages).

```r
x <- c(10, 20, 30, 40, 50)

x[1]          # 10 — first element
x[5]          # 50 — last element
x[c(1,3,5)]   # 10 30 50 — multiple elements
x[2:4]        # 20 30 40 — slice
x[-1]         # 20 30 40 50 — drop first element
x[-c(1,5)]    # 20 30 40 — drop first and last
```

**Logical indexing** — select elements where condition is TRUE:

```r
x <- c(3, 1, 4, 1, 5, 9, 2, 6)

x[x > 3]            # 4 5 9 6
x[x %% 2 == 0]      # 4 2 6
x[x > mean(x)]      # 4 5 9 6
```

**Named indexing**:

```r
scores <- c(Alice = 92, Bob = 78, Carol = 95, Dave = 84)
scores["Alice"]          # 92
scores[c("Bob","Carol")] # Bob 78, Carol 95
```

**Modifying elements**:

```r
x <- c(10, 20, 30, 40, 50)
x[3] <- 99
x                  # 10 20 99 40 50

x[x < 30] <- 0
x                  # 0 0 99 40 50
```

---

## Vectorized Operations

R's superpower: operations apply to all elements at once.

```r
x <- c(1, 2, 3, 4, 5)

x + 10        # 11 12 13 14 15
x * 2         # 2 4 6 8 10
x ^ 2         # 1 4 9 16 25
sqrt(x)       # 1.000 1.414 1.732 2.000 2.236
log(x)        # natural log of each element
```

Operations between two vectors: element-by-element:

```r
a <- c(1, 2, 3)
b <- c(10, 20, 30)
a + b    # 11 22 33
a * b    # 10 40 90
```

**Recycling** — if lengths don't match, the shorter vector repeats:

```r
x <- c(1, 2, 3, 4, 6)
x + c(10, 100)   # 11 102 13 104 16 — recycled: 10,100,10,100,10
```

Recycling is powerful but can cause silent bugs. R warns when lengths aren't multiples of each other.

---

## Vector Functions

```r
x <- c(3, 1, 4, 1, 5, 9, 2, 6, 5, 3)

length(x)         # 10
sum(x)            # 39
prod(x)           # product of all elements
cumsum(x)         # running sum
cumprod(x)        # running product
cummax(x)         # running maximum
cummin(x)         # running minimum
diff(x)           # differences: x[i+1] - x[i]

sort(x)           # sort ascending
sort(x, decreasing = TRUE)  # sort descending
order(x)          # indices that would sort x
rank(x)           # rank of each element
rev(x)            # reverse

which(x > 4)      # indices where condition is TRUE
which.min(x)      # index of minimum
which.max(x)      # index of maximum

any(x > 8)        # TRUE if any element satisfies condition
all(x > 0)        # TRUE if all elements satisfy condition
```

---

## Matrices

A matrix is a 2D vector with dimensions:

```r
m <- matrix(1:12, nrow = 3, ncol = 4)
m
```

Output:
```
     [,1] [,2] [,3] [,4]
[1,]    1    4    7   10
[2,]    2    5    8   11
[3,]    3    6    9   12
```

Note: R fills by column (column-major order). Use `byrow = TRUE` to fill by row:

```r
m <- matrix(1:12, nrow = 3, byrow = TRUE)
```

**Indexing a matrix**:

```r
m[2, 3]     # row 2, column 3
m[1, ]      # entire row 1
m[, 2]      # entire column 2
m[1:2, 3:4] # submatrix
```

**Matrix operations**:

```r
A <- matrix(c(1,2,3,4), nrow = 2)
B <- matrix(c(5,6,7,8), nrow = 2)

A + B       # element-wise addition
A * B       # element-wise multiplication
A %*% B     # matrix multiplication
t(A)        # transpose
det(A)      # determinant
solve(A)    # matrix inverse
```

---

## Program: Grade Analyzer

```r
# grades.R
# Analyze student grades

# Student data
students <- c("Alice", "Bob", "Carol", "Dave", "Eve",
              "Frank", "Grace", "Henry", "Iris", "Jack")

# Test scores (4 tests per student)
scores <- matrix(c(
  85, 92, 78, 88,   # Alice
  72, 68, 75, 80,   # Bob
  95, 98, 92, 96,   # Carol
  60, 55, 62, 58,   # Dave
  88, 85, 90, 87,   # Eve
  78, 82, 79, 84,   # Frank
  93, 91, 95, 89,   # Grace
  65, 70, 68, 72,   # Henry
  82, 87, 84, 90,   # Iris
  75, 79, 77, 81    # Jack
), nrow = 10, byrow = TRUE,
dimnames = list(students, paste("Test", 1:4)))

# Calculate averages
avg_by_student <- rowMeans(scores)
avg_by_test    <- colMeans(scores)

# Letter grades
letter_grade <- function(score) {
  ifelse(score >= 90, "A",
  ifelse(score >= 80, "B",
  ifelse(score >= 70, "C",
  ifelse(score >= 60, "D", "F"))))
}

grades <- letter_grade(avg_by_student)

# Print report
cat("=== Grade Report ===\n\n")
cat(sprintf("%-10s  %5s  %5s  %5s  %5s  %6s  %s\n",
            "Student", "T1", "T2", "T3", "T4", "Avg", "Grade"))
cat(strrep("-", 50), "\n")

for (i in seq_along(students)) {
  cat(sprintf("%-10s  %5.1f  %5.1f  %5.1f  %5.1f  %6.2f  %s\n",
              students[i],
              scores[i, 1], scores[i, 2],
              scores[i, 3], scores[i, 4],
              avg_by_student[i], grades[i]))
}

cat(strrep("-", 50), "\n")
cat(sprintf("%-10s  %5.1f  %5.1f  %5.1f  %5.1f  %6.2f\n",
            "Average",
            avg_by_test[1], avg_by_test[2],
            avg_by_test[3], avg_by_test[4],
            mean(avg_by_student)))

# Statistics
cat("\n=== Class Statistics ===\n")
cat(sprintf("Highest average: %s (%.2f)\n",
            students[which.max(avg_by_student)],
            max(avg_by_student)))
cat(sprintf("Lowest average:  %s (%.2f)\n",
            students[which.min(avg_by_student)],
            min(avg_by_student)))
cat("\nGrade distribution:\n")
print(table(grades))
```

Output:
```
=== Grade Report ===

Student      T1     T2     T3     T4     Avg  Grade
--------------------------------------------------
Alice        85.0   92.0   78.0   88.0   85.75  B
Bob          72.0   68.0   75.0   80.0   73.75  C
Carol        95.0   98.0   92.0   96.0   95.25  A
Dave         60.0   55.0   62.0   58.0   58.75  F
Eve          88.0   85.0   90.0   87.0   87.50  B
Frank        78.0   82.0   79.0   84.0   80.75  B
Grace        93.0   91.0   95.0   89.0   92.00  A
Henry        65.0   70.0   68.0   72.0   68.75  D
Iris         82.0   87.0   84.0   90.0   85.75  B
Jack         75.0   79.0   77.0   81.0   78.00  C
```

---

## Exercises

**1. Vector operations**

Given `x <- c(4, 7, 2, 9, 1, 5, 8, 3, 6)`:
- Extract elements greater than 5
- Extract every other element (indices 1, 3, 5, ...)
- Replace all elements less than 3 with 0
- Find the 3rd largest element (without sorting the whole vector — use `sort` and index)
- Compute the running mean (cumulative mean up to each point)

**2. Matrix operations**

Create a 5×5 matrix filled with the numbers 1–25, filled by row. Then:
- Extract the diagonal (use `diag()`)
- Compute the sum of each row (use `rowSums()`)
- Extract the 3×3 submatrix from rows 2–4, columns 2–4
- Replace all values greater than 15 with NA

**3. Grade analysis extension**

Extend the grade analyzer to also:
- Weight the tests: Test 1 = 20%, Test 2 = 20%, Test 3 = 25%, Test 4 = 35%
- Find students who improved (Test 4 > Test 1)
- Compute the correlation between Test 1 and Test 4 scores

**4. Moving average**

Write `moving_avg(x, k)` that computes the k-period moving average. For `x <- c(1, 2, 3, 4, 5, 6, 7)` and `k = 3`, the result should be `NA NA 2 3 4 5 6`.

---

*Next: Chapter 6 — Strings: processing text in R*
