# Chapter 2: Types and Expressions

*What R stores. How it does arithmetic. Parsing real data.*

---

## Where We Left Off

End of Chapter 1, `analyze.R` looks like this:

```r
# analyze.R — Chapter 1
args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv>\n")
  quit(status = 1)
}

filename <- args[1]

if (!file.exists(filename)) {
  cat("Error: file not found:", filename, "\n")
  quit(status = 1)
}

cat("analyze.R\n")
cat("File:", filename, "\n")
```

It reads a filename. It doesn't read the file. We can't do anything useful until we understand R's types — what the data *is* when it comes in.

---

## The Problem

CSV files are text. Every value comes in as a string. The number `50000` arrives as `"50000"`. Before we can compute with it, we need to parse it.

This chapter is about what R can store and how to convert between types.

---

## The Basic Types

R has five atomic types:

```r
42L          # integer   (L = "store as integer")
3.14         # double    (the default numeric type)
TRUE         # logical
"hello"      # character
2 + 3i       # complex   (rarely needed)
```

Check the type of anything with `class()`:

```r
class(42L)        # "integer"
class(3.14)       # "numeric"
class(TRUE)       # "logical"
class("hello")    # "character"
```

---

## Numerics

Two kinds of numbers: **integer** and **double** (floating point):

```r
x <- 42       # double by default
y <- 42L      # integer (explicit)

class(x)      # "numeric"
class(y)      # "integer"
```

Arithmetic:

```r
5 / 2         # 2.5   (double division)
5 %/% 2       # 2     (integer division)
5 %% 2        # 1     (modulo)
2 ^ 10        # 1024
```

The floating point trap — every R programmer hits this eventually:

```r
0.1 + 0.2 == 0.3    # FALSE
0.1 + 0.2            # 0.3 ... but not exactly
```

Computers store numbers in binary. `0.1` has no exact binary representation. Use `all.equal()` to compare floats:

```r
all.equal(0.1 + 0.2, 0.3)   # TRUE
```

Special values:

```r
Inf          # infinity
-Inf         # negative infinity  
NaN          # Not a Number (0/0)
NA           # missing value — works for any type
NULL         # absence of a value (different from NA)

1 / 0        # Inf
0 / 0        # NaN
is.na(NA)    # TRUE
is.na(NaN)   # TRUE  (NaN is also NA)
is.null(NULL)# TRUE
```

`NA` means "we have a slot but don't know the value." `NULL` means "there's no slot at all."

---

## Logical

`TRUE` and `FALSE`. The result of comparisons:

```r
5 > 3          # TRUE
5 < 3          # FALSE
5 == 5         # TRUE  (== not =)
5 != 4         # TRUE
5 >= 5         # TRUE
```

Logical operators:

```r
TRUE & FALSE    # AND → FALSE
TRUE | FALSE    # OR  → TRUE
!TRUE           # NOT → FALSE
```

`TRUE` and `FALSE` are also `1` and `0` in arithmetic:

```r
TRUE + TRUE       # 2
sum(c(TRUE, FALSE, TRUE, TRUE))  # 3
```

This is useful: `sum(x > 0)` counts how many elements are positive.

---

## Characters (Strings)

Strings use single or double quotes:

```r
x <- "Hello"
y <- 'World'
z <- "It's fine"          # single quote inside double quotes
w <- 'She said "hello"'   # double quote inside single quotes
```

Key string functions:

```r
nchar("hello")              # 5
toupper("hello")            # "HELLO"
tolower("HELLO")            # "hello"
paste("Hello", "World")     # "Hello World"
paste0("Hello", "World")    # "HelloWorld"
paste("a", "b", "c", sep = "-")  # "a-b-c"
substr("Hello World", 1, 5) # "Hello"
gsub("l", "L", "hello")    # "heLLo" — replace all
grepl("World", "Hello World")  # TRUE — does pattern match?
```

---

## Type Coercion

R converts between types automatically. Understanding this prevents bugs.

**Implicit coercion:**

