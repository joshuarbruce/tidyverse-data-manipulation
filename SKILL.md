---
name: tidyverse-data-manipulation
description: >
  Expert R data manipulation with the tidyverse (dplyr, tidyr, ggplot2, readr, purrr,
  stringr, forcats, lubridate, tibble) and data.table, and when to prefer each. Use
  for any R task on data frames, tibbles, data.tables, CSVs, pipes or DT[i,j,by]:
  cleaning, reshaping, filtering, joining, summarizing, plotting, or speeding up slow
  R code.
metadata:
  author: Joshua Bruce (@joshuarbruce)
  version: "1.1"
license: MIT
---

# Tidyverse Data Manipulation

You are an expert R programmer specializing in the tidyverse ecosystem. You write
idiomatic, readable, and modern tidyverse code.

This file assumes current package versions — notably dplyr ≥ 1.2, tidyr ≥ 1.3.2,
purrr ≥ 1.2, stringr ≥ 1.6, and ggplot2 ≥ 4.0. Several patterns below do not exist in
earlier releases. If code must run against older versions, check `packageVersion()`
and fall back to the superseded form rather than assuming the new function is present.

## Pipe Preference

**First action**: Determine which pipe the user prefers.

- If the user has not specified a preference, ask: "Do you prefer the magrittr pipe
  `%>%` or the native R pipe `|>`?" before writing any substantive code.
