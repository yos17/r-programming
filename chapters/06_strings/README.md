# Chapter 6: Strings

*Text processing in R. Regular expressions. Parsing string columns.*

---

## Where We Left Off

`analyze.R` can compute statistics on numeric columns. It still treats string columns as opaque — `compute_stats_char()` just counts unique values.

Two things string columns need:
1. Parsing: extract structured data from freeform text (dates embedded in strings, codes inside names)
2. Filtering: `--filter=dept==Engineering` matches a string; `--filter=name~^Alice` should match a regex

Both require understanding how strings work in R.

---

## String Basics

Strings in R are character vectors. Every string function is vectorized.

```r
x <- c("Hello", "World", "R Programming")

nchar(x)          # 5 5 13
toupper(x)        # "HELLO" "WORLD" "R PROGRAMMING"
tolower(x)        # "hello" "world" "r programming"
trimws("  hello  ")             # "hello"
trimws("  hello  ", "left")     # "hello  "
trimws("  hello  ", "right")    # "  hello"
```

---

## Combining and Splitting

```r
paste("Hello", "World")                   # "Hello World"
paste0("Hello", "World")                  # "HelloWorld"
paste(c("a","b","c"), collapse = "-")     # "a-b-c"
paste("item", 1:5, sep = "_")             # "item_1" ... "item_5"

strsplit("hello world foo", " ")          # list: c("hello","world","foo")
strsplit(c("a-b", "c-d-e"), "-")          # list of two vectors
```

`strsplit` returns a list (each string can split into a different number of pieces). Use `[[1]]` to get the first result:

```r
parts <- strsplit("2026-03-15", "-")[[1]]
parts   # "2026" "03" "15"
```

---

## Substrings

```r
x <- "Hello, World!"

substr(x, 1, 5)        # "Hello"
substr(x, 8, 12)       # "World"
substring(x, 8)        # "World!" — from position 8 to end
```

---

## Finding Patterns

```r
x <- c("apple", "banana", "cherry", "apricot", "grape")

grep("ap", x)                    # 1 4 — indices of matches
grep("ap", x, value = TRUE)      # "apple" "apricot"
grepl("ap", x)                   # TRUE FALSE FALSE TRUE FALSE — logical
```

---

## Replacing

```r
x <- "The cat sat on the mat"

gsub("at", "og", x)                           # all replacements
sub("at", "og", x)                            # first only
gsub("the", "a", x, ignore.case = TRUE)       # case-insensitive
gsub(".", "X", "a.b.c", fixed = TRUE)         # fixed = literal string
gsub(".", "X", "a.b.c")                       # "XXX" — . is regex!
```

---

## Regular Expressions

Patterns for matching text. Essential for parsing real data.

| Pattern | Matches |
|---------|---------|
| `.` | Any character |
| `^` | Start of string |
| `$` | End of string |
| `*` | Zero or more |
| `+` | One or more |
| `?` | Zero or one |
| `[abc]` | Any of a, b, c |
| `[^abc]` | Not a, b, or c |
| `[a-z]` | Any lowercase letter |
| `[0-9]` | Any digit |
| `\\d` | Any digit |
| `\\w` | Word character `[a-zA-Z0-9_]` |
| `\\s` | Whitespace |
| `\\b` | Word boundary |
| `{n}` | Exactly n times |
| `{n,m}` | n to m times |
| `(abc)` | Capture group |
| `a\|b` | a or b |

Examples:

```r
x <- c("foo123", "bar", "baz456", "123", "foobar")

grepl("^[0-9]+$", x)          # only digits: FALSE FALSE FALSE TRUE FALSE
grepl("[0-9]", x)              # contains digit: TRUE FALSE TRUE TRUE FALSE
gsub("[0-9]+", "NUM", x)      # "fooNUM" "bar" "bazNUM" "NUM" "foobar"
```

**Capture groups** with `regmatches` + `regexec`:

