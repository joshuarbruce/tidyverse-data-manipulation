# data.table Reference

`data.table` is a high-performance extension of base R's `data.frame`. It is among the
fastest in-memory data manipulation tools in R, modifies columns by reference (no
copying), and has **zero dependencies** beyond base R — its own documentation cites
aggregation of 100GB in RAM. The trade-off is a terser, less immediately readable
syntax than the tidyverse.

Read this file when a task involves large data, explicit performance needs, an
existing data.table codebase, or the user names data.table directly. For everyday
work the tidyverse remains the default (see the decision guide in `SKILL.md`).

## The core idea: `DT[i, j, by]`

Most data.table work happens inside one bracket, read as SQL (the `set*()` functions,
`melt()`/`dcast()` and `fread()`/`fwrite()` below are the exceptions):

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

# examples use mtcars, with the row names kept as a column
dt <- as.data.table(mtcars, keep.rownames = "car")
```

## Filtering rows (i)

```r
dt[cyl == 6 & hp > 110]      # no dt$ prefix needed
dt[order(-mpg)]              # sort (radix sort, very fast)
```

## Selecting / computing columns (j)

`.()` is data.table's alias for `list()`:

```r
dt[, .(mpg, hp)]                # select columns
dt[, .(avg = mean(mpg))]        # compute a summary
dt[cyl == 6, mean(mpg)]         # filter + compute together
```

## Adding / modifying columns by reference: `:=`

This is data.table's signature feature — it updates the table **in place**, with no
copy. Fast and memory-light, but it mutates the original object, so use
deliberately:

```r
dt[, power_ratio := hp / wt]                  # add one column
dt[, c("hp2", "wt2") := .(hp * 2, wt * 2)]    # add several
dt[cyl == 6, six_cyl := TRUE]                 # conditional update on a subset
dt[, hp2 := NULL]                             # delete a column
dt[, grp_mean := mean(mpg), by = cyl]         # grouped update — writes the
                                              # group mean back to every row
```

The grouped form (`:= ... by =`) is a defining data.table idiom: it computes per group
but assigns to the original rows in place. The dplyr equivalent is
`mutate(z = mean(x), .by = g)` — and note that dplyr returns a new data frame where
data.table modifies the existing one.

To avoid mutating the original, copy first: `dt2 <- copy(dt)`. For assigning inside
a loop over many columns, `set(dt, i, j, value)` is the lowest-overhead option
(it skips the `[ ]` overhead entirely).

## Grouping (by)

```r
dt[, .N, by = cyl]                       # .N = row count per group
dt[, .(avg = mean(mpg)), by = cyl]       # grouped summary
dt[, .(avg = mean(mpg)), keyby = cyl]    # same, but sorted by group
dt[, .(avg = mean(mpg)), by = .(cyl, gear)]   # multiple groups
```

### Operating on many columns: `.SD` and `.SDcols`

`.SD` ("Subset of Data") is the group's data; combine with `lapply()` to apply a
function across columns:

```r
dt[, lapply(.SD, mean), by = cyl, .SDcols = c("mpg", "hp")]
dt[, head(.SD, 2), by = cyl]              # first 2 rows of each group
dt[, .SD[which.max(mpg)], by = cyl]       # the best-mpg row per group
```

`.SD` **excludes the `by` columns**, so `lapply(.SD, mean)` will not try to average
the grouping variable. The grouping columns still appear in the result — data.table
prepends them — which makes it look as though they were included. Naming `.SDcols`
explicitly is still worth doing: it documents intent and avoids silently picking up
new columns if the table gains any later.

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
dt[hp > 100, .(avg = mean(mpg)), by = cyl][order(-avg)]
```

## Keys and fast subsetting

Setting a **key** physically reorders the table by one or more columns (by
reference, in place). Subsetting on a keyed column then uses **binary search**
(O(log n)) instead of a full vector scan. The
[keys vignette](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-keys-fast-subset.html)
measures a **~215× speed-up** for a two-column subset of a 20-million-row, ~380 MiB
table (0.215s scanning versus effectively 0s keyed). Keys are the main reason
data.table is fast for repeated lookups.

```r
setkey(dt, cyl)                    # key by one column (unquoted)
setkeyv(dt, c("cyl", "gear"))      # key by a character vector (for programming)
key(dt)                            # inspect the current key

dt[.(6)]                           # keyed subset — binary search
dt[.(6, 4)]                        # matches cyl then gear
dt[.(6), max(mpg)]                 # combine with j
```

A table has at most one key (it can only be sorted one way). `keyby = ` in a grouped
query both groups and leaves the result keyed/sorted. `setorder(dt, -mpg)`
sorts in place without setting a key.

## Secondary indices and `on=`

