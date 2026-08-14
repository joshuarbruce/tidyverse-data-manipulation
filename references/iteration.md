# Functional Iteration with purrr

Replacing `for` loops and the `*apply` family with `map_*()`. Read this for repeating
an operation over files, columns, list elements, or model specifications.

## The map family

`map()` always returns a list. The typed variants return an atomic vector and **error
if the result is the wrong type** — that type checking is the main reason to prefer
them over `sapply()`:

```r
results <- map(file_list, read_csv)                          # list
means   <- map_dbl(df_list, \(d) mean(d$value, na.rm = TRUE)) # double vector
names   <- map_chr(models, \(m) class(m)[1])                  # character vector
flags   <- map_lgl(x, is.numeric)                             # logical vector
```

Use `\(x)` for anonymous functions rather than `function(x)` or the older `~ .x`
formula syntax.

`map_vec()` returns a vector whose type follows the input, which is the right choice
for dates and other vctrs-backed types that `map_dbl()` would strip.

## Combining results

```r
combined <- map(file_list, read_csv) %>% list_rbind()   # stack data frames
wide     <- map(cols, summarize_col) %>% list_cbind()   # bind side by side
values   <- map(x, f) %>% list_c()                      # flatten to a vector
```

`list_rbind()` accepts `names_to`, which records which element each row came from —
useful when reading many files and needing to know the source:

```r
files <- list.files("data/", pattern = "\\.csv$", full.names = TRUE)

all_data <- files %>%
  set_names(basename) %>%
  map(read_csv, col_types = cols(.default = col_character())) %>%
  list_rbind(names_to = "source_file")
```

## Multiple inputs, and side effects

```r
map2(x, y, \(a, b) a + b)                 # two inputs, in parallel
pmap(list(x, y, z), \(a, b, c) a + b + c) # any number of inputs
imap(x, \(val, nm) paste(nm, val))        # value and its name/index

walk(paths, \(p) write_csv(df, p))        # side effects; returns input invisibly
```

`keep()` / `discard()` filter a list by a predicate; `reduce()` collapses it to a
single value; `accumulate()` keeps the intermediate results.

## Parallel maps

For CPU-bound work, wrap `.f` in `in_parallel()` (purrr ≥ 1.2, powered by mirai,
currently experimental) to run across worker processes without leaving purrr:

```r
mirai::daemons(4)   # without daemons, it silently falls back to sequential

results <- map(files, in_parallel(
  \(f) process(readr::read_csv(f)),
  process = process          # declare every dependency explicitly
))
```

The wrapped function is serialized and shipped to another process, so it must be
self-contained: call package functions with a `::` prefix, and pass any local function
or data it needs as a named argument to `in_parallel()`. That isolation is the point —
it prevents large objects from being captured by accident.

Reserve parallelism for work where each item takes substantial time (>100ms). Below
that, serialization overhead costs more than it saves. See
[performance.md](performance.md) for the furrr alternative
and how to choose.

## Superseded and changed behavior

- `map_dfr()` and `map_dfc()` are superseded — use `map() %>% list_rbind()` and
  `map() %>% list_cbind()` instead.
- `flatten()` is superseded by `list_flatten()` / `list_c()`; `transpose()` by
  `list_transpose()`.
- **purrr 1.2 tightened `map_chr()`**: it no longer silently coerces logical, integer,
  or double to string, and now errors instead. Wrap the value in `as.character()`, or
  use `map_vec()` when the output type should follow the input. This breaks code that
  previously worked, and the error appears at the map call rather than where the type
  was introduced.
- `map()` and friends require R ≥ 4.1 as of purrr 1.1.