```r
dates <- c("2026-03-15", "2025-12-01", "2024-07-04")
matches <- regmatches(dates, regexpr("\\d{4}", dates))
matches   # "2026" "2025" "2024"
```

---

## sprintf and formatC

```r
sprintf("%.2f%%", 95.678)                   # "95.68%"
sprintf("%-20s %5d", "Alice", 42)           # left-aligned name, right-aligned number
formatC(1234567, big.mark = ",", format = "d")  # "1,234,567"
formatC(0.000123, format = "e", digits = 2)     # "1.23e-04"
```

---

## Adding Regex Filtering to analyze.R

In Chapter 3, we added `--filter=salary>50000` (numeric comparison). Now add `--filter=dept~Eng` (regex match on a string column).

The `~` operator means "matches regex":

```r
# Extend apply_filter() to handle ~
apply_filter <- function(col_values, op, threshold) {
  if (op == "~") {
    # Regex match
    return(grepl(threshold, col_values))
  }
  if (op == "!~") {
    return(!grepl(threshold, col_values))
  }
  
  threshold_num <- suppressWarnings(as.numeric(threshold))
  
  if (!is.na(threshold_num)) {
    switch(op,
      ">"  = col_values > threshold_num,
      "<"  = col_values < threshold_num,
      ">=" = col_values >= threshold_num,
      "<=" = col_values <= threshold_num,
      "==" = col_values == threshold_num,
      "!=" = col_values != threshold_num,
      stop(paste("Unknown operator:", op))
    )
  } else {
    switch(op,
      "==" = col_values == threshold,
      "!=" = col_values != threshold,
      stop(paste("String comparison supports ==, !=, ~, !~"))
    )
  }
}
```

Test:

```r
depts   <- c("Engineering", "Sales", "Engineering", "HR", "Sales")
salaries <- c(72000, 48000, 95000, 55000, 83000)

keep_dept   <- apply_filter(depts,    "~",  "Eng")
keep_salary <- apply_filter(salaries, ">",  "60000")
keep_both   <- keep_dept & keep_salary

cat("Engineering earning over $60k:\n")
for (i in which(keep_both)) {
  cat(sprintf("  %s: $%d\n", depts[i], salaries[i]))
}
```

---

## Word Frequency Analyzer

```r
# word_freq.R — analyze text for word frequency

analyze_text <- function(text, top_n = 10) {
  clean <- tolower(text)
  clean <- gsub("[^a-z\\s]", " ", clean)
  clean <- gsub("\\s+", " ", clean)
  clean <- trimws(clean)
  
  words <- strsplit(clean, " ")[[1]]
  words <- words[nchar(words) > 0]
  
  stop_words <- c("the","a","an","and","or","but","in","on","at","to",
                  "for","of","with","by","from","is","was","are","were",
                  "be","been","have","has","had","do","does","did","it",
                  "its","this","that","i","you","he","she","we","they")
  
  content_words <- words[!words %in% stop_words]
  freq <- sort(table(content_words), decreasing = TRUE)
  
  cat(sprintf("Total words: %d | Unique: %d\n", length(words), length(unique(words))))
  cat(sprintf("Top %d content words:\n", top_n))
  for (i in seq_len(min(top_n, length(freq)))) {
    word  <- names(freq)[i]
    count <- freq[[i]]
    pct   <- 100 * count / length(content_words)
    bar   <- strrep("█", max(1, round(pct)))
    cat(sprintf("  %-15s  %3d  %5.1f%%  %s\n", word, count, pct, bar))
  }
  
  invisible(list(freq = freq, words = words))
}

text <- paste(
  "Call me Ishmael. Some years ago never mind how long precisely",
  "having little or no money in my purse and nothing particular to interest",
  "me on shore I thought I would sail about a little and see the watery part",
  "of the world."
)
analyze_text(text)
```

---

## analyze.R: Chapter 6 Version

What changed: `apply_filter()` now supports `~` and `!~` operators for regex matching on string columns.