```r
TRUE + 5          # 6  (logical → numeric)
paste(42)         # "42"  (numeric → character)
```

**Explicit coercion:**

```r
as.numeric("3.14")   # 3.14
as.integer("42")     # 42
as.character(100)    # "100"
as.logical(0)        # FALSE
as.logical(1)        # TRUE
as.numeric("abc")    # NA  (with warning)
```

The coercion hierarchy:
```
logical → integer → double → character
```

When types mix, R promotes to the "higher" type:

```r
c(1L, 2.5)     # numeric (integer promoted to double)
c(1, "a")      # "1" "a" (numeric promoted to character)
```

---

## sprintf: Formatted Output

`sprintf()` gives precise control over output. You'll use it constantly:

```r
sprintf("%.2f", 3.14159)          # "3.14"
sprintf("%d items", 42L)          # "42 items"
sprintf("%s is %d", "Alice", 30)  # "Alice is 30"
sprintf("%10.3f", 3.14)           # "     3.140" (width 10, 3 decimals)
sprintf("%-10s|", "left")         # "left      |" (left-aligned)
sprintf("%05d", 42)               # "00042" (zero-padded)
```

Format codes:
- `%d` — integer
- `%f` — float (decimal)
- `%g` — float (shorter of `%f`/`%e`)
- `%s` — string

---

## Parsing Real Data

Here's the problem `analyze.R` will face. You read a CSV line like:

```
"Alice",50000,Engineering,TRUE
```

Everything is a string: `"Alice"`, `"50000"`, `"Engineering"`, `"TRUE"`. Before computing, you parse:

```r
name   <- "Alice"
salary <- as.numeric("50000")    # 50000
dept   <- "Engineering"
active <- as.logical("TRUE")     # TRUE

# Now you can compute:
cat(sprintf("%s earns $%.0f\n", name, salary))
```

When parsing fails, you get `NA`:

```r
as.numeric("not a number")   # NA (with warning)
```

Always check for NAs after parsing:

```r
salary_str <- "not a number"
salary <- suppressWarnings(as.numeric(salary_str))
if (is.na(salary)) {
  cat("Warning: could not parse salary:", salary_str, "\n")
}
```

`suppressWarnings()` silences the warning when you're handling it yourself.

---

## Unit Conversion

Here's where types and arithmetic come together. A small unit converter:

```r
# Conversion factors (relative to SI base unit)
factors <- c(
  m = 1, km = 1000, cm = 0.01, inch = 0.0254,
  foot = 0.3048, mile = 1609.344,
  kg = 1, g = 0.001, lb = 0.453592
)

convert <- function(value, from, to) {
  if (!from %in% names(factors)) stop(paste("Unknown unit:", from))
  if (!to   %in% names(factors)) stop(paste("Unknown unit:", to))
  value * factors[[from]] / factors[[to]]
}

cat(sprintf("1 mile = %.3f km\n", convert(1, "mile", "km")))
cat(sprintf("185 lb = %.1f kg\n", convert(185, "lb", "kg")))
cat(sprintf("6 foot = %.1f cm\n", convert(6, "foot", "cm") * 100))
```

Output:
```
1 mile = 1.609 km
185 lb = 83.9 kg
6 foot = 182.9 cm
```

New here: `stop()` throws an error with a message. `%in%` tests if a value is in a vector. `names()` gets names of a named vector.

---

## analyze.R: Chapter 2 Version

The new problem: we want to echo back some info about the file. But first we need to read it.

We'll properly read the file in Chapter 7. For now, add type parsing and unit conversion as a standalone feature:

```r
# analyze.R — Chapter 2
# Usage: Rscript analyze.R <file.csv> [--units from:to:value]

args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv>\n")
  quit(status = 1)
}

filename <- args[1]

if (!file.exists(filename)) {
  cat("Error: file not found:", filename, "\n")
  quit(status = 1)
}

cat("analyze.R\n")
cat("File:", filename, "\n")
cat("Date:", format(Sys.time(), "%Y-%m-%d %H:%M:%S"), "\n")

# Unit conversion sub-command
unit_arg <- grep("^--units=", args, value = TRUE)
if (length(unit_arg) > 0) {
  parts <- strsplit(sub("^--units=", "", unit_arg), ":")[[1]]
  if (length(parts) == 3) {
    from  <- parts[1]
    to    <- parts[2]
    value <- as.numeric(parts[3])
    
    factors <- c(
      m = 1, km = 1000, cm = 0.01, inch = 0.0254,
      foot = 0.3048, mile = 1609.344,
      kg = 1, g = 0.001, lb = 0.453592
    )
    
    if (from %in% names(factors) && to %in% names(factors) && !is.na(value)) {
      result <- value * factors[[from]] / factors[[to]]
      cat(sprintf("%.4g %s = %.4g %s\n", value, from, result, to))
    } else {
      cat("Error: invalid unit conversion\n")
    }
  }
}
```

Run it:

```
Rscript analyze.R data.csv --units=mile:km:26.2
```

Output:
```
analyze.R
File: data.csv
Date: 2026-04-02 22:06:00
26.2 mile = 42.16 km
```

---

## Exercises

**1. Type inspection**

For each expression, predict what `class()` returns before running:
```r
class(1)
class(1L)
class(TRUE)
class("1")
class(NA)
class(NULL)
class(Inf)
class(NaN)
```

**2. Coercion consequences**

What does each evaluate to? Predict, then run:
```r
as.numeric(TRUE)
as.integer(3.9)
as.logical(0)
as.logical(42)
as.numeric("3.14abc")
TRUE + TRUE + TRUE
```

**3. String manipulation**

Write expressions that:
- Count the characters in `"supercalifragilistic"`
- Convert `"Hello World"` to uppercase
- Replace all `"o"` in `"one two three four"` with `"0"`
- Check if `"rain"` appears in `"The rain in Spain"`
- Extract characters 5–10 from `"Hello, World!"`

**4. sprintf formatting**

Format a "receipt" using `sprintf`:
```
Item           Qty    Price    Total
Widget A         3     9.99    29.97
Widget B         1    24.99    24.99
Widget C        10     1.49    14.90
                            --------
                       Total:  69.86
```
Columns should be aligned.

**5. The growing program (do this one)**

Extend the unit conversion to handle temperature (Celsius/Fahrenheit/Kelvin), which can't use simple multiplication factors. Use `if/else if` chains for temperature, factors for the rest. Test:

```
Rscript analyze.R data.csv --units=C:F:100
100 C = 212 F
```

This type-aware parsing — checking what kind of conversion is needed before applying it — is exactly the pattern we'll use in Chapter 3 when we start filtering data.

---

*Next: Chapter 3 — Control Flow: add filtering to analyze.R*

---

## Solutions

### Exercise 1 — Type inspection

```r
class(1)      # "numeric"   — undecorated numbers are doubles
class(1L)     # "integer"   — the L suffix forces integer storage
class(TRUE)   # "logical"
class("1")    # "character"
class(NA)     # "logical"   — NA without a type suffix is logical NA
class(NULL)   # "NULL"
class(Inf)    # "numeric"   — Inf is a special double
class(NaN)    # "numeric"   — NaN is also a special double
```

### Exercise 2 — Coercion consequences

```r
as.numeric(TRUE)        # 1     — TRUE maps to 1
as.integer(3.9)         # 3     — truncates toward zero, does NOT round
as.logical(0)           # FALSE — only 0 is FALSE; everything else is TRUE
as.logical(42)          # TRUE
as.numeric("3.14abc")   # NA (with warning) — non-numeric string → NA
TRUE + TRUE + TRUE      # 3     — logical is 0/1 in arithmetic
```

### Exercise 3 — String manipulation

```r
# Count characters
nchar("supercalifragilistic")          # 20

# Uppercase
toupper("Hello World")                  # "HELLO WORLD"

# Replace all "o" with "0"
gsub("o", "0", "one two three four")   # "0ne tw0 three f0ur"

# Check if "rain" appears in the string
grepl("rain", "The rain in Spain")      # TRUE

# Extract characters 5–10 from "Hello, World!"
substr("Hello, World!", 5, 10)          # "o, Wor"
```

