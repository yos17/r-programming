# Chapter 1: Getting Started

*Hello, R. What the language is. How to run your first programs.*

---

## Hello, World

Open RStudio. In the Script editor, type:

```r
print("Hello, World")
```

Press `Cmd+Enter` (Mac) or `Ctrl+Enter` (Windows). In the Console you see:

```
[1] "Hello, World"
```

The `[1]` means this is the first element of the output. It will make more sense in Chapter 5. For now, it's just how R labels output.

You can also write it without `print()`:

```r
"Hello, World"
```

R will print values automatically when you evaluate an expression. This makes the Console feel like a calculator.

---

## R as a Calculator

R is a calculator first. Type these in the Console:

```r
2 + 3
10 - 4
3 * 7
15 / 4
15 %/% 4      # integer division
15 %% 4       # remainder (modulo)
2 ^ 8         # exponent
sqrt(144)
abs(-42)
```

Output:
```
[1] 5
[1] 6
[1] 21
[1] 3.75
[1] 3
[1] 3
[1] 256
[1] 12
[1] 42
```

R uses `#` for comments. Everything after `#` on a line is ignored.

---

## Variables

Variables are created with `<-` (the assignment arrow):

```r
x <- 42
y <- 3.14
name <- "Alice"
```

Read `x <- 42` as: "x gets 42".

`=` also works, but `<-` is the R convention. Use `<-` in scripts; `=` is fine inside function arguments.

Print a variable by typing its name:

```r
x
name
```

Output:
```
[1] 42
[1] "Alice"
```

Variable names: letters, digits, `.`, `_`. Must start with a letter or `.`. Case-sensitive: `x` and `X` are different.

```r
my.variable <- 10
my_variable <- 20
.hidden <- 30
```

---

## Your First Script: A Calculator

Create a new file: `File → New File → R Script`. Save it as `calculator.R`.

```r
# calculator.R
# A simple calculator that does unit conversions

# Constants
km_per_mile    <- 1.60934
cm_per_inch    <- 2.54
kg_per_pound   <- 0.453592
celsius_offset <- 32
celsius_scale  <- 5 / 9

# Inputs
miles  <- 26.2        # marathon distance
inches <- 72          # height
pounds <- 185         # weight
fahrenheit <- 98.6    # body temperature

# Conversions
kilometers <- miles * km_per_mile
centimeters <- inches * cm_per_inch
kilograms   <- pounds * kg_per_pound
celsius     <- (fahrenheit - celsius_offset) * celsius_scale

# Output
cat("Marathon distance:", kilometers, "km\n")
cat("Height:", centimeters, "cm\n")
cat("Weight:", kilograms, "kg\n")
cat("Body temperature:", celsius, "°C\n")
```

Run it with `Ctrl+Shift+Enter`. Output:

```
Marathon distance: 42.1649 km
Height: 182.88 cm
Weight: 83.91452 kg
Body temperature: 37 °C
```

`cat()` concatenates and prints. `\n` is a newline. Unlike `print()`, `cat()` doesn't add `[1]` and lets you control formatting.

---

## Functions

You've already used functions: `print()`, `cat()`, `sqrt()`, `abs()`.

A function call: `name(argument1, argument2, ...)`. The function takes inputs and returns a result.

```r
round(3.14159, digits = 2)   # round to 2 decimal places
ceiling(3.2)                  # round up
floor(3.8)                    # round down
max(10, 20, 5)
min(10, 20, 5)
```

Output:
```
[1] 3.14
[1] 4
[1] 3
[1] 20
[1] 5
```

Arguments can be **positional** or **named**:

```r
round(3.14159, 2)              # positional
round(3.14159, digits = 2)     # named (same result)
round(digits = 2, x = 3.14159) # named, any order
```

---

## Getting Help

For any function, type `?` before it:

```r
?round
?cat
?sqrt
```

The Help panel (bottom-right) shows documentation. Every R function has a help page. Use it constantly.

Also: `example(round)` runs the examples from the help page directly in the Console.

---

## The Script vs the Console

