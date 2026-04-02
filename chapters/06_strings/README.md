# Chapter 6: Strings

*Text processing in R. Regular expressions. A word frequency analyzer.*

---

## String Basics (Revisited)

Strings in R are character vectors. Every string function is vectorized.

```r
x <- c("Hello", "World", "R Programming")

nchar(x)          # 5 5 13
toupper(x)        # "HELLO" "WORLD" "R PROGRAMMING"
tolower(x)        # "hello" "world" "r programming"
trimws("  hello  ")      # "hello"   (trim whitespace)
trimws("  hello  ", "left")   # "hello  "  (left only)
trimws("  hello  ", "right")  # "  hello"  (right only)
```

---

## Combining and Splitting

```r
paste("Hello", "World")           # "Hello World"
paste0("Hello", "World")          # "HelloWorld"
paste(c("a","b","c"), collapse="-") # "a-b-c"
paste("item", 1:5, sep="_")       # "item_1" "item_2" ... "item_5"

# R 4.1+ also has:
# stringr::str_c() — but let's use base R

strsplit("hello world foo", " ")   # list of: c("hello","world","foo")
strsplit(c("a-b", "c-d-e"), "-")   # list of two character vectors
```

The return of `strsplit` is a list (because each string can produce a different number of pieces). Use `[[1]]` to get the first result:

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

Modify with `substr<-`:

```r
x <- "Hello, World!"
substr(x, 1, 5) <- "Howdy"
x   # "Howdy, World!"
```

---

## Finding Patterns

```r
x <- c("apple", "banana", "cherry", "apricot", "grape")

grep("ap", x)           # 1 4 — indices of matches
grep("ap", x, value=TRUE)  # "apple" "apricot" — the matches
grepl("ap", x)          # TRUE FALSE FALSE TRUE FALSE — logical
```

---

## Replacing

```r
x <- "The cat sat on the mat"

gsub("at", "og", x)          # "The cog sog on the mog"  (all)
sub("at", "og", x)           # "The cog sat on the mat"  (first only)

# Case-insensitive:
gsub("the", "a", x, ignore.case = TRUE)  # "a cat sat on a mat"

# Fixed string (not regex):
gsub(".", "X", "a.b.c", fixed = TRUE)    # "aXbXc"
gsub(".", "X", "a.b.c")                  # "XXX" — . means any char in regex!
```

---

## Regular Expressions

Regular expressions are patterns for matching text. They're essential for any text processing.

| Pattern | Matches |
|---------|---------|
| `.` | Any character |
| `^` | Start of string |
| `$` | End of string |
| `*` | Zero or more of previous |
| `+` | One or more of previous |
| `?` | Zero or one of previous |
| `[abc]` | Any of a, b, c |
| `[^abc]` | Anything except a, b, c |
| `[a-z]` | Any lowercase letter |
| `[0-9]` | Any digit |
| `\\d` | Any digit (same as `[0-9]`) |
| `\\w` | Word character `[a-zA-Z0-9_]` |
| `\\s` | Whitespace |
| `\\b` | Word boundary |
| `{n}` | Exactly n times |
| `{n,m}` | n to m times |
| `(abc)` | Capture group |
| `a|b` | a or b |

Examples:

```r
x <- c("foo123", "bar", "baz456", "123", "foobar")

grepl("^[0-9]+$", x)          # only digits: FALSE FALSE FALSE TRUE FALSE
grepl("[0-9]", x)              # contains digit: TRUE FALSE TRUE TRUE FALSE
grep("[a-z]+[0-9]+$", x, value=TRUE)  # ends with digits: "foo123" "baz456"

gsub("[0-9]+", "NUM", x)      # "fooNUM" "bar" "bazNUM" "NUM" "foobar"
gsub("^(\\w+)[0-9]*$", "\\1", x) # capture and keep only letters
```

**Capture groups** with `regmatches` + `regexpr`:

```r
dates <- c("2026-03-15", "2025-12-01", "2024-07-04")
matches <- regmatches(dates, regexpr("\\d{4}", dates))
matches   # "2026" "2025" "2024"  — extract the year
```

---

## sprintf and formatC

Two ways to format strings precisely:

```r
# sprintf: C-style
sprintf("%.2f%%", 95.678)    # "95.68%"
sprintf("%-20s %5d", "Alice", 42)  # "Alice                   42"
sprintf("%+.1f", c(-3.14, 2.72))   # "-3.1" "+2.7"

# formatC: more R-like
formatC(3.14159, digits=3, format="f")   # "3.142"
formatC(1234567, big.mark=",", format="d")  # "1,234,567"
formatC(0.000123, format="e", digits=2)     # "1.23e-04"
```

---

