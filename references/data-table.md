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
```

To avoid mutating the original, copy first: `dt2 <- copy(dt)`.

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
```

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

## tidyverse ↔ data.table translation table

| Task | dplyr / tidyr | data.table |
|---|---|---|
| Filter rows | `filter(x > 0)` | `dt[x > 0]` |
| Select columns | `select(a, b)` | `dt[, .(a, b)]` |
| Add/modify column | `mutate(z = x + y)` | `dt[, z := x + y]` |
| Summarise | `summarise(m = mean(x))` | `dt[, .(m = mean(x))]` |
| Group + summarise | `summarise(m = mean(x), .by = g)` | `dt[, .(m = mean(x)), by = g]` |
| Arrange | `arrange(desc(x))` | `dt[order(-x)]` |
| Count per group | `count(g)` | `dt[, .N, by = g]` |
| Distinct rows | `distinct()` | `unique(dt)` |
| Rename | `rename(new = old)` | `setnames(dt, "old", "new")` |
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
