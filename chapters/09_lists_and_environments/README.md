# Chapter 9: Lists and Environments

*Heterogeneous containers. Named lists as records. Environments as namespaces.*

---

## Lists

A list is like a vector, but each element can be any type — including another list.

```r
person <- list(
  name  = "Alice",
  age   = 30,
  scores = c(92, 88, 95),
  active = TRUE
)
```

Access elements three ways:

```r
person$name        # "Alice"   — dollar sign
person[["name"]]   # "Alice"   — double brackets (by name)
person[[1]]        # "Alice"   — double brackets (by position)
person["name"]     # list of length 1 (single brackets return a list!)
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

Lists can contain lists — ideal for representing hierarchical data:

```r
company <- list(
  name = "Acme Corp",
  employees = list(
    list(name = "Alice", role = "Engineer", salary = 90000),
    list(name = "Bob",   role = "Designer", salary = 75000),
    list(name = "Carol", role = "Manager",  salary = 105000)
  )
)

# Access nested elements
company$employees[[1]]$name      # "Alice"
company$employees[[2]]$salary    # 75000
```

Iterate over a list:

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
sapply(numbers, mean)   # returns a vector (simplified)
```

`sapply` tries to simplify the result. If all results are the same type and same length, it returns a vector or matrix. Otherwise, it returns a list.

```r
# Extract a field from each list element
salaries <- sapply(company$employees, function(emp) emp$salary)
salaries  # 90000 75000 105000
```

---

## Program: Configuration System

A real-world use of lists — a config system that reads, validates, and uses nested configuration:

```r
# config.R
# A configuration management system

# ---- Default configuration ----
default_config <- list(
  server = list(
    host = "localhost",
    port = 8080,
    workers = 4
  ),
  database = list(
    host     = "localhost",
    port     = 5432,
    name     = "myapp",
    timeout  = 30
  ),
  logging = list(
    level  = "INFO",
    file   = "app.log",
    rotate = TRUE
  ),
  features = list(
    cache     = TRUE,
    analytics = FALSE,
    debug     = FALSE
  )
)

# ---- Merge two configs (user overrides defaults) ----
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

# ---- Validate config ----
validate_config <- function(cfg) {
  errors <- character(0)
  
  if (!is.numeric(cfg$server$port) || cfg$server$port < 1 || cfg$server$port > 65535)
    errors <- c(errors, "server.port must be 1-65535")
  
  if (!is.numeric(cfg$server$workers) || cfg$server$workers < 1)
    errors <- c(errors, "server.workers must be >= 1")
  
  if (!cfg$logging$level %in% c("DEBUG", "INFO", "WARNING", "ERROR"))
    errors <- c(errors, "logging.level must be DEBUG/INFO/WARNING/ERROR")
  
  if (length(errors) > 0) {
    stop(paste("Config errors:\n", paste("-", errors, collapse="\n ")))
  }
  invisible(TRUE)
}

# ---- Accessor: nested key lookup ----
# "server.port" → cfg$server$port
get_config <- function(cfg, key) {
  keys <- strsplit(key, "\\.")[[1]]
  result <- cfg
  for (k in keys) {
    if (!k %in% names(result)) return(NULL)
    result <- result[[k]]
  }
  result
}

# ---- Print config tree ----
print_config <- function(cfg, indent = 0) {
  for (key in names(cfg)) {
    if (is.list(cfg[[key]])) {
      cat(strrep("  ", indent), key, ":\n", sep="")
      print_config(cfg[[key]], indent + 1)
    } else {
      val <- cfg[[key]]
      cat(sprintf("%s%-12s = %s\n", strrep("  ", indent), key, 
                  paste(val, collapse=", ")))
    }
  }
}

# ---- Use it ----
user_config <- list(
  server = list(port = 9000, workers = 8),
  features = list(analytics = TRUE, debug = TRUE)
)

config <- merge_config(default_config, user_config)
validate_config(config)

cat("=== Active Configuration ===\n")
print_config(config)

cat("\n=== Key Lookups ===\n")
cat("server.port:      ", get_config(config, "server.port"), "\n")
cat("logging.level:    ", get_config(config, "logging.level"), "\n")
cat("features.debug:   ", get_config(config, "features.debug"), "\n")
```

Output:
```
=== Active Configuration ===
server      :
  host         = localhost
  port         = 9000
  workers      = 8
database    :
  host         = localhost
  port         = 5432
  ...
features    :
  cache        = TRUE
  analytics    = TRUE
  debug        = TRUE
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
get("x", envir = e)   # 42
```

Environments have parent environments (the enclosure chain). Every function has its own environment.

The main use you'll encounter: **closures** — functions that capture their enclosing environment.

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

Each call to `make_counter()` creates a fresh environment with its own `count`. The returned list of functions all share that environment — they form a closure.

---

## Exercises

**1. JSON-like data**

Represent a book catalogue as a list of lists. Each book has: title, author, year, isbn, price, in_stock. Write functions to:
- Add a book
- Find books by author
- Find books under a price threshold
- Calculate total inventory value

**2. Extend the config system**

Add `save_config(cfg, file)` and `load_config(file)` using `saveRDS`/`readRDS`. Add `set_config(cfg, key, value)` that takes a dotted key like `"server.port"` and sets the nested value.

**3. Memoization with environments**

A proper memoization function:
```r
memoize <- function(f) {
  cache <- new.env(hash = TRUE, parent = emptyenv())
  function(...) {
    key <- paste(list(...), collapse="|")
    if (exists(key, envir = cache)) {
      return(get(key, envir = cache))
    }
    result <- f(...)
    assign(key, result, envir = cache)
    result
  }
}
```
Use it: `fib_m <- memoize(fib)`. Time the difference.

**4. Stack and queue**

Implement a stack and a queue using lists:
- Stack: `push(s, x)`, `pop(s)`, `peek(s)`, `is_empty(s)`
- Queue: `enqueue(q, x)`, `dequeue(q)`, `front(q)`, `is_empty(q)`

---

*Next: Chapter 10 — The Apply Family: avoiding loops with vectorized tools*