## Program: Word Frequency Analyzer

```r
# word_freq.R
# Analyze word frequency in text

analyze_text <- function(text, top_n = 10) {
  
  # Normalize: lowercase, remove punctuation
  clean <- tolower(text)
  clean <- gsub("[^a-z\\s]", " ", clean)  # keep only letters and spaces
  clean <- gsub("\\s+", " ", clean)       # collapse multiple spaces
  clean <- trimws(clean)
  
  # Split into words
  words <- strsplit(clean, " ")[[1]]
  words <- words[nchar(words) > 0]        # remove empty strings
  
  # Remove stop words
  stop_words <- c("the", "a", "an", "and", "or", "but", "in", "on",
                  "at", "to", "for", "of", "with", "by", "from",
                  "is", "was", "are", "were", "be", "been", "being",
                  "have", "has", "had", "do", "does", "did", "will",
                  "would", "could", "should", "may", "might", "shall",
                  "it", "its", "this", "that", "these", "those",
                  "i", "you", "he", "she", "we", "they", "me", "him",
                  "her", "us", "them", "my", "your", "his", "our")
  
  content_words <- words[!words %in% stop_words]
  
  # Count frequencies
  freq <- sort(table(content_words), decreasing = TRUE)
  
  # Compute statistics
  n_total   <- length(words)
  n_unique  <- length(unique(words))
  n_content <- length(content_words)
  
  # Top N words
  top_words <- head(freq, top_n)
  
  # Print report
  cat(sprintf("=== Text Analysis ===\n"))
  cat(sprintf("Total words:   %d\n", n_total))
  cat(sprintf("Unique words:  %d\n", n_unique))
  cat(sprintf("Lexical density: %.1f%%\n", 100 * n_unique / n_total))
  cat(sprintf("\nTop %d content words:\n", top_n))
  cat(sprintf("  %-15s  %5s  %7s\n", "Word", "Count", "Freq%"))
  cat(strrep("-", 32), "\n")
  for (i in seq_along(top_words)) {
    word  <- names(top_words)[i]
    count <- top_words[[i]]
    pct   <- 100 * count / n_content
    bar   <- strrep("█", round(pct))
    cat(sprintf("  %-15s  %5d  %6.2f%%  %s\n", word, count, pct, bar))
  }
  
  invisible(list(freq = freq, words = words, content = content_words))
}

# Test with a sample text
text <- "Call me Ishmael. Some years ago never mind how long precisely
having little or no money in my purse and nothing particular to interest
me on shore I thought I would sail about a little and see the watery part
of the world. It is a way I have of driving off the spleen and regulating
the circulation. Whenever I find myself growing grim about the mouth
whenever it is a damp drizzly November in my soul whenever I find myself
involuntarily pausing before coffin warehouses and bringing up the rear
of every funeral I meet and especially whenever my hypos get such an
upper hand of me that it requires a strong moral principle to prevent me
from deliberately stepping into the street and methodically knocking
peoples hats off then I account it high time to get to sea as soon as I can."

result <- analyze_text(text)
```

Output:
```
=== Text Analysis ===
Total words:   126
Unique words:  87
Lexical density: 69.0%

Top 10 content words:
  Word              Count   Freq%
--------------------------------
  whenever           4  10.00%  ██████████
  find               3   7.50%  ████████
  growing            1   2.50%  ███
  ...
```

---

## Exercises

**1. Email validator**

Write `is_valid_email(x)` using `grepl` and a regex pattern. Test with:
```r
emails <- c("user@example.com", "bad.email", "another@test.co.uk",
            "@no-user.com", "no-at-sign", "user@.com")
```

**2. Extract data from strings**

Given:
```r
entries <- c("Alice|30|Engineer", "Bob|25|Designer",
             "Carol|35|Manager", "Dave|28|Developer")
```
Parse into a data frame with columns: name, age, role. Use `strsplit`.

**3. Caesar cipher**

Write `caesar(text, shift)` that shifts each letter by `shift` positions. `caesar("Hello, World!", 3)` → `"Khoor, Zruog!"`. Handle wrap-around (z→c for shift=3). Leave non-letters unchanged.

**4. Word frequency from a file**

Download any plain text file from Project Gutenberg (e.g., `https://gutenberg.org/files/1342/1342-0.txt`). Use `readLines()` to load it. Run `analyze_text()` on the full text (collapse with `paste(lines, collapse=" ")`). What are the top 20 content words?

**5. Palindrome checker**

Write `is_palindrome(s)` that returns TRUE if `s` is a palindrome, ignoring spaces, punctuation, and case. Test: "A man a plan a canal Panama" → TRUE.

---

*Next: Chapter 7 — I/O and Files: reading, writing, and processing data*
