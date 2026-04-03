# Chapter 9: Lists and Environments

*Heterogeneous containers. Named lists as records. Environments as namespaces.*

---

## Where We Left Off

`analyze.R` accepts `--group`, `--filter`, `--stat`, and `--output` arguments. Each time we add a feature, the argument parsing grows. By now the top of the script looks like:

```r
filter_str  <- get_arg(args, "--filter")
stat_str    <- get_arg(args, "--stat")
output_file <- get_arg(args, "--output")
group_col   <- get_arg(args, "--group")
```

Four separate variables, scattered through the code. And we want to add more: `--sep` for CSV separator, `--na` for how to treat missing values, `--decimal` for decimal point character.

Better: collect all configuration into one list. Then pass the list around. Functions that need configuration take `config` as an argument — they don't dig through global state.

---

## Lists

A list is like a vector, but each element can be any type — including another list.

```r
person <- list(
  name   = "Alice",
  age    = 30,
  scores = c(92, 88, 95),
  active = TRUE
)
```

Access elements three ways:

```r
person$name        # "Alice"   — dollar sign
person[["name"]]   # "Alice"   — double brackets (by name)
person[[1]]        # "Alice"   — double brackets (by position)
person["name"]     # list of length 1 (single brackets return a sub-list!)
```

`[[` extracts the element itself. `[` returns a sub-list.

Modify:

```r
person$email <- "alice@example.com"     # add
person$age   <- 31                      # modify
person$active <- NULL                   # remove
```

---

## Nested Lists

Lists can contain lists — ideal for hierarchical data:

```r
company <- list(
  name = "Acme Corp",
  employees = list(
    list(name = "Alice", role = "Engineer", salary = 90000),
    list(name = "Bob",   role = "Designer", salary = 75000),
    list(name = "Carol", role = "Manager",  salary = 105000)
  )
)

company$employees[[1]]$name      # "Alice"
company$employees[[2]]$salary    # 75000
```

Iterate:

```r
for (emp in company$employees) {
  cat(sprintf("  %s (%s): $%s\n",
              emp$name, emp$role,
              formatC(emp$salary, format="d", big.mark=",")))
}
```

---

## lapply and sapply

Apply a function to every element of a list:

```r
numbers <- list(a = c(1,2,3), b = c(4,5,6), c = c(7,8,9))

lapply(numbers, mean)   # returns a list
sapply(numbers, mean)   # returns a named vector (simplified)
```

`sapply` tries to simplify. If all results are the same type and length, it returns a vector or matrix. Otherwise, a list.

```r
# Extract a field from each list element
salaries <- sapply(company$employees, function(emp) emp$salary)
salaries  # 90000 75000 105000
```

---

## Building the Config System

The pattern: a list of defaults, overridden by user input.

```r
# Default config — all options with sensible values
default_config <- list(
  input = list(
    sep              = ",",
    na_strings       = c("NA", "", "N/A", "null"),
    header           = TRUE,
    decimal          = ".",
    stringsAsFactors = FALSE
  ),
  analysis = list(
    stats  = c("n", "mean", "sd", "median", "min", "max"),
    na_rm  = TRUE,
    digits = 4
  ),
  output = list(
    file   = NULL,
    format = "text",  # "text" or "csv"
    quiet  = FALSE
  ),
  filter = NULL,
  group  = NULL
)
```

Merge function — user values override defaults, but keep defaults for anything not specified:

```r
merge_config <- function(default, override) {
  result <- default
  for (key in names(override)) {
    if (is.list(default[[key]]) && is.list(override[[key]])) {
      result[[key]] <- merge_config(default[[key]], override[[key]])
    } else {
      result[[key]] <- override[[key]]
    }
  }
  result
}
```

---

## Parsing Arguments into Config

```r
args_to_config <- function(args) {
  override <- list()
  
  # --filter=salary>50000
  f <- grep("^--filter=", args, value = TRUE)
  if (length(f) > 0) override$filter <- sub("^--filter=", "", f[1])
  
  # --group=dept
  g <- grep("^--group=", args, value = TRUE)
  if (length(g) > 0) override$group <- sub("^--group=", "", g[1])
  
  # --stat=mean,sd,median
  s <- grep("^--stat=", args, value = TRUE)
  if (length(s) > 0)
    override$analysis <- list(stats = strsplit(sub("^--stat=", "", s[1]), ",")[[1]])
  
  # --output=report.txt
  o <- grep("^--output=", args, value = TRUE)
  if (length(o) > 0)
    override$output <- list(file = sub("^--output=", "", o[1]))
  
  # --sep=;
  sep <- grep("^--sep=", args, value = TRUE)
  if (length(sep) > 0)
    override$input <- list(sep = sub("^--sep=", "", sep[1]))
  
  merge_config(default_config, override)
}
```