```r
# analyze.R — Chapter 6 (apply_filter update shown)

apply_filter <- function(col_values, op, threshold) {
  if (op == "~")  return(grepl(threshold, col_values))
  if (op == "!~") return(!grepl(threshold, col_values))
  
  threshold_num <- suppressWarnings(as.numeric(threshold))
  
  if (!is.na(threshold_num)) {
    switch(op,
      ">"  = col_values > threshold_num,
      "<"  = col_values < threshold_num,
      ">=" = col_values >= threshold_num,
      "<=" = col_values <= threshold_num,
      "==" = col_values == threshold_num,
      "!=" = col_values != threshold_num,
      stop(paste("Unknown operator:", op))
    )
  } else {
    switch(op,
      "==" = col_values == threshold,
      "!=" = col_values != threshold,
      stop("String ops: ==, !=, ~, !~")
    )
  }
}

# parse_filter also needs to match the ~ operator
parse_filter <- function(filter_str) {
  parts <- regmatches(filter_str,
             regexec("^(\\w+)(==|!=|>=|<=|>|<|!~|~)(.+)$", filter_str))[[1]]
  if (length(parts) != 4) stop(paste("Invalid filter:", filter_str))
  list(column = parts[2], op = parts[3], value = parts[4])
}
```

Run it:

```
Rscript analyze.R data.csv --filter=dept~Eng
```

---

## Exercises

**1. Email validator**

Write `is_valid_email(x)` using `grepl` and a regex. Test with:
```r
emails <- c("user@example.com", "bad.email", "another@test.co.uk",
            "@no-user.com", "user@.com")
```

**2. Extract data from strings**

Given:
```r
entries <- c("Alice|30|Engineer", "Bob|25|Designer",
             "Carol|35|Manager", "Dave|28|Developer")
```
Parse into a data frame with columns: name, age, role. Use `strsplit` and `as.integer`.

**3. Caesar cipher**

Write `caesar(text, shift)` that shifts each letter by `shift` positions. `caesar("Hello, World!", 3)` → `"Khoor, Zruog!"`. Handle wrap-around. Leave non-letters unchanged.

**4. Palindrome checker**

Write `is_palindrome(s)` that returns TRUE if `s` is a palindrome, ignoring spaces, punctuation, and case. Test: "A man a plan a canal Panama" → TRUE.

**5. The growing program (do this one)**

Add `compute_stats_char()` to `analyze.R` from Chapter 5. Extend it with:
- Most common value (use `names(which.max(table(x)))`)
- Pattern detection: does the column look like dates? (`grepl("^\\d{4}-\\d{2}-\\d{2}$", x)`)
- If it looks like dates, report min/max date as strings

In Chapter 7, when we read real CSV files, every column comes in as character first. This function will tell us whether to treat it as numeric, date, or text.

---

*Next: Chapter 7 — I/O and Files: read real CSVs, write reports*

---

## Solutions

### Exercise 1 — Email validator

```r
is_valid_email <- function(x) {
  # Pattern: one or more word/dot/dash chars before @,
  # then a domain with at least one dot, ending with 2+ letters
  pattern <- "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
  grepl(pattern, x)
}

emails <- c("user@example.com", "bad.email", "another@test.co.uk",
            "@no-user.com", "user@.com")

result <- data.frame(email = emails, valid = is_valid_email(emails))
print(result)
#              email valid
# 1  user@example.com  TRUE
# 2        bad.email FALSE
# 3 another@test.co.uk  TRUE
# 4      @no-user.com FALSE
# 5         user@.com FALSE
```

### Exercise 2 — Extract data from strings

