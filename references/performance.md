# Performance Reference

## Profile before optimizing

Never guess at bottlenecks. Measure first:

```r
library(profvis)
profvis({ your_code_here })  # line-by-line flame graph in RStudio viewer

library(bench)
bench::mark(
  dplyr  = df %>% filter(x > 0),
  base   = df[df$x > 0, ],
  check  = FALSE  # skip output equality check
)
```

## Fast CSV reading: vroom / readr

`readr::read_csv()` is fast enough for most files. For multi-GB CSVs, `vroom` is
faster via lazy loading:

```r
library(vroom)
df <- vroom("large.csv", col_types = cols(id = col_character()))
```

`vroom` also reads multiple files efficiently:
```r
df <- vroom(list.files("data/", "\\.csv$", full.names = TRUE))
```

## dtplyr — dplyr syntax over data.table

For datasets in the 1–10 GB range, `dtplyr` lets you write standard dplyr code
that gets translated to `data.table` operations automatically:

```r
library(dtplyr)
library(data.table)

dt <- lazy_dt(df)              # wrap in a lazy data.table
result <- dt %>%
  filter(year >= 2020) %>%
  summarise(total = sum(sales), .by = region) %>%
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
results <- map(file_list, in_parallel(\(f) process(f), process = process))
```

**furrr** mirrors purrr's API with a `future_` prefix and is the mature option:

```r
library(furrr)
plan(multisession, workers = 4)  # use 4 cores

results <- future_map(file_list, read_and_process)
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