---

## Validating Config

```r
validate_config <- function(cfg) {
  errors <- character(0)
  
  valid_stats <- c("n","mean","sd","median","min","max","q1","q3","iqr")
  bad_stats   <- setdiff(cfg$analysis$stats, valid_stats)
  if (length(bad_stats) > 0)
    errors <- c(errors, paste("Unknown stats:", paste(bad_stats, collapse=",")))
  
  valid_formats <- c("text","csv")
  if (!cfg$output$format %in% valid_formats)
    errors <- c(errors, paste("Unknown format:", cfg$output$format))
  
  if (length(errors) > 0) {
    for (e in errors) cat("Config error:", e, "\n")
    quit(status = 1)
  }
  invisible(TRUE)
}
```

---

## Environments

An environment is like a list — a collection of named objects — but with different semantics. It's R's internal mechanism for variable storage.

```r
e <- new.env()
e$x <- 42
e$y <- "hello"

ls(e)          # "x" "y"
e$x            # 42
```

Environments have parent environments (the enclosure chain). Every function has its own environment.

The main practical use: **closures** — functions that capture their enclosing environment.

```r
make_counter <- function() {
  count <- 0
  list(
    increment = function() count <<- count + 1,
    reset     = function() count <<- 0,
    get       = function() count
  )
}

counter <- make_counter()
counter$increment()
counter$increment()
counter$increment()
counter$get()     # 3
counter$reset()
counter$get()     # 0
```

Each call to `make_counter()` creates a fresh environment with its own `count`. The returned functions all share that environment.

---

## analyze.R: Chapter 9 Version

What changed: all loose arguments collapsed into one `config` list. Functions that need settings take `config` as a parameter.

```r
# analyze.R — Chapter 9
# Usage: Rscript analyze.R <file.csv> [--group=col] [--filter=col>val]
#                                     [--stat=mean,sd] [--output=file.txt]
#                                     [--sep=,] [--format=text]

args <- commandArgs(trailingOnly = TRUE)

if (length(args) < 1) {
  cat("Usage: Rscript analyze.R <file.csv> [options]\n")
  cat("Options:\n")
  cat("  --filter=col>val     Filter rows\n")
  cat("  --group=col          Group by column\n")
  cat("  --stat=mean,sd,...   Statistics to compute\n")
  cat("  --output=file.txt    Write report to file\n")
  cat("  --sep=,              CSV separator\n")
  quit(status = 1)
}

filename <- args[1]
if (!file.exists(filename)) {
  cat("Error: file not found:", filename, "\n")
  quit(status = 1)
}

# Build config from args
config <- args_to_config(args)
validate_config(config)

# Read file using config
df_raw <- read.csv(filename,
  sep              = config$input$sep,
  header           = config$input$header,
  na.strings       = config$input$na_strings,
  dec              = config$input$decimal,
  stringsAsFactors = config$input$stringsAsFactors
)

if (!config$output$quiet)
  cat(sprintf("Read %d rows × %d columns from %s\n", nrow(df_raw), ncol(df_raw), filename))

# ... (rest of processing uses config throughout)
```

Now the function signatures are clean:

```r
compute_grouped_stats <- function(df, group_col, value_cols, config) {
  stats <- config$analysis$stats
  na.rm <- config$analysis$na_rm
  # ...
}
```

No global state. All behavior flows through `config`.

---

## Exercises

**1. JSON-like data**

Represent a book catalogue as a list of lists. Each book has: title, author, year, isbn, price, in_stock. Write functions to:
- Add a book
- Find books by author
- Find books under a price threshold
- Calculate total inventory value

**2. Config persistence**

Add `save_config(cfg, file)` and `load_config(file)` using `saveRDS`/`readRDS`. Then add `set_config(cfg, key, value)` that takes a dotted key like `"analysis.digits"` and sets the nested value.

**3. Memoization with environments**