A **secondary index** gives fast lookups on a column *without* physically reordering
the table. The `on=` argument creates one on the fly (auto-indexing) — which is why
the joins further down use `on=` rather than requiring a key:

```r
setindex(dt, gear)                 # build a reusable secondary index
dt[gear == 4]                      # auto-indexed after first use
dt[.(4), on = "gear"]              # explicit index subset, no key needed
```

Use `on=` for ad-hoc fast subsets/joins; promote to a `setkey()` when you query the
same column repeatedly and don't mind the table being reordered.

## Reshaping: melt / dcast

```r
# wide -> long  (tidyr::pivot_longer equivalent)
long <- melt(dt, id.vars = "car", measure.vars = c("mpg", "hp"),
             variable.name = "metric", value.name = "value")

# long -> wide  (tidyr::pivot_wider equivalent)
dcast(long, car ~ metric, value.var = "value")
```

## Joins

data.table joins via the bracket (`X[Y]`) or `merge()`. `merge()` reads more like
dplyr and is usually clearer:

```r
makers <- data.table(cyl = c(4, 6, 8), label = c("small", "mid", "big"))
merge(dt, makers, by = "cyl", all.x = TRUE)   # left join
```

The bracket form also supports the advanced joins dplyr gained in 1.1 — non-equi and
rolling:

```r
dt[makers, on = "cyl"]              # join by column

# non-equi: match each car to the mpg band containing it
bands <- data.table(lo = c(0, 20), hi = c(20, 40), band = c("thirsty", "efficient"))
dt[bands, on = .(mpg >= lo, mpg < hi)]

# rolling: match to the nearest preceding value
recs  <- data.table(t = c(1, 5, 9), reading = c(10, 20, 30))
probe <- data.table(t = c(4, 8))
recs[probe, on = "t", roll = TRUE]
```

`roll = TRUE` carries the last observation forward; `roll = "nearest"` matches whichever
side is closer. The dplyr counterpart is `join_by(closest(...))` — see
[joins.md](joins.md).

## Fast file I/O

These are worth using even in an otherwise-tidyverse workflow — they are markedly
faster than `read_csv()`/`write_csv()` on large files:

```r
path <- tempfile(fileext = ".csv")
fwrite(dt, path)
back <- fread(path)             # auto-detects types, very fast
```

## Fast vectorized helpers

data.table ships drop-in, faster (and more type-stable) replacements for common
operations. Prefer these inside data.table code — they avoid copies and handle types
predictably:

```r
x <- c(-1, 0, 5)

fifelse(x > 0, "pos", "neg")              # vectorized if_else (cf. dplyr::if_else)
fcase(                                    # vectorized case_when
  x < 0,  "neg",
  x == 0, "zero",
  x > 0,  "pos"
)
shift(x, n = 1, type = "lag")             # lag/lead (type = "lead" for forward)
uniqueN(x)                                # count distinct (cf. n_distinct)
nafill(c(1, NA, 3), type = "locf")        # fill NAs: "locf", "nocb", or fill = 0
rowid(c("a", "a", "b"))                   # within-group row counter
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
| Grouped mutate | `mutate(z = mean(x), .by = g)` | `dt[, z := mean(x), by = g]` |
| Arrange | `arrange(desc(x))` | `dt[order(-x)]` |
| Count per group | `count(g)` | `dt[, .N, by = g]` |
| Distinct rows | `distinct()` | `unique(dt)` |
| Distinct count | `n_distinct(x)` | `uniqueN(x)` |
| Rename | `rename(new = old)` | `setnames(dt, "old", "new")` |
| Conditional value | `if_else(c, a, b)` | `fifelse(c, a, b)` |
| Multi-condition | `case_when(...)` | `fcase(...)` |
| Lag / lead | `lag(x)` / `lead(x)` | `shift(x)` / `shift(x, type = "lead")` |
| Left join | `left_join(y, by = join_by(id))` | `merge(dt, y, by = "id", all.x = TRUE)` |
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

result <- lazy_dt(mtcars) %>%           # wrap in a lazy data.table
  filter(mpg > 20) %>%
  summarise(mean_hp = mean(hp), .by = cyl) %>%
  collect()                              # materialise back to a tibble
```

For most "I need it faster but I think in dplyr" situations, reach for dtplyr before
hand-writing data.table. Drop to raw data.table when you need `:=` in-place updates,
the absolute lowest overhead, or you're maintaining an existing data.table codebase.

## Gotchas

data.table's reference semantics are what make it fast, and they are also the source
of every serious surprise in this section. Coming from the tidyverse — where every
verb returns a new data frame — the mental model has to change: **`:=` and every
`set*()` function modify an existing object rather than producing a new one.**

### Assignment does not copy

This is the trap that costs the most debugging time. `dt2 <- dt` creates a second
name for the *same* table, not a copy:

