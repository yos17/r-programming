# Chapter 11: Packages and OOP

*Installing and using packages. S3 classes. Building a mini data package.*

---

## Packages

R's strength is its ecosystem. CRAN has 20,000+ packages. You need to know how to find, install, and use them.

```r
# Install a package (done once)
install.packages("ggplot2")
install.packages(c("dplyr", "tidyr", "readr"))

# Load a package (done every session)
library(ggplot2)
library(dplyr)

# Check if installed
"ggplot2" %in% installed.packages()[, "Package"]

# See what's in a package
help(package = "ggplot2")
ls("package:ggplot2")
```

**Namespace access** — use a function from a package without loading the whole thing:

```r
dplyr::filter(df, age > 30)   # use filter() from dplyr
```

---

## Essential Packages to Know

**Data manipulation:**
- `dplyr` — `filter`, `select`, `mutate`, `group_by`, `summarise`
- `tidyr` — `pivot_longer`, `pivot_wider`, `separate`, `unite`
- `data.table` — fast data manipulation for large data

**Reading data:**
- `readr` — fast CSV reading
- `readxl` — Excel files
- `jsonlite` — JSON

**Visualization:**
- `ggplot2` — the grammar of graphics
- `plotly` — interactive plots

**Statistics:**
- `stats` (built-in) — regression, tests, distributions
- `lme4` — mixed models
- `survival` — survival analysis

**Dates/Times:**
- `lubridate` — painless date handling

---

## S3: R's Main OOP System

R has several OOP systems. S3 is the most common — it's simple, flexible, and used throughout base R.

S3 is based on **generic functions** and **methods**. A generic function like `print()` looks at the class of its argument and dispatches to the right method.

**Create an S3 class:**

```r
# Constructor function
new_dog <- function(name, breed, age) {
  structure(
    list(name = name, breed = breed, age = age),
    class = "Dog"
  )
}

# Or: add class to an existing list
new_dog <- function(name, breed, age) {
  obj <- list(name = name, breed = breed, age = age)
  class(obj) <- "Dog"
  obj
}
```

**Methods:** define `generic.ClassName`:

```r
# print method
print.Dog <- function(x, ...) {
  cat(sprintf("Dog: %s (%s), age %d\n", x$name, x$breed, x$age))
  invisible(x)
}

# summary method
summary.Dog <- function(object, ...) {
  cat(sprintf("Name: %s\nBreed: %s\nAge: %d years (%d in dog years)\n",
              object$name, object$breed, object$age, object$age * 7))
}

# Custom generic
speak <- function(x, ...) UseMethod("speak")
speak.Dog <- function(x, ...) cat(x$name, "says: Woof!\n")
speak.Cat <- function(x, ...) cat(x$name, "says: Meow!\n")
speak.default <- function(x, ...) cat("...\n")

# Arithmetic operator override
"+.Dog" <- function(a, b) {
  # "Adding" two dogs = new litter name idea
  new_dog(paste(a$name, "&", b$name), 
          paste(a$breed, "/", b$breed), 0)
}
```

**Use it:**

```r
rex  <- new_dog("Rex",  "German Shepherd", 5)
luna <- new_dog("Luna", "Labrador",         3)

print(rex)       # calls print.Dog
summary(rex)     # calls summary.Dog
speak(rex)       # "Rex says: Woof!"

is(rex, "Dog")   # TRUE
inherits(rex, "Dog")  # TRUE
class(rex)       # "Dog"
```

---

## S3 Inheritance

```r
new_guide_dog <- function(name, breed, age, owner) {
  obj <- new_dog(name, breed, age)   # base constructor
  obj$owner <- owner
  class(obj) <- c("GuideDog", "Dog")  # inherits from Dog
  obj
}

# Override only what's different
print.GuideDog <- function(x, ...) {
  NextMethod()   # calls print.Dog first
  cat(sprintf("  Guide dog for: %s\n", x$owner))
  invisible(x)
}

speak.GuideDog <- function(x, ...) {
  cat(x$name, "says: Quiet woof. I'm working.\n")
}

buddy <- new_guide_dog("Buddy", "Golden Retriever", 4, "John")
print(buddy)    # print.GuideDog → NextMethod() → print.Dog
speak(buddy)    # speak.GuideDog
```

---

## Program: A Mini Data Package

Build a complete S3 class for a data analysis object:

```r
# dataset_class.R
# A Dataset S3 class with methods

# ---- Constructor ----
new_dataset <- function(data, name = "Unnamed", description = "") {
  if (!is.data.frame(data)) stop("data must be a data.frame")
  
  structure(
    list(
      data        = data,
      name        = name,
      description = description,
      created     = Sys.time(),
      n_rows      = nrow(data),
      n_cols      = ncol(data)
    ),
    class = "Dataset"
  )
}

# ---- print ----
print.Dataset <- function(x, ...) {
  cat(sprintf("Dataset: %s\n", x$name))
  cat(sprintf("  %d rows × %d columns\n", x$n_rows, x$n_cols))
  if (nchar(x$description) > 0)
    cat(sprintf("  %s\n", x$description))
  cat(sprintf("  Created: %s\n", format(x$created, "%Y-%m-%d %H:%M")))
  invisible(x)
}

# ---- summary ----
summary.Dataset <- function(object, ...) {
  cat(sprintf("=== %s ===\n", object$name))
  cat(sprintf("Dimensions: %d × %d\n\n", object$n_rows, object$n_cols))
  
  df <- object$data
  for (col in names(df)) {
    x <- df[[col]]
    if (is.numeric(x)) {
      cat(sprintf("%-15s [numeric] : min=%.2f, mean=%.2f, max=%.2f, NAs=%d\n",
                  col, min(x,na.rm=T), mean(x,na.rm=T), max(x,na.rm=T), sum(is.na(x))))
    } else if (is.character(x) || is.factor(x)) {
      n_unique <- length(unique(x))
      cat(sprintf("%-15s [character]: %d unique values, NAs=%d\n",
                  col, n_unique, sum(is.na(x))))
    } else if (is.logical(x)) {
      cat(sprintf("%-15s [logical] : TRUE=%d (%.1f%%), FALSE=%d\n",
                  col, sum(x,na.rm=T), 100*mean(x,na.rm=T), sum(!x,na.rm=T)))
    }
  }
  invisible(object)
}

# ---- Subsetting ([ operator) ----
"[.Dataset" <- function(x, rows, cols) {
  sub_data <- x$data[rows, cols, drop = FALSE]
  new_dataset(sub_data, 
              name        = paste(x$name, "[subset]"),
              description = x$description)
}

# ---- Filter method ----
filter_dataset <- function(x, ...) UseMethod("filter_dataset")
filter_dataset.Dataset <- function(x, condition) {
  sub_data <- x$data[eval(substitute(condition), x$data), ]
  new_dataset(sub_data, 
              name = paste(x$name, "[filtered]"),
              description = x$description)
}

# ---- Save / load ----
save_dataset <- function(x, ...) UseMethod("save_dataset")
save_dataset.Dataset <- function(x, path, ...) {
  saveRDS(x, path)
  cat(sprintf("Saved %s to %s\n", x$name, path))
  invisible(x)
}

load_dataset <- function(path) {
  obj <- readRDS(path)
  if (!inherits(obj, "Dataset")) stop("File does not contain a Dataset object")
  cat(sprintf("Loaded: %s\n", obj$name))
  obj
}

# ---- Test it ----
set.seed(42)
df <- data.frame(
  age    = sample(20:65, 100, replace = TRUE),
  salary = round(rnorm(100, 60000, 15000), -3),
  dept   = sample(c("Eng","Sales","HR","Finance"), 100, replace = TRUE),
  active = sample(c(TRUE,FALSE), 100, prob=c(.9,.1), replace=TRUE)
)

ds <- new_dataset(df, "HR Data", "Employee records Q1 2026")
print(ds)
summary(ds)

# Subset
ds_eng <- filter_dataset(ds, dept == "Eng")
print(ds_eng)
```

---

## Exercises

**1. Extend the Dog class**

Add:
- A `feed(x)` generic that increases age health
- A `compare(a, b)` method that compares two dogs by age
- A `format.Dog` method that lets `paste(rex)` work

**2. Matrix class**

Create a `Matrix2x2` class for 2×2 matrices with:
- `print`: shows the matrix nicely
- `det`: determinant
- `inv`: inverse
- `+`, `*` (`%.%`): addition and multiplication
- `solve.Matrix2x2`: solve Ax = b

**3. Write a package**

Create a minimal R package:
```bash
# In RStudio: File → New Project → New Directory → R Package
```
Add one or two functions from this course, document them with `#'` roxygen comments, and `Build → Install and Restart`.

---

*Next: Chapter 12 — Real Tools: a complete analysis report*
