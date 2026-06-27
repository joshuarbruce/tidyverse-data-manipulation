# Tidy Evaluation Reference

Tidy eval is the mechanism that lets dplyr/tidyr refer to column names without
quoting them. Understanding the two paradigms — **data masking** and **tidy
selection** — tells you which tools to reach for when writing your own functions.

## Data masking vs tidy selection

**Data masking** — columns are evaluated as if they were variables in the
environment. Used by `filter()`, `mutate()`, `summarise()`, `arrange()`.

**Tidy selection** — a mini-language for selecting column positions/names. Used by
`select()`, `rename()`, `relocate()`, `across()`, `pivot_longer(cols = ...)`.

## Writing functions: the embrace operator `{{ }}`

Use `{{ }}` to forward a single column argument into a data-masking context:

```r
group_mean <- function(df, col, group) {
  df %>%
    summarise(mean = mean({{ col }}, na.rm = TRUE), .by = {{ group }})
}

group_mean(sales, revenue, region)
```

`{{ col }}` is shorthand for `!!enquo(col)` — you rarely need the long form.

## Passing character column names: `.data[[]]`

When the caller passes a string rather than a bare name, use `.data[[]]`:

```r
filter_above <- function(df, col_name, threshold) {
  df %>% filter(.data[[col_name]] > threshold)
}

filter_above(df, "price", 100)
```

## Tidy selection in functions: `all_of()` / `any_of()`

When forwarding a character vector of column names into a tidy-selection context:

```r
drop_cols <- function(df, cols) {
  df %>% select(-all_of(cols))
}

drop_cols(df, c("id", "temp"))
```

- `all_of()` — errors if any name is missing (safe default).
- `any_of()` — silently skips missing names (useful for "drop if present").

## Dynamic dots `...`

Pass multiple column arguments through with `...`:

```r
group_summary <- function(df, ...) {
  df %>% summarise(n = n(), .by = c(...))
}

group_summary(df, region, year)
```

## Injecting computed names with `:=`

When the output column name should be dynamic, use `:=` with `glue`-style
interpolation via `"{name}"`:

```r
add_zscore <- function(df, col) {
  col_name <- paste0(rlang::as_label(enquo(col)), "_z")
  df %>% mutate("{col_name}" := scale({{ col }})[,1])
}
```

## Common patterns cheat sheet

| Situation | Tool |
|---|---|
| Pass bare column name to dplyr verb | `{{ col }}` |
| Pass string column name to dplyr verb | `.data[[col]]` |
| Pass vector of strings to select/rename | `all_of(cols)` |
| Build column name dynamically | `"{name}" :=` |
| Accept any number of grouping columns | `...` |
