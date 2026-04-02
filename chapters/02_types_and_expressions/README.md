# Chapter 2: Types and Expressions

*What R stores. How it does arithmetic. Type coercion and what it means.*

---

## The Basic Types

R has five atomic types. Everything is built from these:

```r
42L          # integer   (the L means "store as integer, not double")
3.14         # double    (the default numeric type)
TRUE         # logical
"hello"      # character
2 + 3i       # complex   (rarely needed)
```

Check the type of anything with `class()` or `typeof()`:

```r
class(42L)        # "integer"
class(3.14)       # "numeric"
class(TRUE)       # "logical"
class("hello")    # "character"

typeof(42L)       # "integer"
typeof(3.14)      # "double"
typeof(TRUE)      # "logical"
typeof("hello")   # "character"
```

`class()` is the high-level description R uses for dispatch. `typeof()` is the low-level storage type. For now, use `class()`.

---

## Numerics

Two kinds of numbers: **integer** and **double** (floating point):

```r
x <- 42       # double by default
y <- 42L      # integer (explicit)

class(x)      # "numeric"
class(y)      # "integer"

is.integer(x)  # FALSE
is.integer(y)  # TRUE
is.double(x)   # TRUE
```

Arithmetic works as expected:

```r
5 / 2         # 2.5   (double division)
5 %/% 2       # 2     (integer division)
5 %% 2        # 1     (modulo)
2 ^ 10        # 1024
```

Floating point caveat — always relevant:

```r
0.1 + 0.2 == 0.3    # FALSE — floating point!
0.1 + 0.2            # 0.3 ... but not exactly
```

Use `all.equal()` to compare floats:

```r
all.equal(0.1 + 0.2, 0.3)   # TRUE
```

Special numeric values:

```r
Inf          # infinity
-Inf         # negative infinity
NaN          # Not a Number (0/0)
NA           # missing value (works for any type)
NULL         # absence of a value (different from NA)

1 / 0        # Inf
-1 / 0       # -Inf
0 / 0        # NaN
is.na(NA)    # TRUE
is.null(NULL)# TRUE
is.nan(NaN)  # TRUE
is.infinite(Inf) # TRUE
```

---

## Logical

`TRUE` and `FALSE`. Can be abbreviated `T` and `F` (but avoid it — `T` and `F` can be overwritten).

```r
TRUE & FALSE    # AND → FALSE
TRUE | FALSE    # OR  → TRUE
!TRUE           # NOT → FALSE

TRUE && FALSE   # short-circuit AND (for single values)
TRUE || FALSE   # short-circuit OR
```

`&` and `|` are vectorized — they operate element-by-element.  
`&&` and `||` evaluate only the first element (used in `if` conditions).

Comparisons return logical:

```r
5 > 3          # TRUE
5 < 3          # FALSE
5 == 5         # TRUE
5 != 4         # TRUE
5 >= 5         # TRUE
5 <= 4         # FALSE
```

**Important**: use `==` to compare, not `=` (which assigns).

---

## Characters (Strings)

Strings use single or double quotes:

```r
x <- "Hello"
y <- 'World'
z <- "It's a test"         # single quote inside double quotes
w <- 'She said "hello"'    # double quote inside single quotes
```

Key string functions:

```r
nchar("hello")              # 5 — number of characters
toupper("hello")            # "HELLO"
tolower("HELLO")            # "hello"
paste("Hello", "World")     # "Hello World"
paste0("Hello", "World")    # "HelloWorld"
paste("a", "b", "c", sep = "-")  # "a-b-c"
substr("Hello World", 1, 5) # "Hello"
gsub("l", "L", "hello")    # "heLLo" — replace all
sub("l", "L", "hello")     # "heLlo" — replace first only
grepl("World", "Hello World")  # TRUE — does pattern match?
```

`paste()` is R's main string builder. Use it constantly:

```r
name <- "Alice"
age  <- 30
cat(paste("Name:", name, "Age:", age), "\n")
```

---

## Type Coercion

R automatically converts between types when needed. Understanding this saves hours of debugging.

**Implicit coercion** — happens automatically:

```r
TRUE + TRUE       # 2 (logical → numeric: TRUE=1, FALSE=0)
TRUE + 5          # 6
sum(c(TRUE, FALSE, TRUE, TRUE))  # 3 — count of TRUE values

paste(42)         # "42" — numeric → character
paste(TRUE)       # "TRUE"
```

**Explicit coercion** — you control it:

```r
as.numeric("3.14")   # 3.14
as.integer("42")     # 42
as.character(100)    # "100"
as.logical(0)        # FALSE
as.logical(1)        # TRUE
as.logical("TRUE")   # TRUE
as.numeric("abc")    # NA (with warning — can't convert)
```

The coercion hierarchy (lowest to highest):
```
logical → integer → double → character
```

When types mix, R promotes to the "higher" type:

```r
c(1L, 2.5)     # numeric (integer promoted to double)
c(1, "a")      # "1" "a" (numeric promoted to character)
c(TRUE, 1L)    # 1 1 (logical promoted to integer)
```

---

## A Program: Unit Converter

A proper unit converter using functions and type checking:

```r
# unit_converter.R
# Convert between common units with validation

convert <- function(value, from, to) {
  # Conversion table (all to SI base unit)
  # Length: base = meters
  # Weight: base = kilograms
  # Temperature: handled separately
  
  factors <- list(
    # Length
    m       = 1,
    km      = 1000,
    cm      = 0.01,
    mm      = 0.001,
    inch    = 0.0254,
    foot    = 0.3048,
    yard    = 0.9144,
    mile    = 1609.344,
    
    # Weight
    kg      = 1,
    g       = 0.001,
    lb      = 0.453592,
    oz      = 0.028350,
    
    # Area
    sqm     = 1,
    sqkm    = 1e6,
    sqft    = 0.092903,
    acre    = 4046.86,
    hectare = 10000
  )
  
  # Handle temperature separately
  if (from == "C" && to == "F") return(value * 9/5 + 32)
  if (from == "F" && to == "C") return((value - 32) * 5/9)
  if (from == "C" && to == "K") return(value + 273.15)
  if (from == "K" && to == "C") return(value - 273.15)
  if (from == "F" && to == "K") return((value - 32) * 5/9 + 273.15)
  if (from == "K" && to == "F") return((value - 273.15) * 9/5 + 32)
  if (from == to) return(value)
  
  # Look up conversion factors
  if (!from %in% names(factors)) stop(paste("Unknown unit:", from))
  if (!to   %in% names(factors)) stop(paste("Unknown unit:", to))
  
  # Convert: value → base unit → target unit
  base_value <- value * factors[[from]]
  result     <- base_value / factors[[to]]
  return(result)
}

# Format result nicely
show_conversion <- function(value, from, to) {
  result <- convert(value, from, to)
  cat(sprintf("%g %s = %g %s\n", value, from, result, to))
}

# Run conversions
cat("=== Length ===\n")
show_conversion(1,   "mile", "km")
show_conversion(100, "m",    "foot")
show_conversion(6,   "foot", "m")
show_conversion(5.9, "foot", "cm")

cat("\n=== Weight ===\n")
show_conversion(70,  "kg",   "lb")
show_conversion(185, "lb",   "kg")
show_conversion(1,   "oz",   "g")

cat("\n=== Temperature ===\n")
show_conversion(100,  "C", "F")
show_conversion(32,   "F", "C")
show_conversion(0,    "C", "K")
show_conversion(98.6, "F", "C")

cat("\n=== Area ===\n")
show_conversion(1,  "acre",    "sqm")
show_conversion(1,  "hectare", "sqft")
```

Output:
```
=== Length ===
1 mile = 1.60934 km
100 m = 328.084 foot
6 foot = 1.8288 m
5.9 foot = 179.832 cm

=== Weight ===
70 kg = 154.324 lb
185 lb = 83.9145 kg
1 oz = 28.35 g

=== Temperature ===
100 C = 212 F
32 F = 0 C
0 C = 273.15 K
98.6 F = 37 C

=== Area ===
1 acre = 4046.86 sqm
1 hectare = 107639 sqft
```

New things here:
- `list()` — a named list used as a lookup table (Chapter 9)
- `%in%` — tests if a value is in a vector
- `stop()` — throws an error with a message
- `sprintf()` — C-style formatted output (`%g` = compact number format)
- `!` — logical NOT
- `names()` — gets the names of a named object

---

## sprintf: Formatted Output

`sprintf()` gives you precise control over output:

```r
sprintf("%.2f", 3.14159)     # "3.14"     (2 decimal places)
sprintf("%d items", 42L)     # "42 items"  (integer)
sprintf("%s is %d", "Alice", 30)  # "Alice is 30"
sprintf("%10.3f", 3.14)      # "     3.140" (width 10, 3 decimals)
sprintf("%-10s|", "left")    # "left      |" (left-aligned)
sprintf("%05d", 42)          # "00042" (zero-padded)
```

Format codes:
- `%d` — integer
- `%f` — float (decimal notation)
- `%e` — float (scientific notation)
- `%g` — float (shorter of `%f` / `%e`)
- `%s` — string
- `%i` — integer (same as `%d`)

---

## Exercises

**1. Type inspection**

For each expression, predict what `class()` returns before running it:
```r
class(1)
class(1L)
class(TRUE)
class("1")
class(1 + 2i)
class(NA)
class(NULL)
class(Inf)
class(NaN)
```

**2. Coercion consequences**

What does each evaluate to? Predict, then run:
```r
as.numeric(TRUE)
as.numeric(FALSE)
as.integer(3.9)
as.character(3.14)
as.logical(0)
as.logical(42)
as.numeric("3.14abc")
TRUE + TRUE + TRUE
```

**3. String manipulation**

Write expressions (not a function) that:
- Count the characters in `"supercalifragilistic"`
- Convert `"Hello World"` to all uppercase
- Replace all `"o"` in `"one two three four"` with `"0"`
- Check if `"rain"` appears in `"The rain in Spain"`
- Extract characters 5–10 from `"Hello, World!"`

**4. Extend the converter**

Add speed conversions to `unit_converter.R`:
- `mph` (miles per hour) → base unit `mps` (meters per second): 1 mph = 0.44704 mps
- `kph` (km per hour): 1 kph = 0.27778 mps
- `knot`: 1 knot = 0.514444 mps

Test: convert 100 kph to mph. (Answer: ~62.1 mph)

**5. sprintf formatting**

Format a "receipt" using `sprintf`:
```
Item           Qty    Price    Total
Widget A         3     9.99    29.97
Widget B         1    24.99    24.99
Widget C        10     1.49    14.90
                            --------
                       Total:  69.86
```
Columns should be aligned. Use `sprintf` with width specifiers.

---

*Next: Chapter 3 — Control Flow: if, for, while, and flow of execution*
