# data.table Reference

`data.table` is a high-performance extension of base R's `data.frame`. It is the
fastest in-memory data manipulation tool in R, handles datasets up to ~100GB,
modifies columns by reference (no copying), and has **zero dependencies** beyond
base R. The trade-off is a terser, less immediately readable syntax than the
tidyverse.

Read this file when a task involves large data, explicit performance needs, an
existing data.table codebase, or the user names data.table directly. For everyday
work the tidyverse remains the default (see the decision guide in `SKILL.md`).

## The core idea: `DT[i, j, by]`

Every data.table operation fits one bracket, read as SQL:

```
DT[ i ,        j ,        by ]
   WHERE     SELECT/    GROUP BY
   /ORDER BY UPDATE
```

- **i** — which rows (filter / sort)
- **j** — what to do with columns (select / compute / update)
- **by** — grouping

```r
library(data.table)
dt <- as.data.table(df)        # or fread() to read straight into a data.table
```

## Filtering rows (i)

```r
dt[origin == "JFK" & month == 6L]      # no dt$ prefix needed
dt[order(-arr_delay)]                  # sort (radix sort, very fast)
```

## Selecting / computing columns (j)

`.()` is data.table's alias for `list()`:

```r
dt[, .(arr_delay, dep_delay)]                       # select columns
dt[, .(avg = mean(arr_delay))]                       # compute a summary
dt[origin == "JFK", mean(arr_delay)]                 # filter + compute together
```

## Adding / modifying columns by reference: `:=`

This is data.table's signature feature — it updates the table **in place**, with no
copy. Fast and memory-light, but it mutates the original object, so use
deliberately:

```r
dt[, speed := distance / air_time]            # add one column
dt[, c("a", "b") := .(x * 2, y * 3)]          # add several
dt[origin == "JFK", flag := TRUE]             # conditional update on a subset
dt[, col := NULL]                              # delete a column
dt[, grp_mean := mean(arr_delay), by = origin] # grouped update — writes the
                                               # group mean back to every row
```

The grouped form (`:= ... by =`) is a defining data.table idiom: it computes per
group but assigns to the original rows in place, which in dplyr would take a
`group_by() %>% mutate() %>% ungroup()`.

To avoid mutating the original, copy first: `dt2 <- copy(dt)`. For assigning inside
a loop over many columns, `set(dt, i, j, value)` is the lowest-overhead option
(it skips the `[ ]` overhead entirely).

## Grouping (by)

```r
dt[, .N, by = origin]                          # .N = row count per group
dt[, .(avg = mean(arr_delay)), by = origin]    # grouped summary
dt[, .(avg = mean(arr_delay)), keyby = origin] # same, but sorted by group
dt[, .(avg = mean(arr_delay)), by = .(origin, month)]   # multiple groups
```

### Operating on many columns: `.SD` and `.SDcols`

`.SD` ("Subset of Data") is the group's data; combine with `lapply()` to apply a
function across columns:

```r
dt[, lapply(.SD, mean), by = origin, .SDcols = c("arr_delay", "dep_delay")]
dt[, head(.SD, 2), by = origin]                 # first 2 rows of each group
dt[, .SD[which.max(arr_delay)], by = origin]    # the worst-delay row per group
```

### Special symbols

Inside `j` and `by`, data.table provides shorthand symbols. Knowing these unlocks
most idioms:

| Symbol | Meaning |
|---|---|
| `.N` | Number of rows (in the group, or whole table) |
| `.SD` | The subset of data for the current group |
| `.SDcols` | Which columns `.SD` should include |
| `.I` | Integer row positions (e.g. `dt[, .I[which.max(x)], by = g]`) |
| `.GRP` | Current group counter (1, 2, 3, …) |
| `.BY` | Named list of the current group's `by` values |

## Chaining

Append brackets to pipe one result into the next query — data.table's equivalent of
the `%>%` chain:

```r
dt[origin == "JFK", .(avg = mean(arr_delay)), by = month][order(-avg)]
```

## Keys and fast subsetting

Setting a **key** physically reorders the table by one or more columns (by
reference, in place). Subsetting on a keyed column then uses **binary search**
(O(log n)) instead of a full vector scan — the keys vignette shows a ~489× speedup
on 20M rows. Keys are the main reason data.table is fast for repeated lookups.

```r
setkey(dt, origin)                 # key by one column (unquoted)
setkeyv(dt, c("origin", "dest"))   # key by a character vector (for programming)
key(dt)                            # inspect the current key

dt[.("JFK")]                       # keyed subset — binary search
dt[.("JFK", "MIA")]                # matches origin then dest
dt[.("JFK"), max(arr_delay)]       # combine with j
```

A table has at most one key (it can only be sorted one way). `keyby = ` in a grouped
query both groups and leaves the result keyed/sorted. `setorder(dt, -arr_delay)`
sorts in place without setting a key.

## Secondary indices and `on=`

A **secondary index** gives fast lookups on a column *without* physically reordering
the table. The `on=` argument creates one on the fly (auto-indexing) — this is why
the joins above use `on=`:

```r
setindex(dt, origin)               # build a reusable secondary index
dt[origin == "JFK"]                # auto-indexed after first use
dt["JFK", on = "origin"]           # explicit index subset, no key needed
```

Use `on=` for ad-hoc fast subsets/joins; promote to a `setkey()` when you query the
same column repeatedly and don't mind the table being reordered.

## Reshaping: melt / dcast

```r
# wide -> long  (tidyr::pivot_longer equivalent)
melt(dt, id.vars = "id", measure.vars = c("jan", "feb"),
     variable.name = "month", value.name = "sales")

# long -> wide  (tidyr::pivot_wider equivalent)
dcast(dt, id ~ month, value.var = "sales")
```

