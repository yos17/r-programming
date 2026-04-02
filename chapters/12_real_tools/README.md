# Chapter 12: Real Tools

*Everything together. A complete data analysis pipeline with visualization and reporting.*

---

## What You've Built

Before this final chapter, you have:

- Chapter 1–2: variables, types, expressions, formatted output
- Chapter 3: control flow — if, for, while
- Chapter 4: functions — arguments, scope, recursion
- Chapter 5: vectors and matrices — indexing, vectorized ops
- Chapter 6: strings — regex, pattern matching, text analysis
- Chapter 7: I/O — CSV, file handling
- Chapter 8: data frames — subsetting, merging, aggregating
- Chapter 9: lists and environments — closures, nested structures
- Chapter 10: apply family — lapply, sapply, tapply, Reduce
- Chapter 11: packages and S3 OOP

Now we build something real.

---

## The Project: Sales Analysis Pipeline

A complete analysis of a fictional company's sales data:

1. Generate and save realistic data
2. Load, clean, and validate it
3. Compute business metrics
4. Produce a formatted text report
5. Plot visualizations (base R graphics)

---

## Part 1: Data Generation

```r
# sales_pipeline.R
# Complete sales analysis pipeline

library(grDevices)   # for PDF output

set.seed(2026)

# ---- Generate sales data ----
generate_sales <- function(n_months = 24) {
  
  products   <- c("Widget A", "Widget B", "Gadget X", "Gadget Y", "Service Z")
  regions    <- c("North", "South", "East", "West")
  channels   <- c("Online", "Retail", "B2B")
  
  base_prices <- c(29.99, 49.99, 99.99, 149.99, 299.99)
  names(base_prices) <- products
  
  # Generate monthly sales
  months <- seq(as.Date("2024-01-01"), by = "month", length.out = n_months)
  
  rows <- lapply(months, function(month) {
    n_sales <- sample(50:200, 1)
    prod    <- sample(products, n_sales, replace = TRUE,
                      prob = c(0.30, 0.25, 0.20, 0.15, 0.10))
    reg     <- sample(regions,  n_sales, replace = TRUE)
    chan    <- sample(channels, n_sales, replace = TRUE,
                     prob = c(0.45, 0.35, 0.20))
    qty     <- sample(1:10, n_sales, replace = TRUE)
    # Price with random discount (0-15%)
    discount <- runif(n_sales, 0, 0.15)
    price    <- base_prices[prod] * (1 - discount)
    
    data.frame(
      date     = month,
      product  = prod,
      region   = reg,
      channel  = chan,
      quantity = qty,
      unit_price = round(price, 2),
      revenue  = round(price * qty, 2),
      stringsAsFactors = FALSE
    )
  })
  
  do.call(rbind, rows)
}

sales <- generate_sales(24)
write.csv(sales, "sales_data.csv", row.names = FALSE)
cat(sprintf("Generated %d sales records\n", nrow(sales)))
```

---

## Part 2: Load and Validate

```r
# ---- Load and validate ----
load_and_validate <- function(path) {
  df <- read.csv(path, stringsAsFactors = FALSE)
  df$date <- as.Date(df$date)
  
  errors <- character(0)
  
  # Check columns
  required <- c("date", "product", "region", "channel", "quantity",
                "unit_price", "revenue")
  missing_cols <- setdiff(required, names(df))
  if (length(missing_cols) > 0)
    errors <- c(errors, paste("Missing columns:", paste(missing_cols, collapse=", ")))
  
  # Check types
  if (!inherits(df$date, "Date"))
    errors <- c(errors, "date column is not a Date")
  
  if (any(df$quantity <= 0, na.rm = TRUE))
    errors <- c(errors, sprintf("%d rows with non-positive quantity",
                               sum(df$quantity <= 0, na.rm = TRUE)))
  
  if (any(df$revenue < 0, na.rm = TRUE))
    errors <- c(errors, sprintf("%d rows with negative revenue",
                               sum(df$revenue < 0, na.rm = TRUE)))
  
  # Check revenue calculation
  expected_rev <- df$unit_price * df$quantity
  tolerance <- 0.01
  mismatched <- sum(abs(df$revenue - expected_rev) > tolerance, na.rm = TRUE)
  if (mismatched > 0)
    errors <- c(errors, sprintf("%d rows with revenue mismatch", mismatched))
  
  if (length(errors) > 0) {
    cat("VALIDATION ERRORS:\n")
    for (e in errors) cat(" -", e, "\n")
    stop("Data validation failed")
  }
  
  cat(sprintf("Loaded and validated: %d rows, %d columns\n", nrow(df), ncol(df)))
  df
}

df <- load_and_validate("sales_data.csv")
```

