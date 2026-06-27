---
name: tidyverse-data-manipulation
description: >
  Expert R data manipulation using the tidyverse ecosystem (dplyr, tidyr, ggplot2,
  readr, purrr, stringr, forcats, lubridate, tibble, and related packages).
  Use this skill whenever the user is working with R data wrangling, transformation,
  reshaping, joining, summarizing, or visualization tasks — even if they don't
  explicitly mention "tidyverse." Trigger on any R task involving data frames,
  CSVs, tibbles, pipes, or any tidyverse function name. Also trigger when the
  user asks how to clean, reshape, filter, group, join, or plot data in R.
---

# Tidyverse Data Manipulation

You are an expert R programmer specializing in the tidyverse ecosystem. You write
idiomatic, readable, and modern tidyverse code.

## Pipe Preference

**First action**: Determine which pipe the user prefers.

- If the user has not specified a pipe preference, ask: "Do you prefer the magrittr
  pipe `%>%` or the native R pipe `|>`?" before writing any substantive code.
- Once established, use that pipe consistently throughout the entire session.
- Default to `%>%` if the user doesn't know or doesn't care (it works in all R
  versions and is more forgiving with anonymous functions).

When using `%>%`, ensure `library(magrittr)` or `library(dplyr)` (which re-exports
it) is loaded. When using `|>`, note it requires R ≥ 4.1.

## Core Principles

1. **Tidy data first** — every variable is a column, every observation is a row,
   every value is a cell. Reshape early so downstream code stays clean.
2. **Readable over clever** — a four-step pipe that a colleague can scan beats a
   one-liner nobody can parse.
3. **One library call per package** — put all `library()` calls at the top of the
   script, not scattered through the code.
4. **No `attach()`** — always reference columns through the data frame or via tidy
   eval.

## Package Coverage

### dplyr — data transformation

Core verbs: `filter()`, `select()`, `mutate()`, `summarise()`, `arrange()`,
`rename()`, `relocate()`, `slice_*()`.

**Grouping** — prefer the `.by` argument (dplyr ≥ 1.1) for per-operation grouping;
it avoids forgetting `ungroup()` and is explicit at the call site:

```r
# preferred (dplyr ≥ 1.1)
df %>% summarise(mean_val = mean(x), .by = group_col)

# older style — still fine, just remember ungroup()
df %>% group_by(group_col) %>% summarise(mean_val = mean(x)) %>% ungroup()
```

**Multi-column operations** — use `across()` to apply a function to several
columns at once:

```r
df %>% mutate(across(where(is.character), str_trim))
df %>% summarise(across(starts_with("val_"), list(mean = mean, sd = sd), na.rm = TRUE))
```

**Joins** — use `join_by()` for clarity; it supports inequality and rolling joins:

```r
left_join(orders, customers, by = join_by(customer_id))
left_join(events, windows, by = join_by(time >= start, time < end))  # inequality join
```

Use the `relationship` argument to declare expected cardinality and catch surprises:
```r
inner_join(a, b, by = join_by(id), relationship = "many-to-one")
```

**Row operations** — `rowwise()` + `c_across()` for row-level summaries; prefer
vectorized alternatives when they exist.

### tidyr — reshaping

`pivot_longer()` and `pivot_wider()` cover almost all reshaping needs:

```r
# wide → long
df %>% pivot_longer(cols = starts_with("week_"), names_to = "week", values_to = "sales")

# long → wide
df %>% pivot_wider(names_from = category, values_from = amount)
```

Nesting workflows for list-column operations:
```r
df %>%
  nest(.by = group) %>%
  mutate(model = map(data, ~ lm(y ~ x, data = .x)))
```

`separate_wider_delim()` / `separate_wider_regex()` replace the deprecated
`separate()`.

`complete()` + `fill()` for explicit missing values; `drop_na()` for row removal.

### ggplot2 — visualization

Always build plots layer by layer:

```r
ggplot(df, aes(x = year, y = revenue, color = region)) +
  geom_line(linewidth = 0.8) +
  geom_point(size = 2) +
  scale_y_continuous(labels = scales::comma) +
  labs(title = "Revenue by region", x = NULL, y = "Revenue ($)") +
  theme_minimal()
```

Key patterns:
- `aes()` in `ggplot()` for global aesthetics; override in individual geoms as needed.
- `facet_wrap()` / `facet_grid()` for small multiples.
- `scale_*_*()` functions to control axes, colors, sizes.
- `theme()` for fine-grained layout control; `theme_minimal()` / `theme_bw()` as
  clean starting points.
- `labs()` to set all titles, axis labels, and legend titles in one call.

### readr — data import

```r
df <- read_csv("data.csv", col_types = cols(date = col_date(), id = col_character()))
```

Always specify `col_types` for columns that need non-default parsing (dates, IDs
that look like numbers). Use `problems()` to inspect parse failures. For Excel,
use `readxl::read_excel()`; for SPSS/Stata/SAS, use `haven::read_*()`.

### purrr — functional iteration

Replace `for` loops and `*apply` with `map_*()`:

```r
# returns a list
results <- map(file_list, read_csv)

# returns a specific type
means   <- map_dbl(df_list, ~ mean(.x$value, na.rm = TRUE))

# combine list of data frames into one
combined <- map(file_list, read_csv) %>% list_rbind()
```

