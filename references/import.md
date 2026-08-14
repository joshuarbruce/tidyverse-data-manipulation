# Importing Data and Building Tibbles

Getting data into a tibble, and constructing tibbles by hand. Read this for reading
files, controlling column parsing, or diagnosing import problems.

## readr — delimited files

```r
df <- read_csv("data.csv", col_types = cols(
  date = col_date(),
  id   = col_character()
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

```r
df <- read_csv("data.csv")
problems(df)     # rows/columns where parsing failed, and what was expected
```

`problems()` is the first thing to check when a column comes back as the wrong type or
full of `NA`. readr guesses types from the first 1,000 rows by default; a value that
breaks the guess appears far down the file. Raise `guess_max`, or set `col_types`
explicitly.

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
df %>% rename_with(str_to_snake)
```

See [strings.md](strings.md) for the rest of the
`str_to_*` family.

## tibble — constructing data frames

```r
tibble(x = 1:3, y = x * 2)     # later columns can reference earlier ones
as_tibble(df)                  # convert a base data.frame
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