| Console | Script |
|---------|--------|
| Interactive — run one line | Permanent — saved code |
| `2 + 2` → immediate result | Full programs |
| Exploration | Production |

Rule: **explore in the Console, write keepers in the Script**.

---

## Building a Word Counter

Now something more interesting. Create `wordcount.R`:

```r
# wordcount.R
# Count words and characters in a text

text <- "The quick brown fox jumps over the lazy dog"

# Count words
words <- strsplit(text, " ")[[1]]
word_count <- length(words)

# Count characters (excluding spaces)
char_count <- nchar(gsub(" ", "", text))

# Count characters (including spaces)  
total_chars <- nchar(text)

# Find unique words
unique_words <- unique(tolower(words))
unique_count <- length(unique_words)

# Most frequent letter
all_letters <- strsplit(tolower(gsub("[^a-z]", "", text)), "")[[1]]
letter_freq  <- table(all_letters)
top_letter   <- names(which.max(letter_freq))

# Report
cat("Text:", text, "\n\n")
cat("Words:           ", word_count, "\n")
cat("Unique words:    ", unique_count, "\n")
cat("Characters:      ", total_chars, "(with spaces)\n")
cat("Characters:      ", char_count, "(without spaces)\n")
cat("Most used letter:", top_letter,
    "(", letter_freq[top_letter], "times )\n")
```

Run it. Output:

```
Text: The quick brown fox jumps over the lazy dog

Words:            9
Unique words:     8
Characters:       43 (with spaces)
Characters:       35 (without spaces)
Most used letter: o ( 4 times )
```

This uses functions you haven't fully learned yet — `strsplit`, `table`, `which.max`. That's fine. Read the code, see what it does, look up anything unfamiliar with `?`.

---

## What Just Happened

Line by line:

```r
words <- strsplit(text, " ")[[1]]
```
`strsplit` splits a string at a separator. It returns a list (Chapter 9), so `[[1]]` extracts the first element — the vector of words.

```r
word_count <- length(words)
```
`length()` returns how many elements are in a vector.

```r
unique_words <- unique(tolower(words))
```
`tolower()` converts to lowercase. `unique()` removes duplicates.

```r
letter_freq <- table(all_letters)
```
`table()` counts how often each value occurs. The result is a named vector of counts.

```r
top_letter <- names(which.max(letter_freq))
```
`which.max()` finds the index of the maximum. `names()` gets the name at that index.

---

## The Environment

After running your script, look at the **Environment** panel (top-right). You'll see all the variables you created: `text`, `words`, `word_count`, etc., with their values.

This is your workspace. Everything you've assigned lives here until you clear it or restart R.

To remove a variable:
```r
rm(x)
```

To clear everything:
```r
rm(list = ls())
```

---

## Summary

| Concept | Example |
|---------|---------|
| Assignment | `x <- 42` |
| Print | `print(x)` or just `x` |
| Output | `cat("Value:", x, "\n")` |
| Arithmetic | `+`, `-`, `*`, `/`, `^`, `%%`, `%/%` |
| Function call | `round(3.14, digits = 2)` |
| Comment | `# this is ignored` |
| Help | `?function_name` |

---

## Exercises

**1. Calculator script**

Write a script that calculates and prints:
- The area of a circle with radius 7 (π × r²) — use `pi` for π
- The hypotenuse of a right triangle with sides 3 and 4 (`sqrt(a^2 + b^2)`)
- The compound interest after 5 years: principal 1000, rate 7% (`P * (1 + r)^n`)

**2. Modify the word counter**

Change the text to something longer (a sentence from a book, or something you write). Run it. What changes?

**3. Explore `round()`**

Look up `?round`. What does `round(2.5)` return? What about `round(3.5)`? R uses "banker's rounding" — find out what that means.

**4. Convert the converter**

Modify `calculator.R` to also convert:
- 100 meters to feet (1 meter = 3.28084 feet)
- 1 liter to US gallons (1 liter = 0.264172 gallons)

---

*Next: Chapter 2 — Types and Expressions*
