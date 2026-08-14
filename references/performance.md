# Performance Reference

## Profile before optimizing

Never guess at bottlenecks. Measure first:

```r
library(profvis)
profvis({ mtcars[rep(1:32, 1000), ] |> subset(mpg > 20) })  # flame graph

library(bench)
bench::mark(
  dplyr = dplyr::filter(mtcars, mpg > 20),
  base  = mtcars[mtcars$mpg > 20, ],
  check = FALSE  # skip output equality check
)
```

## Fast CSV reading: vroom / readr

`readr::read_csv()` is fast enough for most files. For multi-GB CSVs, `vroom` is
faster via lazy loading:

```r
library(vroom)
chickens <- vroom(readr::readr_example("chickens.csv"), show_col_types = FALSE)
```

`vroom` also reads multiple files efficiently:
```r
vroom(c(readr::readr_example("mini-gapminder-africa.csv"),
        readr::readr_example("mini-gapminder-americas.csv")), show_col_types = FALSE)
```

## dtplyr — dplyr syntax over data.table

For datasets in the 1–10 GB range, `dtplyr` lets you write standard dplyr code
that gets translated to `data.table` operations automatically:

```r
library(dtplyr)
library(dplyr)

lazy_dt(mtcars) %>%            # wrap in a lazy data.table
  filter(mpg > 20) %>%
  summarise(mean_hp = mean(hp), .by = cyl) %>%
  collect()                    # materialise the result
```

The `collect()` call triggers evaluation. Without it you get a lazy query object.

## duckdb — SQL-powered analytics in-process

For very large files or complex aggregations, DuckDB is often the fastest option
and integrates with dplyr via `duckplyr` or direct SQL:

```r
library(duckdb)
library(dplyr)

con <- dbConnect(duckdb())
duckdb_read_csv(con, "sales", "sales.csv")

tbl(con, "sales") %>%
  filter(year >= 2020) %>%
  summarise(total = sum(amount), .by = region) %>%
  collect()

dbDisconnect(con)
```

DuckDB can query Parquet, CSV, and JSON files directly without loading into memory.

## Parallel processing

As of purrr 1.2 there are two reasonable options. Reserve either for operations where
the per-item work is substantial (>100ms each) — below that, the serialization
overhead costs more than it saves.

**purrr's own `in_parallel()`** (experimental, mirai-backed) keeps you in the ordinary
`map()` call and forces dependencies to be explicit:

```r
mirai::daemons(4)
results <- purrr::map(1:4, purrr::in_parallel(\(i) i^2))
```

**furrr** mirrors purrr's API with a `future_` prefix and is the mature option:

```r
library(furrr)
plan(multisession, workers = 4)  # use 4 cores

results <- future_map(split(mtcars, mtcars$cyl), nrow)
```

Prefer `in_parallel()` for new code in a purrr-centric pipeline; prefer furrr when you
already use the futureverse elsewhere or need its established plan backends.

## dplyr 1.2 recoding performance

`if_else()`, `case_when()`, and `coalesce()` were rewritten in C via vctrs in dplyr
1.2.0 — substantially faster and much lighter on memory. If you previously dropped to
`data.table::fifelse()` / `fcase()` purely for speed on recoding-heavy pipelines,
re-benchmark before assuming that's still necessary.

## General tips

- **Vectorize** — replace row-by-row `rowwise()` loops with vectorized operations
  or `across()` whenever possible.
- **Filter early** — reduce the data size before expensive joins or mutations.
- **Avoid repeated `rbind()` in a loop** — collect results in a list and call
  `list_rbind()` once at the end.
- **Use column types** — storing integers as doubles or IDs as factors wastes
  memory and slows comparisons. Set `col_types` in `read_csv()`.
- **`collect()` late** — if using dtplyr or dbplyr, don't materialise until you
  need the final result.