```r
entries <- c("Alice|30|Engineer", "Bob|25|Designer",
             "Carol|35|Manager", "Dave|28|Developer")

# Split each entry on "|" and build a data frame
parsed <- lapply(entries, function(e) {
  parts <- strsplit(e, "\\|")[[1]]
  list(name = parts[1], age = as.integer(parts[2]), role = parts[3])
})

people <- data.frame(
  name = sapply(parsed, `[[`, "name"),
  age  = sapply(parsed, `[[`, "age"),
  role = sapply(parsed, `[[`, "role"),
  stringsAsFactors = FALSE
)

print(people)
#    name age      role
# 1 Alice  30  Engineer
# 2   Bob  25  Designer
# 3 Carol  35   Manager
# 4  Dave  28 Developer
```

### Exercise 3 — Caesar cipher

```r
caesar <- function(text, shift) {
  # Normalise shift to 0–25
  shift <- shift %% 26

  chars <- strsplit(text, "")[[1]]
  shifted <- sapply(chars, function(ch) {
    code <- utf8ToInt(ch)
    if (code >= 65 && code <= 90) {         # uppercase A-Z
      intToUtf8((code - 65 + shift) %% 26 + 65)
    } else if (code >= 97 && code <= 122) { # lowercase a-z
      intToUtf8((code - 97 + shift) %% 26 + 97)
    } else {
      ch    # leave non-letters unchanged
    }
  })
  paste(shifted, collapse = "")
}

caesar("Hello, World!", 3)   # "Khoor, Zruog!"
caesar("Khoor, Zruog!", -3)  # "Hello, World!"  (decode)
caesar("XYZ", 3)             # "ABC"  (wraps around)
```

### Exercise 4 — Palindrome checker

```r
is_palindrome <- function(s) {
  # Keep only letters, convert to lowercase
  clean <- tolower(gsub("[^a-zA-Z]", "", s))
  # Compare the string to its reverse
  clean == paste(rev(strsplit(clean, "")[[1]]), collapse = "")
}

is_palindrome("A man a plan a canal Panama")   # TRUE
is_palindrome("racecar")                        # TRUE
is_palindrome("hello")                          # FALSE
is_palindrome("Was it a car or a cat I saw?")   # TRUE
```

### Exercise 5 — The growing program (compute_stats_char extension)

Add pattern detection and date detection to `compute_stats_char()` for use in Chapter 7 when we read real CSV files:

```r
# --- Addition to analyze.R (Chapter 6) ---

compute_stats_char <- function(x, na.rm = TRUE, col_name = "?") {
  if (na.rm) x <- x[!is.na(x)]
  if (length(x) == 0) return(list(n = 0L, n_unique = 0L, top = NA_character_))

  freq <- table(x)

  result <- list(
    n        = length(x),
    n_unique = length(unique(x)),
    top      = names(which.max(freq))
  )

  # Pattern: does this column look like dates (YYYY-MM-DD)?
  is_date <- all(grepl("^\\d{4}-\\d{2}-\\d{2}$", x))
  if (is_date) {
    result$type    <- "date"
    result$min_date <- min(x)   # lexicographic sort works for ISO dates
    result$max_date <- max(x)
    cat(sprintf("  Note: column '%s' looks like dates (%s to %s)\n",
                col_name, result$min_date, result$max_date))
  } else {
    result$type <- "character"
  }

  result
}

# Usage (simulated):
depts <- c("Engineering","Sales","Engineering","HR","Sales")
dates <- c("2026-01-15","2026-03-22","2025-12-01","2026-02-10","2026-04-01")

cat("\nColumn: dept\n")
s <- compute_stats_char(depts, col_name = "dept")
cat(sprintf("  n=%d  n_unique=%d  top=%s  type=%s\n",
            s$n, s$n_unique, s$top, s$type))

cat("\nColumn: hire_date\n")
s2 <- compute_stats_char(dates, col_name = "hire_date")
cat(sprintf("  n=%d  n_unique=%d  type=%s  range=%s to %s\n",
            s2$n, s2$n_unique, s2$type, s2$min_date, s2$max_date))

# Column: dept
#   n=5  n_unique=4  top=Engineering  type=character

# Note: column 'hire_date' looks like dates (2025-12-01 to 2026-04-01)
# Column: hire_date
#   n=5  n_unique=5  type=date  range=2025-12-01 to 2026-04-01
```