A proper memoization function:
```r
memoize <- function(f) {
  cache <- new.env(hash = TRUE, parent = emptyenv())
  function(...) {
    key <- paste(list(...), collapse = "|")
    if (exists(key, envir = cache)) return(get(key, envir = cache))
    result <- f(...)
    assign(key, result, envir = cache)
    result
  }
}
```

**4. Stack and queue**

Implement a stack and queue using lists:
- Stack: `push(s, x)`, `pop(s)`, `peek(s)`, `is_empty(s)`
- Queue: `enqueue(q, x)`, `dequeue(q)`, `front(q)`, `is_empty(q)`

**5. The growing program (do this one)**

Add `--config=config.rds` support to `analyze.R`. If passed, load config from the file and merge with any command-line overrides (command-line wins). This way, users can save a config for a recurring report:

```r
# Create config file
cfg <- default_config
cfg$analysis$stats <- c("mean","sd","n")
cfg$group <- "dept"
saveRDS(cfg, "payroll_config.rds")

# Use it later
Rscript analyze.R employees.csv --config=payroll_config.rds --output=report.txt
```

In Chapter 10, we'll process *multiple* files at once. The config system makes this clean: one config object controls how all files are processed.

---

*Next: Chapter 10 — The Apply Family: process multiple files with lapply/sapply*

---

## Solutions

### Exercise 1 — JSON-like book catalogue

