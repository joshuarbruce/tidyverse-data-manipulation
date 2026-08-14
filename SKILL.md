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
  version: "2.1"
license: MIT
---

# Tidyverse Data Manipulation

Write idiomatic, readable, modern tidyverse code. Much of the R on blog posts and
Stack Overflow uses superseded APIs; this guide encodes the current approach and
points at deeper references for each topic.

Assumes **R ≥ 4.1** — the examples use the `\(x)` lambda shorthand, which is a syntax
error on older R — and current packages, notably **dplyr ≥ 1.2, tidyr ≥ 1.3.2,
purrr ≥ 1.2, stringr ≥ 1.6, ggplot2 ≥ 4.0**. Several patterns here do not exist in
earlier releases. If code must run against older versions, check `getRversion()` and
`packageVersion()`, and fall back to the superseded form rather than assuming the new
function is present.

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
for **data.table** when the data is large enough that dplyr feels slow, when speed is
the explicit goal, when zero dependencies matter, or when the codebase already uses it.
There is no useful size threshold — the crossover depends far more on the operation
than on row count, so measure rather than guess. `data.table::fread()`/`fwrite()` are
worth using for large files in any script.

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
starwars %>% summarise(avg = mean(height, na.rm = TRUE), .by = c(species, gender))
```

`group_by()` is the opposite: grouping is attached to the data frame and **every later
verb inherits it until `ungroup()`**. `summarise()` drops only the *last* grouping
variable, so a two-column grouping leaves the result still grouped — pass
`.groups = "drop"`, or use `.by`. Wrongly-scoped results here are valid, not errors,
so nothing warns you.

**Joins** — always `join_by()`; declare cardinality to catch surprises.

```r
left_join(band_members, band_instruments, by = join_by(name), relationship = "many-to-one")
```

**Dropping rows** — `filter()` says what to keep, `filter_out()` what to drop.
`filter_out()` retains `NA` rows, removing the `is.na()` guard negation usually needs.

```r
starwars %>% filter_out(hair_color == "blond")   # keeps the NA rows too
```

**Recoding** — pick by two questions: conditions or values, new vector or update?

| | Match on conditions | Map old values to new |
|---|---|---|
| **Create** a new vector | `case_when()` | `recode_values()` |
| **Update** an existing vector | `replace_when()` | `replace_values()` |

**Iteration** — `map()` plus an explicit bind; `map_dfr()` is superseded.

```r
map(split(mtcars, mtcars$cyl), \(d) head(d, 2)) %>% list_rbind()
```

**Column names** — normalize on import: `df %>% rename_with(str_to_snake)`.

### Superseded and deprecated

| Avoid | Use |
|---|---|
| Avoid | Use | Status |
|---|---|---|
| `case_match()` | `recode_values()` / `replace_values()` | **Deprecated** — warns |
| `fct_explicit_na()` | `fct_na_value_to_level()` | **Deprecated** |
| `..var..`, `stat()`, `qplot()` | `after_stat()`, `ggplot()` + geoms | **Deprecated** |
| `geom_errorbarh()` | `geom_errorbar(orientation = "y")` | **Deprecated** |
| `recode()` | `recode_values()` | Superseded |
| `map_dfr()`, `map_dfc()` | `map() %>% list_rbind()` / `list_cbind()` | Superseded |
| `separate()`, `extract()` | `separate_wider_delim()` / `separate_wider_regex()` | Superseded |
| `coord_flip()` | `orientation = "y"` on the geom | Superseded |
| `size` for line width | `linewidth` | Renamed in ggplot2 3.4 |

**Deprecated** warns and is scheduled for removal; **superseded** still works silently
and is not going away. Both are worth avoiding in new code, but only the first is
urgent — and reporting one as the other is a common way to be confidently wrong.

Two things that are *not* superseded, despite often being described that way:
`by = c("a" = "b")` in joins (prefer `join_by()` for clarity, but the character form is
documented and supported) and `replace_na()` / `na_if()` (`replace_values()` generalizes
them; it does not replace them).

`map_chr()` no longer coerces numbers to strings and now errors instead — wrap in
`as.character()` or use `map_vec()`.

### Silent failure modes

These produce wrong answers rather than errors. Check for them before trusting a
result, and read the linked reference when the task touches one.

| Trap | Consequence | Detail |
|---|---|---|
| `group_by()` grouping is inherited until `ungroup()` | Later verbs compute per group | [grouping.md](references/grouping.md) |
| `arrange()` ignores grouping unless `.by_group = TRUE` | Sort order wrong before `slice`/`lag` | [grouping.md](references/grouping.md) |
| `slice_max(n = 1)` keeps ties by default | More rows than requested | [grouping.md](references/grouping.md) |
| `NA` join keys **match** by default (unlike SQL) | Spurious matches on missing keys | [joins.md](references/joins.md) |
| Duplicate keys on both sides | Row count silently multiplies | [joins.md](references/joins.md) |
| `unmatched = "error"` guards the *dropped* side, not the `NA` side | Broken lookup table goes undetected | [joins.md](references/joins.md) |
| `NA` conditions drop rows in `filter()` | Data lost without warning | [filtering-and-recoding.md](references/filtering-and-recoding.md) |
| base `ifelse()` strips class | Dates become numbers | [filtering-and-recoding.md](references/filtering-and-recoding.md) |
| `as.numeric()` on a factor | Returns level codes, not values | [factors-and-dates.md](references/factors-and-dates.md) |
| `date + months(1)` on a month-end | Returns `NA`; use `%m+%` | [factors-and-dates.md](references/factors-and-dates.md) |
| Filtering does not drop factor levels | Phantom empty categories in plots/counts | [factors-and-dates.md](references/factors-and-dates.md) |
| A column shadows a same-named variable | Condition compares against the wrong thing | [tidy-eval.md](references/tidy-eval.md) |
| `geom_bar()` counts rows; `geom_col()` plots values | Valid-looking chart, wrong heights | [visualization.md](references/visualization.md) |
| `dt2 <- dt` aliases; `:=` then edits both | Original table mutated unexpectedly | [data-table.md](references/data-table.md) |
| `setkey()` reorders rows in place | Original row order lost; `dt[1]` changes meaning | [data-table.md](references/data-table.md) |

**Verify row counts after every join and filter.** An unexpected change in `nrow()` is
the earliest and cheapest signal that one of these has fired.

## Worked example

```r
library(tidyverse)

size_labels <- tibble(
  species = c("human", "wookiee", "gungan"),
  label   = c("common", "tall", "tall")
)

result <- starwars %>%
  # parse and clean
  mutate(
    species = str_to_lower(str_trim(species)),
    bmi     = mass / (height / 100)^2
  ) %>%
  # drop droids — filter_out keeps the 4 rows whose species is NA,
  # where filter(species != "droid") would silently discard them
  filter_out(species == "droid") %>%
  filter(!is.na(bmi)) %>%
  # enrich
  left_join(size_labels, by = join_by(species)) %>%
  # aggregate
  summarise(
    avg_bmi = mean(bmi),
    n       = n(),
    .by     = c(species, label)
  ) %>%
  arrange(desc(n))
```

The `filter_out()` step is the one worth pausing on: it keeps 81 rows where
`filter(species != "droid")` keeps 77, and the four-row gap is entirely characters
whose species is unrecorded. Dropping them is a decision, not a default.

## Style

Follow the [tidyverse style guide](https://style.tidyverse.org/) — it is the authority
wherever this skill is silent. In brief: snake_case throughout, descriptive
intermediate names (`sales_by_region`, not `df2`), one verb per pipe step when the
step is complex, and `TRUE`/`FALSE` never `T`/`F`.
