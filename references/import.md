# Importing Data and Building Tibbles

Getting data into a tibble, and constructing tibbles by hand. Read this for reading
files, controlling column parsing, or diagnosing import problems.

## readr — delimited files

```r
library(readr)

# readr ships example files, so this runs as written
chickens <- read_csv(readr_example("chickens.csv"), col_types = cols(
  chicken   = col_character(),
  eggs_laid = col_integer()
))
```

**Always specify `col_types` for columns that need non-default parsing.** The two that
bite most often are dates and identifiers that look numeric — zip codes, account
numbers, and anything with leading zeros will silently become integers and lose them.

Other functions in the family: `read_tsv()`, `read_delim()` (explicit `delim`),
`read_csv2()` (semicolon-separated, comma decimal), `read_lines()`, and `read_fwf()`
for fixed-width files.

Useful arguments:

- `n_max` — read a few rows first to check parsing before committing to the full file.
- `na` — the strings to treat as missing, e.g. `na = c("", "NA", "NULL", "-99")`.
- `skip` / `comment` — for files with preamble junk.
- `name_repair` — control how bad column names are fixed.
- `col_select` — read only the columns you need, which is much faster on wide files.

### Diagnosing parse failures

`problems()` reports where the file disagreed with the column types in force — which
in practice means the types **you** specified:

```r
path <- tempfile(fileext = ".csv")
writeLines(c("id,score", paste0(1:1200, ",", 1:1200), "1201,not_measured"), path)

# declare score numeric; the last row cannot be parsed as one
d <- read_csv(path, col_types = cols(score = col_double()))
problems(d)
#> row 1202, col 2, expected "a double", actual "not_measured"
```

`problems()` gives the row, column, what was expected, and what was actually found —
enough to go straight to the offending line.

**Do not rely on `guess_max` the way older material suggests.** readr 1.x guessed
column types from the first 1,000 rows, so a stray value further down produced a wrong
type, and the standard advice was to raise `guess_max`. readr 2.x is vroom-backed and
types the whole column: reading the file above with no `col_types` yields `character`
regardless of whether `guess_max` is 10, 1,000, or `Inf`. Tested across the classic
failure shapes — a late text value in a numeric column, a late decimal in an integer
column, a late non-logical in a `TRUE`/`FALSE` column, and a column that is empty or
`NA` until the final row — `guess_max` changed nothing on readr 2.2.0.

The practical consequence is a reversal of the old workflow. Guessing rarely misfires
now, so `problems()` is empty until you declare types — and declaring them is still
worth doing, because a silently correct guess today can become a wrong one when next
month's file arrives with different data. Specify `col_types` for anything load-bearing
and let `problems()` tell you when reality stops matching the spec.

## Other formats

| Format | Function |
|---|---|
| Excel | `readxl::read_excel(path, sheet = )` |
| SPSS / Stata / SAS | `haven::read_sav()` / `read_dta()` / `read_sas()` |
| Google Sheets | `googlesheets4::read_sheet()` |
| JSON | `jsonlite::fromJSON()` |
| Databases | `DBI::dbConnect()` + `dplyr::tbl()` — see [performance.md](performance.md) |

For very large delimited files, `data.table::fread()` is markedly faster than
`read_csv()` and worth using even inside an otherwise-tidyverse script. See
[data-table.md](data-table.md) and [performance.md](performance.md).

## Cleaning column names on import

Imported headers are rarely snake_case. Normalize them immediately, before writing any
downstream code against them:

```r
tibble::tibble(`Total Revenue` = 1) %>% dplyr::rename_with(stringr::str_to_snake)
```

See [strings.md](strings.md) for the rest of the
`str_to_*` family.

## tibble — constructing data frames

```r
library(tibble)

tibble(x = 1:3, y = x * 2)     # later columns can reference earlier ones
as_tibble(mtcars)              # convert a base data.frame
```

`tribble()` builds a tibble row by row, which is far more readable for small lookup
tables and test fixtures:

```r
tribble(
  ~island,     ~latitude,
  "Biscoe",    -65.5,
  "Dream",     -64.7,
  "Torgersen", -64.8
)
```

Tibbles never convert strings to factors, never mangle column names, never do partial
matching on `$`, and print compactly with column types shown.

Use `glimpse()` rather than `str()` for a quick overview — it prints one row per
column with its type and first few values, which is far easier to scan on wide data.