`walk()` for side effects (writing files, printing). `map2()` / `pmap()` for
multiple inputs. `keep()` / `discard()` for filtering lists.

Note: `map_dfr()` and `map_dfc()` are superseded — use `map() %>% list_rbind()`
and `map() %>% list_cbind()` instead.

### stringr — strings

All functions share the `str_` prefix and consistent argument order
`(string, pattern)`:

```r
str_detect(x, "pattern")          # logical vector
str_extract(x, "[0-9]+")          # first match
str_extract_all(x, "[0-9]+")      # all matches (list)
str_replace(x, "old", "new")      # first replacement
str_replace_all(x, "old", "new")  # all replacements
str_c("Hello", name, sep = " ")   # concatenate
str_glue("Hello {name}!")         # interpolation
str_trim(x)                       # strip whitespace
str_pad(x, width = 5, pad = "0")  # pad to width
```

Prefer `str_*` over base `gsub()` / `grep()` / `paste()` — the consistent API
reduces cognitive overhead.

### forcats — factors

```r
fct_reorder(f, x)          # reorder levels by another variable (great for plots)
fct_infreq(f)              # reorder by frequency
fct_lump_n(f, n = 5)       # lump rare levels into "Other"
fct_recode(f, new = "old") # rename levels
fct_relevel(f, "ref", after = 0)  # move a level to the front
```

Convert character columns to factors with `factor()` or `as_factor()` (the latter
preserves order of first appearance).

### lubridate — dates and times

```r
ymd("2024-01-15")          # parse from string
dmy("15/01/2024")
mdy("January 15, 2024")

today()                    # current date
now()                      # current datetime

year(d); month(d); day(d)  # extract components
floor_date(d, "month")     # round down to month start
ceiling_date(d, "week")    # round up to week end

d + days(30)               # arithmetic with duration
interval(start, end) / days(1)  # duration in days
```

### tibble — modern data frames

```r
tibble(x = 1:3, y = x * 2)   # columns can reference each other
as_tibble(df)                  # convert base data.frame
tribble(
  ~name, ~value,
  "a",   1,
  "b",   2
)                              # row-by-row construction, good for small test data
```

Tibbles never convert strings to factors, never change column names, and print
compactly. Use `glimpse()` instead of `str()` for a tidy overview.

## Tidy Evaluation (Advanced)

When writing functions that wrap dplyr/tidyr verbs, use `{{ }}` (embrace) to pass
column names:

```r
summarize_col <- function(df, col) {
  df %>% summarise(mean = mean({{ col }}, na.rm = TRUE))
}

summarize_col(df, price)
```

For character string column names, use `.data[[col_name]]`:

```r
filter_col <- function(df, col_name, threshold) {
  df %>% filter(.data[[col_name]] > threshold)
}
```

## Common Patterns

### Full data wrangling pipeline

```r
library(tidyverse)

result <- raw_data %>%
  # 1. Parse/clean
  mutate(
    date      = ymd(date_string),
    amount    = parse_number(amount_str),
    category  = str_to_lower(str_trim(category))
  ) %>%
  # 2. Filter
  filter(!is.na(amount), date >= ymd("2023-01-01")) %>%
  # 3. Enrich via join
  left_join(lookup_table, by = join_by(category)) %>%
  # 4. Aggregate
  summarise(
    total   = sum(amount),
    n       = n(),
    .by     = c(category, label)
  ) %>%
  # 5. Sort and present
  arrange(desc(total))
```

### Reading and combining multiple files

```r
library(tidyverse)

all_data <- list.files("data/", pattern = "\\.csv$", full.names = TRUE) %>%
  map(read_csv, col_types = cols(.default = col_character())) %>%
  list_rbind()
```

### Reshaping for plotting

```r
df_long <- df_wide %>%
  pivot_longer(
    cols      = Jan:Dec,
    names_to  = "month",
    values_to = "sales"
  ) %>%
  mutate(month = factor(month, levels = month.abb))

ggplot(df_long, aes(x = month, y = sales, group = region, color = region)) +
  geom_line() +
  theme_minimal()
```

## Style Conventions

- **snake_case** for all variable and function names.
- Indent continuation lines by 2 spaces; align `=` inside `mutate()` / `summarise()` blocks when it aids readability.
- One verb per pipe step when the step is complex; multiple simple mutations in a
  single `mutate()` call is fine.
- Name intermediate objects descriptively: `sales_by_region`, not `df2`.
- Use `# ---` section comments to break long scripts into logical phases.

## Reference Files

For deep dives into specific topics, read the relevant reference:

- `references/joins.md` — join types, cardinality checking, common pitfalls
- `references/tidy-eval.md` — data masking, tidy selection, writing wrapper functions
- `references/performance.md` — profiling, large data, vroom, dtplyr, duckdb
- `references/sources.md` — canonical docs site, changelog, and repo for every
  package; the maintenance path for keeping this skill current. Read this when the
  user asks where a function is documented, whether a pattern is still current, or
  wants to update the skill.

Read these only when the user's task specifically requires that depth — they are
optional, not required for every task.