---

## Part 3: Metrics

```r
# ---- Compute business metrics ----

# Total revenue
total_rev <- sum(df$revenue)
total_qty <- sum(df$quantity)
avg_order <- mean(df$revenue)

# Monthly revenue
df$year_month <- format(df$date, "%Y-%m")
monthly_rev   <- tapply(df$revenue, df$year_month, sum)
monthly_orders <- tapply(df$revenue, df$year_month, length)

# Month-over-month growth
mom_growth <- diff(monthly_rev) / head(monthly_rev, -1) * 100

# Revenue by product
prod_rev <- sort(tapply(df$revenue, df$product, sum), decreasing = TRUE)
prod_pct <- 100 * prod_rev / total_rev

# Revenue by region
reg_rev  <- sort(tapply(df$revenue, df$region, sum), decreasing = TRUE)

# Revenue by channel
chan_rev <- sort(tapply(df$revenue, df$channel, sum), decreasing = TRUE)

# Top products by region (cross-tab)
prod_region <- tapply(df$revenue, list(df$product, df$region), sum)
```

---

## Part 4: Formatted Report

```r
# ---- Generate text report ----
generate_report <- function(output_file = "sales_report.txt") {
  
  sink(output_file)   # redirect cat() output to file
  on.exit(sink())     # always restore at end
  
  cat("╔══════════════════════════════════════════════════╗\n")
  cat("║         SALES ANALYSIS REPORT                   ║\n")
  cat(sprintf("║         Generated: %-28s ║\n", 
              format(Sys.time(), "%Y-%m-%d %H:%M")))
  cat("╚══════════════════════════════════════════════════╝\n\n")
  
  # Executive summary
  cat("EXECUTIVE SUMMARY\n")
  cat(strrep("─", 50), "\n")
  cat(sprintf("Total Revenue:     $%s\n", 
              formatC(total_rev, format="f", digits=2, big.mark=",")))
  cat(sprintf("Total Orders:      %s\n",
              formatC(nrow(df), format="d", big.mark=",")))
  cat(sprintf("Total Units Sold:  %s\n",
              formatC(total_qty, format="d", big.mark=",")))
  cat(sprintf("Average Order:     $%.2f\n", avg_order))
  cat(sprintf("Period:            %s to %s\n",
              format(min(df$date), "%B %Y"),
              format(max(df$date), "%B %Y")))
  
  # Monthly trend
  cat("\n\nMONTHLY REVENUE TREND\n")
  cat(strrep("─", 50), "\n")
  max_rev <- max(monthly_rev)
  for (i in seq_along(monthly_rev)) {
    month <- names(monthly_rev)[i]
    rev   <- monthly_rev[i]
    bar_len <- round(30 * rev / max_rev)
    bar  <- strrep("█", bar_len)
    cat(sprintf("  %s  $%10s  %s\n",
                month,
                formatC(rev, format="f", digits=0, big.mark=","),
                bar))
  }
  
  # Growth
  avg_growth <- mean(mom_growth)
  cat(sprintf("\nAverage MoM growth: %+.1f%%\n", avg_growth))
  best_month <- names(which.max(mom_growth))
  cat(sprintf("Best growth month:  %s (%+.1f%%)\n",
              best_month, max(mom_growth)))
  
  # Products
  cat("\n\nPRODUCT PERFORMANCE\n")
  cat(strrep("─", 50), "\n")
  cat(sprintf("%-15s  %12s  %7s\n", "Product", "Revenue", "Share"))
  cat(strrep("─", 38), "\n")
  for (prod in names(prod_rev)) {
    cat(sprintf("%-15s  $%10s  %6.1f%%\n",
                prod,
                formatC(prod_rev[prod], format="f", digits=2, big.mark=","),
                prod_pct[prod]))
  }
  
  # Regions
  cat("\n\nREGIONAL BREAKDOWN\n")
  cat(strrep("─", 50), "\n")
  for (reg in names(reg_rev)) {
    pct <- 100 * reg_rev[reg] / total_rev
    bar <- strrep("█", round(pct / 2))
    cat(sprintf("  %-8s  $%10s  %5.1f%%  %s\n",
                reg,
                formatC(reg_rev[reg], format="f", digits=2, big.mark=","),
                pct, bar))
  }
  
  cat("\n\n[End of Report]\n")
}

generate_report()
cat("Report written to sales_report.txt\n")
```

---

## Part 5: Plots

