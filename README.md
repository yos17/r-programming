# R Programming

*Learn R by writing programs that build on each other.*

---

This course teaches R the way Kernighan taught C — not by cataloguing features, but by writing programs. Every chapter grows a single tool. By the end, you have something real.

The environment is RStudio for interactive work, plus the command line for running scripts.

## The Red Thread: analyze.R

The book has one project: `analyze.R`, a command-line data analysis tool. Each chapter adds one capability to it.

By Chapter 12 it does this:

```
Rscript analyze.R data.csv --group dept --stat mean,sd --filter "salary>50000" --output report.txt
```

150 lines. Reads any CSV, computes statistics, filters rows, groups data, writes a report.

## How Each Chapter Works

1. **Start with the problem**: "Here's what we have so far. It can't do X yet. Let's fix that."
2. **Show the minimal code first** — 5–15 lines — then build up.
3. **Show what changed**: the diff from last chapter.
4. **Exercises connect forward**: the last exercise in each chapter produces something needed in the next.

## Chapters

| # | Title | analyze.R gains |
|---|-------|----------------|
| 1 | Getting Started | Reads filename, prints "Hello" |
| 2 | Types and Expressions | Parses types, converts units |
| 3 | Control Flow | Parses filter expressions |
| 4 | Functions | Computes mean, sd, median |
| 5 | Vectors and Arrays | Vectorized ops, NA handling |
| 6 | Strings | Regex filter operators `~`, `!~` |
| 7 | I/O and Files | Reads real CSVs, writes reports |
| 8 | Data Frames | Groups with `--group` |
| 9 | Lists and Environments | Config system |
| 10 | The Apply Family | Multiple files, `lapply` |
| 11 | Packages and OOP | Dataset S3 class |
| 12 | Real Tools | Final tool, 150 lines |
| 13 | Debugging | Break it 3 ways, fix it |

## Setup

1. Install R from [r-project.org](https://www.r-project.org)
2. Install RStudio from [posit.co](https://posit.co/downloads/)
3. Open RStudio. Work in the **Script editor** (top-left); run code in the **Console** (bottom-left).

**Run a line**: `Ctrl+Enter` (Windows/Linux) or `Cmd+Enter` (Mac)  
**Run the whole script**: `Ctrl+Shift+Enter`  
**Run from terminal**: `Rscript script.R`

## RStudio Layout

```
┌──────────────────┬──────────────────┐
│   Script Editor  │   Environment /  │
│   (write here)   │   Files / Plots  │
├──────────────────┼──────────────────┤
│   Console        │   Help / Viewer  │
│   (runs here)    │                  │
└──────────────────┴──────────────────┘
```

---

*Type every example yourself. Don't copy-paste. The act of typing is part of the learning.*