```r
# Book catalogue as a list of lists

new_book <- function(title, author, year, isbn, price, in_stock = TRUE) {
  list(title=title, author=author, year=year, isbn=isbn,
       price=price, in_stock=in_stock)
}

catalogue <- list(
  new_book("The R Inferno",      "Patrick Burns",     2011, "978-0000001",  15.99),
  new_book("Advanced R",         "Hadley Wickham",    2019, "978-0000002",  49.99),
  new_book("R for Data Science", "Hadley Wickham",    2023, "978-0000003",  39.99),
  new_book("The Art of R",       "Norman Matloff",    2011, "978-0000004",  35.00, FALSE),
  new_book("R Packages",         "Hadley Wickham",    2023, "978-0000005",  45.00)
)

# Add a book
add_book <- function(cat, ...) c(cat, list(new_book(...)))

# Find books by author
find_by_author <- function(cat, author) {
  Filter(function(b) grepl(author, b$author, ignore.case = TRUE), cat)
}

# Find books under a price threshold
find_under <- function(cat, max_price) {
  Filter(function(b) b$price <= max_price, cat)
}

# Total inventory value (in-stock only)
inventory_value <- function(cat) {
  sum(sapply(Filter(function(b) b$in_stock, cat), `[[`, "price"))
}

# --- Tests ---
wickham_books <- find_by_author(catalogue, "Wickham")
cat("Books by Wickham:", length(wickham_books), "\n")
sapply(wickham_books, `[[`, "title")

cheap <- find_under(catalogue, 40)
cat("Books under $40:", length(cheap), "\n")

cat(sprintf("Total inventory value: $%.2f\n", inventory_value(catalogue)))
# Total inventory value: $185.97  (excludes out-of-stock "The Art of R")
```

### Exercise 2 — Config persistence

```r
# Save and load config (using saveRDS/readRDS)

save_config <- function(cfg, file) {
  saveRDS(cfg, file)
  cat(sprintf("Config saved to %s\n", file))
  invisible(cfg)
}

load_config <- function(file) {
  if (!file.exists(file)) stop(paste("Config file not found:", file))
  cfg <- readRDS(file)
  cat(sprintf("Config loaded from %s\n", file))
  cfg
}

# set_config: set nested value using dotted key like "analysis.digits"
set_config <- function(cfg, key, value) {
  keys <- strsplit(key, "\\.")[[1]]
  # Recursive descent
  set_nested <- function(obj, ks, val) {
    if (length(ks) == 1) {
      obj[[ks[1]]] <- val
    } else {
      obj[[ks[1]]] <- set_nested(obj[[ks[1]]], ks[-1], val)
    }
    obj
  }
  set_nested(cfg, keys, value)
}

# --- Usage ---
cfg <- list(analysis = list(stats = c("mean","sd"), digits = 4),
            output   = list(file = NULL))

cfg <- set_config(cfg, "analysis.digits", 6)
cfg <- set_config(cfg, "output.file", "report.txt")

cat("digits:", cfg$analysis$digits, "\n")   # 6
cat("output:", cfg$output$file,     "\n")   # report.txt

save_config(cfg, "my_config.rds")
cfg2 <- load_config("my_config.rds")
identical(cfg, cfg2)   # TRUE
```

### Exercise 3 — Memoization with environments

```r
memoize <- function(f) {
  cache <- new.env(hash = TRUE, parent = emptyenv())
  function(...) {
    key <- paste(list(...), collapse = "|")
    if (exists(key, envir = cache)) {
      return(get(key, envir = cache))
    }
    result <- f(...)
    assign(key, result, envir = cache)
    result
  }
}

# Test: slow Fibonacci (exponential without memoization)
fib <- function(n) {
  if (n <= 1) return(n)
  fib(n - 1) + fib(n - 2)
}

fib_memo <- memoize(function(n) {
  if (n <= 1) return(n)
  fib_memo(n - 1) + fib_memo(n - 2)
})

system.time(fib(30))       # ~1–2 seconds
system.time(fib_memo(30))  # near-instant
system.time(fib_memo(30))  # instant (cached)

fib_memo(10)  # 55
fib_memo(20)  # 6765
```

### Exercise 4 — Stack and queue

```r
# Stack (LIFO)
new_stack <- function() list(items = list())

push <- function(s, x) { s$items <- c(s$items, list(x)); s }
pop  <- function(s) {
  if (is_empty(s)) stop("Stack is empty")
  val <- s$items[[length(s$items)]]
  s$items <- s$items[-length(s$items)]
  list(value = val, stack = s)
}
peek     <- function(s) { if (is_empty(s)) stop("Stack is empty"); s$items[[length(s$items)]] }
is_empty <- function(s) length(s$items) == 0

# Queue (FIFO)
new_queue  <- function() list(items = list())

enqueue <- function(q, x) { q$items <- c(q$items, list(x)); q }
dequeue <- function(q) {
  if (is_empty(q)) stop("Queue is empty")
  val <- q$items[[1]]
  q$items <- q$items[-1]
  list(value = val, queue = q)
}
front <- function(q) { if (is_empty(q)) stop("Queue is empty"); q$items[[1]] }

# --- Tests ---
s <- new_stack()
s <- push(s, 10); s <- push(s, 20); s <- push(s, 30)
cat("peek:", peek(s), "\n")         # 30 (LIFO)
r <- pop(s); cat("popped:", r$value, "\n")  # 30

q <- new_queue()
q <- enqueue(q, "first"); q <- enqueue(q, "second"); q <- enqueue(q, "third")
cat("front:", front(q), "\n")      # "first" (FIFO)
r <- dequeue(q); cat("dequeued:", r$value, "\n")  # "first"
```

### Exercise 5 — The growing program (`--config` support)

Add `--config=config.rds` to `analyze.R`. The file-based config is loaded first, then command-line flags override it.

```r
# --- Addition to analyze.R (Chapter 9) ---
# Place this block just after defining default_config and merge_config.

args_to_config <- function(args) {
  override <- list()

  get_val <- function(flag) {
    m <- grep(paste0("^--", flag, "="), args, value = TRUE)
    if (length(m) == 0) return(NULL)
    sub(paste0("^--", flag, "="), "", m[1])
  }

  if (!is.null(v <- get_val("filter"))) override$filter <- v
  if (!is.null(v <- get_val("group")))  override$group  <- v
  if (!is.null(v <- get_val("output"))) override$output <- list(file = v)
  if (!is.null(v <- get_val("sep")))    override$input  <- list(sep  = v)
  if (!is.null(v <- get_val("stat")))
    override$analysis <- list(stats = strsplit(v, ",")[[1]])

  # Start from defaults
  cfg <- merge_config(default_config, override)

  # Load saved config if --config= was provided, then re-apply overrides on top
  if (!is.null(cfg_file <- get_val("config"))) {
    if (!file.exists(cfg_file)) {
      cat("Warning: config file not found:", cfg_file, "\n")
    } else {
      saved_cfg <- readRDS(cfg_file)
      # saved config < command-line overrides
      cfg <- merge_config(saved_cfg, override)
      cat(sprintf("Loaded config from %s\n", cfg_file))
    }
  }

  cfg
}

# --- Demo: save a reusable config ---
# cfg <- default_config
# cfg$analysis$stats <- c("mean","sd","n")
# cfg$group <- "dept"
# saveRDS(cfg, "payroll_config.rds")
#
# Rscript analyze.R employees.csv --config=payroll_config.rds --output=report.txt
# → Uses group=dept, stats=mean,sd,n from file, output from command line
```
