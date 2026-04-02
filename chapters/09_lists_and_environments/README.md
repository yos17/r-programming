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
