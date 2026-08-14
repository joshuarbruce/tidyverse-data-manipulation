---
name: tidyverse-data-manipulation
description: >
  Expert R data manipulation across the tidyverse and data.table, including when to
  prefer each. Use for any R task that cleans or reshapes a data frame, joins tables,
  summarizes by group, or plots results. Covers dplyr and tidyr wrangling, readr
  import, purrr iteration, stringr and lubridate cleanup, forcats factors, tibble
  basics, ggplot2 charts, and the dtplyr bridge for large data.
metadata:
  author: Joshua Bruce (@joshuarbruce)
  version: "2.0"
license: MIT
---

# Tidyverse Data Manipulation

Write idiomatic, readable, modern tidyverse code. Much of the R on blog posts and
Stack Overflow uses superseded APIs; this guide encodes the current approach and
points at deeper references for each topic.

Assumes current packages — notably **dplyr ≥ 1.2, tidyr ≥ 1.3.2, purrr ≥ 1.2,
stringr ≥ 1.6, ggplot2 ≥ 4.0**. Several patterns here do not exist in earlier
releases. If code must run against older versions, check `packageVersion()` and fall
back to the superseded form rather than assuming the new function is present.

## Reference files

Read the reference for the topic at hand rather than working from this page alone.
For a task spanning several topics, read several.

| Topic | Reference | When to consult |
|---|---|---|
| **Grouping & columns** | [grouping.md](references/grouping.md) | `.by`, `group_by`, `across`, `pick`, `reframe`, row-wise work, counting |
| **Joins** | [joins.md](references/joins.md) | Merging tables, `join_by()`, cardinality, unmatched rows |
| **Filtering & recoding** | [filtering-and-recoding.md](references/filtering-and-recoding.md) | `filter_out`, `when_any`/`when_all`, `case_when`, `recode_values`, `replace_values`, `replace_when`, lookup tables |
| **Reshaping** | [reshaping.md](references/reshaping.md) | Pivoting, nesting, splitting columns, missing values |
| **Import** | [import.md](references/import.md) | `read_csv`, `col_types`, parse failures, Excel/SPSS, building tibbles |
| **Iteration** | [iteration.md](references/iteration.md) | `map_*`, `walk`, `list_rbind`, parallel maps |
| **Strings** | [strings.md](references/strings.md) | `str_*`, regex, case conversion, cleaning column names |
| **Factors & dates** | [factors-and-dates.md](references/factors-and-dates.md) | `fct_*` ordering and lumping, date parsing, rounding, intervals |
| **Visualization** | [visualization.md](references/visualization.md) | ggplot2 layers, facets, scales, themes, 4.0 deprecations |
| **Tidy evaluation** | [tidy-eval.md](references/tidy-eval.md) | Writing functions that wrap dplyr verbs, `{{ }}`, `.data[[ ]]` |
| **data.table** | [data-table.md](references/data-table.md) | `DT[i,j,by]`, `:=`, `.SD`, keys, the dtplyr bridge |
| **Performance** | [performance.md](references/performance.md) | Profiling, large data, duckdb, parallelism |
| **Sources** | [sources.md](references/sources.md) | Where each function is documented; keeping this skill current |

## Core principles

1. **Tidy data first** — one variable per column, one observation per row. Reshape
   early so downstream code stays simple.
2. **Readable over clever** — a four-step pipe a colleague can scan beats a one-liner
   nobody can parse.
3. **Fail loudly** — prefer arguments that error on surprises (`relationship`,
   `unmatched = "error"`) over defaults that silently absorb them.
4. **Match the surrounding code** — consistency with an existing codebase beats
   importing a different style.
5. All `library()` calls at the top of the script; never `attach()`.

## Choosing tidyverse or data.table

The tidyverse is the default: more readable, and it covers the whole workflow. Reach
for **data.table** when data is large (roughly >1GB, or millions of rows), when speed
is the explicit goal, when zero dependencies matter, or when the codebase already uses
it. `data.table::fread()`/`fwrite()` are worth using for large files in any script.