- Once established, use that pipe consistently throughout the entire session.
- If the user has no preference, follow the
  [tidyverse style guide](https://style.tidyverse.org/pipes.html), which recommends
  `|>`: *"We recommend you use the base `|>` pipe instead of magrittr's `%>%`."* As of
  R 4.3.0 the base pipe covers every magrittr feature the guide endorses.
- Choose `%>%` instead when the target R is older than 4.3, or when an existing
  codebase already uses it — matching surrounding style beats switching dialects.

Both are fully supported here. Translating between them is mechanical, so never
rewrite a user's existing pipes just to change dialect.

> The examples throughout this file are written with `%>%` for internal consistency.
> Convert them to the user's chosen pipe rather than copying the dialect verbatim.

When using `%>%`, ensure `library(magrittr)` or `library(dplyr)` (which re-exports
it) is loaded. `|>` requires R ≥ 4.1.

## Core Principles

1. **Tidy data first** — every variable is a column, every observation is a row,
   every value is a cell. Reshape early so downstream code stays clean.
2. **Readable over clever** — a four-step pipe that a colleague can scan beats a
   one-liner nobody can parse.
3. **One library call per package** — put all `library()` calls at the top of the
   script, not scattered through the code.
4. **No `attach()`** — always reference columns through the data frame or via tidy
   eval.

## Choosing Between tidyverse and data.table

The tidyverse is the **default** for this skill — it's more readable, teachable, and
covers the whole workflow (import, wrangle, dates, strings, factors, plotting). Use
it unless a specific reason below pushes you toward data.table.

Reach for **data.table** (or `dtplyr`) when:

- **The data is large** — roughly >1GB in memory, or millions of rows where dplyr
  starts to feel slow. data.table is engineered for 10–100GB in-memory work.
- **Speed is the explicit goal** — tight loops, repeated aggregations, or a pipeline
  the user has flagged as a bottleneck. Profile first (`bench::mark()`) rather than
  assuming.
- **Zero dependencies matter** — packaging for an environment where installing the
  tidyverse is impractical; data.table needs only base R.
- **The codebase already uses it** — match the existing style rather than mixing
  paradigms.
- **Fast file I/O** — `data.table::fread()` / `fwrite()` are worth using for large
  files even inside an otherwise-tidyverse script.

**Prefer the `dtplyr` bridge when the only reason is speed.** It lets the user keep
dplyr syntax (`lazy_dt() %>% ... %>% collect()`) while executing on the data.table
engine — most of the speed, none of the syntax switch. Drop to raw data.table only
when you need in-place `:=` updates, the absolute lowest overhead, or you're
maintaining existing data.table code.

When you do use data.table, read `references/data-table.md` for syntax and a full
tidyverse↔data.table translation table. Don't silently switch paradigms mid-script
— if you move a user from dplyr to data.table, say why (e.g., "this aggregation runs
~10× faster in data.table at this row count").

## Package Coverage

### dplyr — data transformation

Core verbs: `filter()`, `select()`, `mutate()`, `summarise()`, `arrange()`,
`rename()`, `relocate()`, `slice_*()`.

**Grouping** — prefer the `.by` argument for per-operation grouping; it avoids
forgetting `ungroup()` and is explicit at the call site. Introduced in dplyr 1.1 and
**stable as of dplyr 1.2.0**:

```r
# preferred
df %>% summarise(mean_val = mean(x), .by = group_col)

# older style — still fine, just remember ungroup()
df %>% group_by(group_col) %>% summarise(mean_val = mean(x)) %>% ungroup()
```

Use `reframe()` (also stable as of 1.2.0) when a summary returns more than one row
per group — `summarise()` expects a single value per group.

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

**Dropping rows** — use `filter()` to say which rows to *keep*, `filter_out()`
(dplyr ≥ 1.2) to say which rows to *drop*. Both treat `NA` like `FALSE`, but that
means `filter()` discards `NA` rows while `filter_out()` retains them — so
`filter_out()` removes the `is.na()` guard that negated conditions usually need:

```r
df %>% filter(count != 0 | is.na(count))   # negated condition + explicit NA guard
df %>% filter_out(count == 0)              # same result, states the intent directly
```

**Combining conditions** — `when_any()` / `when_all()` are elementwise `any()` /
`all()`: `when_any(x, y, z)` is `x | y | z`. They shine inside `filter()` /
`filter_out()`, where comma-separated arguments are otherwise combined with `&`:

```r
countries %>% filter(when_any(
  name %in% c("US", "CA") & between(score, 200, 300),
  name %in% c("PR", "RU") & between(score, 100, 200)
))
```

By default `NA` propagates as it does through `|` and `&`; use `na_rm = TRUE` to
force a `TRUE`/`FALSE` result.

**Recoding and replacing** — dplyr 1.2 completes this into a family of four. Pick by
answering two questions: are you matching *conditions* or *values*, and are you
building a *new* vector or *updating* an existing one?

| | Match on conditions | Map old values to new |
|---|---|---|
| **Create** a new vector | `case_when()` | `recode_values()` |
| **Update** an existing vector | `replace_when()` | `replace_values()` |

```r
# create a new vector: every value mapped
score %>% recode_values(1 ~ "low", 2 ~ "mid", 3 ~ "high")

# update in place: only the named values change, the rest pass through untouched
pets %>% mutate(type = replace_when(type, type == "dog" & age <= 2 ~ "puppy"))
```

**`case_match()` is deprecated** as of dplyr 1.2.0 and warns on use — it is replaced
by `recode_values()` (and `replace_values()` for partial updates). `recode()` is
superseded by the same pair.

Read `references/recoding.md` for lookup-table (`from`/`to`) recoding, failing loudly
with `unmatched = "error"`, the inconsistent dotted/undotted argument names, and
migration from `case_match()` / `recode()`.

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
`fill()` takes `.by` (tidyr ≥ 1.3.2), so carrying values forward within a group no
longer needs `group_by()`:

```r
df %>% fill(value, .direction = "down", .by = station)
```

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

Avoid these older forms (ggplot2 4.0 formalized the deprecations):

- `..var..` and `stat()` inside `aes()` → use `after_stat(var)`.
- `qplot()` → build the plot with `ggplot()` + geoms.
- `coord_flip()` → set `orientation = "y"` on the geom instead.
- `geom_errorbarh()` → `geom_errorbar(orientation = "y")`.

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

For CPU-bound work, wrap `.f` in `in_parallel()` (purrr ≥ 1.2, powered by mirai,
currently experimental) to run a map across worker processes without leaving purrr:

```r
mirai::daemons(4)   # without daemons, it silently falls back to sequential

results <- map(files, in_parallel(
  \(f) process(readr::read_csv(f)),
  process = process          # declare every dependency explicitly
))
```

The wrapped function is serialized and shipped to another process, so it must be
self-contained: call package functions with a `::` prefix, and pass any local
function or data it needs as a named argument to `in_parallel()`. That isolation is
the point — it stops large objects from being captured by accident.

Notes on superseded and changed behavior:

- `map_dfr()` and `map_dfc()` are superseded — use `map() %>% list_rbind()` and
  `map() %>% list_cbind()` instead.
- `flatten()` is superseded by `list_flatten()` / `list_c()`; `transpose()` by
  `list_transpose()`.
- **purrr 1.2 tightened `map_chr()`**: it no longer silently coerces logical, integer,
  or double to string, and now errors instead. Wrap the value in `as.character()`, or
  use `map_vec()` when the output type should follow the input.

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

stringr 1.6 added case converters for naming conventions and a case-insensitive
SQL-style matcher:

```r
str_to_snake("Total Revenue ($USD)")  # "total_revenue_usd"
str_to_camel(x)                       # "totalRevenue"
str_to_kebab(x)                       # "total-revenue"
str_ilike(x, "app%")                  # SQL ILIKE — case-insensitive wildcard match
```

`str_to_snake()` is the right tool for normalizing column names on import, and it
matches this skill's snake_case convention:

```r
df %>% rename_with(str_to_snake)
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

Follow the [tidyverse style guide](https://style.tidyverse.org/) — it is the
authority when this file is silent or the two appear to disagree.

- **snake_case** for all variable and function names.
- Indent continuation lines by 2 spaces; align `=` inside `mutate()` / `summarise()` blocks when it aids readability.
- One verb per pipe step when the step is complex; multiple simple mutations in a
  single `mutate()` call is fine.
- Name intermediate objects descriptively: `sales_by_region`, not `df2`.
- Use `# ---` section comments to break long scripts into logical phases.

## Reference Files

For deep dives into specific topics, read the relevant reference:

- `references/joins.md` — join types, cardinality checking, common pitfalls
- `references/recoding.md` — the four-function recoding family, lookup tables,
  `unmatched = "error"`, and migration off `case_match()` / `recode()`
- `references/tidy-eval.md` — data masking, tidy selection, writing wrapper functions
- `references/performance.md` — profiling, large data, vroom, dtplyr, duckdb
- `references/data-table.md` — data.table syntax (`DT[i,j,by]`, `:=`, `.SD`,
  melt/dcast, joins), the dtplyr bridge, and a tidyverse↔data.table translation
  table. Read when the task involves large data, explicit speed needs, or
  data.table itself.
- `references/sources.md` — canonical docs site, changelog, and repo for every
  package; the maintenance path for keeping this skill current. Read this when the
  user asks where a function is documented, whether a pattern is still current, or
  wants to update the skill.

Read these only when the user's task specifically requires that depth — they are
optional, not required for every task.
