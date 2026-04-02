# Chapter 1: Getting Started

*Hello, R. Running programs from the command line. Your first analyze.R.*

---

## The Program We're Building

By Chapter 12 you'll have a tool that does this:

```
Rscript analyze.R data.csv --group dept --stat mean,sd --filter "salary>50000" --output report.txt
```

It reads any CSV, computes statistics, filters rows, groups data, and writes a report. 150 lines of R. Genuinely useful.

We start smaller. Much smaller.

---

## Hello, World

Open a terminal. Type:

```
Rscript --version
```

If R is installed, you'll see something like `R scripting front-end version 4.3.1`. Good.

Create a file called `analyze.R`:

```r
cat("Hello\n")
```

Run it:

```
Rscript analyze.R
```

Output:
```
Hello
```

That's it. That's the whole program. We'll grow it from here.

---

## Reading a Command-Line Argument

The tool needs to know which file to analyze. Let's read the first argument:

```r
args <- commandArgs(trailingOnly = TRUE)
cat("Hello\n")
cat("File:", args[1], "\n")
```

Run it:

```
Rscript analyze.R data.csv
```

Output:
```
Hello
File: data.csv
```

`commandArgs(trailingOnly = TRUE)` returns a character vector of everything you typed after the script name. `args[1]` is the first one.

What if they forgot the filename?

```r
args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv>\n")
  quit(status = 1)
}

cat("Hello\n")
cat("File:", args[1], "\n")
```

`quit(status = 1)` exits with error code 1 — the Unix convention for "something went wrong".

---

## R as a Calculator

Before we go further, you should be comfortable with the basics. Open RStudio and type these in the Console:

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

The `[1]` means this is the first element of the output. R has no scalars — every value is a vector, even a single number. The index marker will make more sense in Chapter 5.

---

## Variables

```r
x <- 42
y <- 3.14
name <- "Alice"
```

Read `x <- 42` as: "x gets 42". `<-` is the R assignment operator. `=` also works, but `<-` is the convention.

Print a variable by typing its name, or with `cat()`:

```r
x
name
cat("x is", x, "\n")
```

Variable names: letters, digits, `.`, `_`. Must start with a letter. Case-sensitive: `x` and `X` are different.

---

## Output: cat() vs print()

Two ways to print:

```r
print("Hello, World")    # [1] "Hello, World"
cat("Hello, World\n")    # Hello, World
```

`print()` shows R's internal representation (with quotes, type hints). `cat()` outputs raw text. For a command-line tool, we want `cat()` — it lets us control exactly what the user sees.

`\n` is a newline. Without it, the next output lands on the same line.

---

## Functions

You've already used functions: `cat()`, `sqrt()`, `abs()`. A function call: `name(argument1, argument2, ...)`.

```r
round(3.14159, digits = 2)   # 3.14
ceiling(3.2)                  # 4
floor(3.8)                    # 3
max(10, 20, 5)                # 20
min(10, 20, 5)                # 5
```

Arguments can be **positional** or **named**:

```r
round(3.14159, 2)              # positional
round(3.14159, digits = 2)     # named (same result)
round(digits = 2, x = 3.14159) # named, any order
```

For help on any function: `?round`

---

## analyze.R: Chapter 1 Version

Here's where we leave off this chapter. The program reads its filename argument and prints it:

```r
# analyze.R — Chapter 1
# Usage: Rscript analyze.R <file.csv>

args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv>\n")
  quit(status = 1)
}

filename <- args[1]
cat("analyze.R\n")
cat("File:", filename, "\n")
```

Run it:

```
Rscript analyze.R mydata.csv
```

Output:
```
analyze.R
File: mydata.csv
```

Not useful yet. But it runs, it takes input, and it has a clean error message. That's a real program.

---

## The Script vs the Console

| Console | Script |
|---------|--------|
| Interactive — run one line | Permanent — saved code |
| `2 + 2` → immediate result | Full programs |
| Exploration | Production |

Rule: **explore in the Console, write keepers in the Script**.

---

## Getting Help

```r
?round
?cat
?commandArgs
```

The Help panel (bottom-right in RStudio) shows documentation. `example(round)` runs the examples from the help page directly in the Console.

---

## Summary

| Concept | Example |
|---------|---------|
| Assignment | `x <- 42` |
| Print | `cat("Value:", x, "\n")` |
| Arithmetic | `+`, `-`, `*`, `/`, `^`, `%%`, `%/%` |
| Function call | `round(3.14, digits = 2)` |
| Comment | `# this is ignored` |
| Help | `?function_name` |
| Command args | `commandArgs(trailingOnly = TRUE)` |
| Exit | `quit(status = 1)` |

---

## Exercises

**1. Calculator**

Write a script `calc.R` that prints:
- The area of a circle with radius 7 (`pi * r^2` — use `pi`)
- The hypotenuse of a right triangle with sides 3 and 4 (`sqrt(a^2 + b^2)`)
- Compound interest after 5 years: principal 1000, rate 7% (`P * (1 + r)^n`)

**2. Explore `round()`**

Look up `?round`. What does `round(2.5)` return? What about `round(3.5)`? R uses "banker's rounding" — look that up.

**3. Message formatting**

Modify `analyze.R` so it prints the date and time along with the filename:

```
analyze.R
File: mydata.csv
Date: 2026-04-02 22:06:00
```

Hint: `Sys.time()` returns the current time. `format(Sys.time(), "%Y-%m-%d %H:%M:%S")` formats it as a string.

**4. Two arguments**

Extend `analyze.R` to accept a second optional argument `--verbose`. If present, print extra information. If not, stay quiet. Hint: `"--verbose" %in% args` tests if a string is in a vector.

**5. The growing program (do this one)**

Add to `analyze.R`: if the file doesn't exist, print an error message and quit. Use `file.exists(filename)`. This will be useful in Chapter 2 when we actually read the file.

```
Rscript analyze.R missing.csv
Error: file not found: missing.csv
```

---

*Next: Chapter 2 — Types and Expressions: parse the file, convert units*
