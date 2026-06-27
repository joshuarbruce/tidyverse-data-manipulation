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

## Parallel processing with purrr + furrr

For CPU-bound map operations:

```r
library(furrr)
plan(multisession, workers = 4)  # use 4 cores

results <- future_map(file_list, read_and_process)
```

`furrr` mirrors `purrr`'s API with a `future_` prefix. Reserve it for operations
where the per-item work is substantial (>100ms each) — the overhead isn't worth it
for fast operations.

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