### Exercise 4 — sprintf formatting

```r
# Format a receipt with aligned columns

items <- data.frame(
  name  = c("Widget A", "Widget B", "Widget C"),
  qty   = c(3, 1, 10),
  price = c(9.99, 24.99, 1.49)
)
items$total <- items$qty * items$price

cat(sprintf("%-12s  %4s  %7s  %7s\n", "Item", "Qty", "Price", "Total"))
for (i in seq_len(nrow(items))) {
  cat(sprintf("%-12s  %4d  %7.2f  %7.2f\n",
              items$name[i], items$qty[i], items$price[i], items$total[i]))
}
cat(sprintf("%s\n", strrep("-", 37)))
cat(sprintf("%-12s  %4s  %7s  %7.2f\n", "", "", "Total:", sum(items$total)))

# Item          Qty    Price    Total
# Widget A        3     9.99    29.97
# Widget B        1    24.99    24.99
# Widget C       10     1.49    14.90
# -------------------------------------
#                        Total:  69.86
```

### Exercise 5 — The growing program (temperature conversion)

Add temperature handling to `analyze.R`. Temperature can't use simple multiplication factors because Celsius, Fahrenheit, and Kelvin all have different zero points.

```r
# analyze.R — Chapter 2, Exercise 5
# Adds temperature conversion to the --units flag

args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv> [--units=from:to:value]\n")
  quit(status = 1)
}

filename <- args[1]

if (!file.exists(filename)) {
  cat("Error: file not found:", filename, "\n")
  quit(status = 1)
}

cat("analyze.R\n")
cat("File:", filename, "\n")
cat("Date:", format(Sys.time(), "%Y-%m-%d %H:%M:%S"), "\n")

# --- Unit conversion (length/mass) ---
factors <- c(
  m = 1, km = 1000, cm = 0.01, inch = 0.0254,
  foot = 0.3048, mile = 1609.344,
  kg = 1, g = 0.001, lb = 0.453592
)

# Temperature units that need special-case handling
temp_units <- c("C", "F", "K")

convert_temp <- function(value, from, to) {
  # Convert 'from' unit → Celsius first, then Celsius → 'to' unit
  celsius <- if (from == "C") value
             else if (from == "F") (value - 32) * 5 / 9
             else if (from == "K") value - 273.15
             else stop(paste("Unknown temperature unit:", from))

  if (to == "C") celsius
  else if (to == "F") celsius * 9 / 5 + 32
  else if (to == "K") celsius + 273.15
  else stop(paste("Unknown temperature unit:", to))
}

unit_arg <- grep("^--units=", args, value = TRUE)
if (length(unit_arg) > 0) {
  parts <- strsplit(sub("^--units=", "", unit_arg), ":")[[1]]
  if (length(parts) == 3) {
    from  <- parts[1]
    to    <- parts[2]
    value <- as.numeric(parts[3])

    if (is.na(value)) {
      cat("Error: invalid numeric value\n")
    } else if (from %in% temp_units || to %in% temp_units) {
      # Temperature path
      if (!(from %in% temp_units && to %in% temp_units)) {
        cat("Error: cannot mix temperature and non-temperature units\n")
      } else {
        result <- convert_temp(value, from, to)
        cat(sprintf("%.4g %s = %.4g %s\n", value, from, result, to))
      }
    } else if (from %in% names(factors) && to %in% names(factors)) {
      # Standard multiplication path
      result <- value * factors[[from]] / factors[[to]]
      cat(sprintf("%.4g %s = %.4g %s\n", value, from, result, to))
    } else {
      cat("Error: unknown unit(s):", from, "or", to, "\n")
    }
  }
}

# Rscript analyze.R data.csv --units=C:F:100
# 100 C = 212 F

# Rscript analyze.R data.csv --units=F:C:32
# 32 F = 0 C

# Rscript analyze.R data.csv --units=K:C:373.15
# 373.15 K = 100 C
```