## Joins

data.table joins via the bracket (`X[Y]`) or `merge()`. `merge()` reads more like
dplyr and is usually clearer:

```r
merge(orders, customers, by = "customer_id", all.x = TRUE)   # left join
```

It also supports advanced joins the tidyverse does too — non-equi, rolling:

```r
setkey(dt, id)                      # set a key for fast repeated joins
dt[customers, on = "customer_id"]   # join by key
dt[windows, on = .(time >= start, time < end)]   # non-equi join
```

## Fast file I/O

These are worth using even in an otherwise-tidyverse workflow — they are markedly
faster than `read_csv()`/`write_csv()` on large files:

```r
dt <- fread("big.csv")          # auto-detects types, very fast
fwrite(dt, "out.csv")
```

## Fast vectorized helpers

data.table ships drop-in, faster (and more type-stable) replacements for common
operations. Prefer these inside data.table code — they avoid copies and handle types
predictably:

```r
fifelse(x > 0, "pos", "neg")              # vectorized if_else (cf. dplyr::if_else)
fcase(                                    # vectorized case_when
  x < 0, "neg",
  x == 0, "zero",
  x > 0, "pos"
)
shift(x, n = 1, type = "lag")             # lag/lead (type = "lead" for forward)
uniqueN(x)                                # count distinct (cf. n_distinct)
nafill(x, type = "locf")                  # fill NAs: "locf", "nocb", or fill = 0
rowid(group)                              # within-group row counter
frank(x, ties.method = "min")             # fast rank
```

## tidyverse ↔ data.table translation table

| Task | dplyr / tidyr | data.table |
|---|---|---|
| Filter rows | `filter(x > 0)` | `dt[x > 0]` |
| Select columns | `select(a, b)` | `dt[, .(a, b)]` |
| Add/modify column | `mutate(z = x + y)` | `dt[, z := x + y]` |
| Summarise | `summarise(m = mean(x))` | `dt[, .(m = mean(x))]` |
| Group + summarise | `summarise(m = mean(x), .by = g)` | `dt[, .(m = mean(x)), by = g]` |
| Grouped mutate | `group_by(g) %>% mutate(z = mean(x))` | `dt[, z := mean(x), by = g]` |
| Arrange | `arrange(desc(x))` | `dt[order(-x)]` |
| Count per group | `count(g)` | `dt[, .N, by = g]` |
| Distinct rows | `distinct()` | `unique(dt)` |
| Distinct count | `n_distinct(x)` | `uniqueN(x)` |
| Rename | `rename(new = old)` | `setnames(dt, "old", "new")` |
| Conditional value | `if_else(c, a, b)` | `fifelse(c, a, b)` |
| Multi-condition | `case_when(...)` | `fcase(...)` |
| Lag / lead | `lag(x)` / `lead(x)` | `shift(x)` / `shift(x, type = "lead")` |
| Left join | `left_join(y, by = "id")` | `merge(dt, y, by = "id", all.x = TRUE)` |
| Wide → long | `pivot_longer(...)` | `melt(dt, ...)` |
| Long → wide | `pivot_wider(...)` | `dcast(dt, ...)` |
| Read CSV | `read_csv("f.csv")` | `fread("f.csv")` |

## The bridge: dtplyr (best of both)

If you want tidyverse readability **and** data.table speed, `dtplyr` translates
dplyr code to data.table operations automatically — you rarely have to write
bracket syntax at all:

```r
library(dtplyr)
library(dplyr)

result <- lazy_dt(df) %>%               # wrap in a lazy data.table
  filter(year >= 2020) %>%
  summarise(total = sum(sales), .by = region) %>%
  collect()                              # materialise back to a tibble
```

For most "I need it faster but I think in dplyr" situations, reach for dtplyr before
hand-writing data.table. Drop to raw data.table when you need `:=` in-place updates,
the absolute lowest overhead, or you're maintaining an existing data.table codebase.

## Gotchas

- **`:=` mutates in place.** The original object changes even without reassignment.
  `copy()` first if you need to preserve it.
- **`.()` is `list()`** — forgetting it in `j` changes the return type.
- **Auto-printing** — a data.table created or modified inside a function may not
  print; add a trailing `dt[]` to force it.
- **Type coercion in `:=`** — assigning a double into an integer column can warn or
  truncate; match types or create the column fresh.

## Sources & further reading

This reference distills data.table's official vignettes. For the full treatment of
any topic, read the corresponding vignette:

| Topic in this file | Official vignette |
|---|---|
| `DT[i, j, by]`, basics | [Introduction](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-intro.html) |
| `:=`, grouped update, `set()` | [Reference semantics](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-reference-semantics.html) |
| `setkey`, binary search | [Keys and fast subsetting](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-keys-fast-subset.html) |
| `on=`, `setindex`, auto-indexing | [Secondary indices and auto-indexing](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-secondary-indices-and-auto-indexing.html) |
| `.SD`, `.SDcols`, special symbols | [Using .SD for data analysis](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-sd-usage.html) |
| `melt` / `dcast` | [Reshaping data](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-reshape.html) |
| `merge`, non-equi, rolling joins | [Joins](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-joins.html) |
| `fread` / `fwrite` | [File reading and writing](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-fread-and-fwrite.html) |

Package home: <https://r-datatable.com/> · Source: <https://github.com/Rdatatable/data.table>
(also recorded in `sources.md`). The dtplyr bridge is documented at
<https://dtplyr.tidyverse.org/>.