```r
d1 <- data.table(x = 1:3)
d2 <- d1
d2[, y := 99]

names(d1)     # "x" "y"  <- the ORIGINAL changed too
```

`copy()` is the only way to get an independent table:

```r
d3 <- copy(d1)
d3[, z := 1]   # d1 is untouched
```

Nothing warns, and the code reads exactly like the tidyverse equivalent that would be
safe. Assume any `<-` between data.tables is an alias until you have written `copy()`.

### Functions modify their caller's table

The same semantics reach across function boundaries. A function that takes a
data.table and uses `:=` changes the object the caller passed in, even if it returns
nothing:

```r
add_flag <- function(d) {
  d[, flag := TRUE]
  invisible(NULL)
}

d4 <- data.table(x = 1:2)
add_flag(d4)
names(d4)   # "x" "flag"  <- modified from inside the function
```

That is often the *point* — it avoids copying a large table — but it must be
deliberate. If a function should not mutate its input, `copy()` on entry. Document
which of your functions mutate; a reader cannot tell from the call site.

**`setDT()` converts in place; `as.data.table()` copies.** `setDT(df)` turns the
caller's `data.frame` into a data.table without copying, which is efficient but means
`df` is no longer a `data.frame` afterwards.

### `setkey()` reorders rows in place

Setting a key physically sorts the table — the original row order is gone, silently:

```r
d5 <- data.table(id = c(3L, 1L, 2L))
setkey(d5, id)
d5$id   # 1 2 3
```

If anything downstream depends on the incoming order — a row number recorded earlier,
a positional join, `dt[1]` — it now means something different. `dt[1]` is always a
*row* index, never a key lookup, so it silently returns a different row after keying.
Use `setorder()` when you want sorting without a key, and capture the original order in
a column first if you need to restore it.

### `melt()` coerces mixed types where tidyr errors

Melting columns of different types does not fail — data.table warns and coerces
everything to the highest type in its hierarchy, usually `character`:

```r
mixed <- data.table(id = 1, num = 1.5, chr = "a")
melt(mixed, id.vars = "id", measure.vars = c("num", "chr"))
# warning: not all of the same type ... will be of type 'character'
```

tidyr's `pivot_longer()` errors on the same input. If you are translating a pipeline
between the two, this is a place where the data.table version keeps going and produces
a silently stringified column.

### Loading data.table masks lubridate's date accessors

This one bites precisely because mixing is recommended — `fread()` is worth using
inside an otherwise-tidyverse script, and the `library(data.table)` that enables it
also masks **twelve** lubridate functions:

```
hour  isoweek  isoyear  mday  minute  month
quarter  second  wday  week  yday  year
```

data.table's versions are plain numeric extractors with no `label`, `abbr`, or
`week_start` arguments, so lubridate idioms break as soon as data.table is attached
second:

```r
d <- as.Date("2024-03-15")

library(lubridate)
month(d, label = TRUE)              # "Mar"

library(data.table)
month(d, label = TRUE)              # Error: unused argument (label = TRUE)
lubridate::month(d, label = TRUE)   # "Mar" — qualify to be safe
```

It fails loudly, which is the good case — but the failure appears far from the
`library()` call that caused it. Qualify explicitly (`lubridate::month(d, label = TRUE)`)
in any script that loads both, or load data.table *before* the tidyverse so the
tidyverse wins.

data.table also masks three dplyr functions: **`between()`, `first()`, `last()`**.
These behave the same for ordinary positional use, so nothing breaks silently — but the
argument names differ (`lower`/`upper` versus `left`/`right`), so a named call like
`between(x, left = 1, right = 10)` fails once data.table is attached.

### Smaller traps

- **`.()` is `list()`** — forgetting it in `j` changes the return type. `dt[, x]`
  returns a vector; `dt[, .(x)]` returns a data.table.
- **Auto-printing is suppressed after `:=`.** At the console, `dt[, y := 2]` prints
  nothing at all. Append `[]` — `dt[, y := 2][]` — to force it. This also affects
  tables built or modified inside functions.
- **Type coercion in `:=` differs by scope.** Replacing a whole column promotes its
  type silently (`dt[, i := i + 0.5]` turns an integer column into a double). Assigning
  into a *subset* cannot change the column type, so it truncates and warns:
  `dt[1, i := 1.5]` warns "truncated (precision lost)" and stores `1`. Assigning a
  character into a numeric column also warns and coerces. Treat any coercion warning
  from `:=` as a real bug, not noise.
- **`:=` does not recycle silently.** Supplying 2 values for 3 rows errors, and so does
  2 for 4 — data.table requires `rep()` to make recycling explicit. This one is a
  safety feature, not a trap.

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
(also recorded in [sources.md](sources.md)). The dtplyr bridge is documented at
<https://dtplyr.tidyverse.org/>.