```r
# ---- Visualizations (base R graphics) ----
generate_plots <- function(pdf_file = "sales_plots.pdf") {
  
  pdf(pdf_file, width = 10, height = 7)
  par(mfrow = c(1, 1))
  
  # 1. Monthly revenue line chart
  months_idx <- seq_along(monthly_rev)
  plot(months_idx, monthly_rev / 1000,
       type = "b", pch = 19, col = "steelblue", lwd = 2,
       main = "Monthly Revenue",
       xlab = "Month", ylab = "Revenue ($000)",
       xaxt = "n")
  axis(1, at = months_idx[seq(1, length(months_idx), by = 3)],
       labels = names(monthly_rev)[seq(1, length(monthly_rev), by = 3)],
       las = 2, cex.axis = 0.7)
  grid()
  
  # 2. Product revenue bar chart
  par(mar = c(8, 6, 4, 2))
  bp <- barplot(prod_rev / 1000,
                main = "Revenue by Product",
                ylab = "Revenue ($000)",
                col  = c("steelblue","coral","seagreen","goldenrod","mediumpurple"),
                las  = 2, cex.names = 0.8)
  grid(nx = 0)
  
  # 3. Region pie chart
  par(mar = c(3, 3, 4, 3))
  pie(reg_rev,
      labels = paste0(names(reg_rev), "\n",
                      round(100 * reg_rev / sum(reg_rev), 1), "%"),
      main   = "Revenue by Region",
      col    = c("steelblue","coral","seagreen","goldenrod"))
  
  # 4. Channel comparison
  par(mar = c(5, 6, 4, 2))
  barplot(chan_rev / 1000,
          main   = "Revenue by Channel",
          ylab   = "Revenue ($000)",
          col    = c("steelblue","coral","seagreen"),
          names.arg = names(chan_rev),
          horiz  = TRUE, las = 1)
  
  dev.off()
  cat(sprintf("Plots written to %s\n", pdf_file))
}

generate_plots()

cat("\nPipeline complete!\n")
cat("Files generated:\n")
cat("  - sales_data.csv\n")
cat("  - sales_report.txt\n")
cat("  - sales_plots.pdf\n")
```

Open `sales_report.txt` in a text editor and `sales_plots.pdf` in a PDF viewer to see your complete analysis.

---

## What Comes Next

You now have working knowledge of R. The next level:

**The Tidyverse** — `dplyr`, `ggplot2`, `tidyr`, `purrr`. A coherent set of packages that many R users use for data analysis. Powerful, well-documented, widely taught.

```r
library(dplyr)
library(ggplot2)

# dplyr: pipe-based data manipulation
sales %>%
  group_by(product) %>%
  summarise(total_revenue = sum(revenue)) %>%
  arrange(desc(total_revenue))

# ggplot2: layered graphics
ggplot(sales, aes(x = date, y = revenue, color = product)) +
  geom_line() +
  facet_wrap(~region) +
  theme_minimal()
```

**R Markdown** — write documents that mix prose with R code. The code runs when you render the document; the output (tables, plots) is embedded. This is how professional R analysts deliver work.

**Shiny** — build interactive web applications in R. Your analysis becomes a dashboard someone can use without touching code.

**Statistical modeling** — base R's `lm()`, `glm()`, and packages like `caret`, `tidymodels`, `mlr3` for machine learning.

---

## Final Exercises

**1. Extend the pipeline**

Add to the sales pipeline:
- Customer retention metric (assume each unique month+channel is a "customer segment")
- Seasonal analysis: compare Q1/Q2/Q3/Q4
- A "hotspot" table: product × region with highest revenue combination

**2. R Markdown report**

Install `rmarkdown`. Create `report.Rmd` that runs the full sales pipeline and renders to HTML:
```r
install.packages("rmarkdown")
rmarkdown::render("report.Rmd")
```

**3. Interactive with Shiny**

```r
install.packages("shiny")
```

Build a minimal Shiny app with:
- A slider for date range
- A dropdown for product filter  
- A reactive bar chart that updates when filters change

**4. Your own project**

Apply everything to a real dataset. Find something at:
- `data()` — R's built-in datasets
- [Kaggle](https://www.kaggle.com/datasets) — real-world data
- Your own work

Write a complete pipeline: load → clean → analyze → visualize → report.

---

## The Programs You've Written

| Chapter | Program |
|---------|---------|
| 1 | calculator.R, wordcount.R |
| 2 | unit_converter.R |
| 3 | guess.R, times_table.R, primes.R |
| 4 | stats_toolkit.R |
| 5 | grades.R |
| 6 | word_freq.R |
| 7 | csv_processor.R |
| 8 | eda.R |
| 9 | config.R |
| 10 | batch_cleaner.R |
| 11 | dataset_class.R |
| 12 | sales_pipeline.R |

Twelve programs. Every concept demonstrated through something that runs. That's how you learn to program.