**When the only reason is speed, prefer the `dtplyr` bridge** — `lazy_dt()` keeps
dplyr syntax on the data.table engine. Drop to raw data.table only for in-place `:=`
updates, the lowest possible overhead, or an existing data.table codebase.

Profile before switching (`bench::mark()`), and never switch paradigms silently — say
why, e.g. "this aggregation runs ~10× faster in data.table at this row count".

## Pipes

**First action**: determine which pipe the user prefers, then use it consistently for
the whole session.

- If unstated, ask: magrittr `%>%` or native `|>`?
- With no preference, follow the [tidyverse style
  guide](https://style.tidyverse.org/pipes.html), which recommends `|>`. As of R 4.3.0
  the base pipe covers every magrittr feature the guide endorses.
- Choose `%>%` when the target R is older than 4.3, or the codebase already uses it.

Translating between them is mechanical — never rewrite a user's pipes just to change
dialect. Examples in these files use `%>%`; convert them to the user's choice.

## Quick reference

**Grouping** — `.by` for a single verb, `group_by()` when grouping spans several.
`.by` always returns ungrouped data, so never follow it with `ungroup()`.

```r
df %>% summarise(total = sum(x), .by = c(region, year))
```

**Joins** — always `join_by()`; declare cardinality to catch surprises.

```r
left_join(orders, customers, by = join_by(customer_id), relationship = "many-to-one")
```

**Dropping rows** — `filter()` says what to keep, `filter_out()` what to drop.
`filter_out()` retains `NA` rows, removing the `is.na()` guard negation usually needs.

```r
df %>% filter_out(count == 0)     # not: filter(count != 0 | is.na(count))
```

**Recoding** — pick by two questions: conditions or values, new vector or update?

| | Match on conditions | Map old values to new |
|---|---|---|
| **Create** a new vector | `case_when()` | `recode_values()` |
| **Update** an existing vector | `replace_when()` | `replace_values()` |

**Iteration** — `map()` plus an explicit bind; `map_dfr()` is superseded.

```r
map(files, read_csv) %>% list_rbind()
```

**Column names** — normalize on import: `df %>% rename_with(str_to_snake)`.

### Superseded and deprecated

| Avoid | Use |
|---|---|
| `case_match()`, `recode()` | `recode_values()` / `replace_values()` |
| `map_dfr()`, `map_dfc()` | `map() %>% list_rbind()` / `list_cbind()` |
| `separate()` | `separate_wider_delim()` / `separate_wider_regex()` |
| `by = c("a" = "b")` | `join_by(a == b)` |
| `coord_flip()` | `orientation = "y"` on the geom |
| `..var..`, `stat()`, `qplot()` | `after_stat()`, `ggplot()` + geoms |
| `fct_explicit_na()` | `fct_na_value_to_level()` |
| `size` for line width | `linewidth` |

`case_match()` warns as of dplyr 1.2.0. `map_chr()` no longer coerces numbers to
strings and now errors instead — wrap in `as.character()` or use `map_vec()`.

## Worked example

```r
library(tidyverse)

result <- raw_data %>%
  # parse and clean
  mutate(
    date     = ymd(date_string),
    amount   = parse_number(amount_str),
    category = str_to_lower(str_trim(category))
  ) %>%
  # drop rows we don't want, NA-safely
  filter_out(is.na(amount)) %>%
  filter(date >= ymd("2023-01-01")) %>%
  # enrich
  left_join(lookup_table, by = join_by(category), unmatched = "error") %>%
  # aggregate
  summarise(
    total = sum(amount),
    n     = n(),
    .by   = c(category, label)
  ) %>%
  arrange(desc(total))
```

## Style

Follow the [tidyverse style guide](https://style.tidyverse.org/) — it is the authority
wherever this skill is silent. In brief: snake_case throughout, descriptive
intermediate names (`sales_by_region`, not `df2`), one verb per pipe step when the
step is complex, and `TRUE`/`FALSE` never `T`/`F`.
