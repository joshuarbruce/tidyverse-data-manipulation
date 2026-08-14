# Strings with stringr

All functions share the `str_` prefix and a consistent argument order
`(string, pattern)`, which makes them pipe-friendly in a way base R's
`gsub(pattern, replacement, x)` is not. Prefer `str_*` over `gsub()` / `grep()` /
`paste()` throughout.

## Core set

```r
str_detect(x, "pattern")          # logical vector
str_subset(x, "pattern")          # keep matching elements
str_extract(x, "[0-9]+")          # first match
str_extract_all(x, "[0-9]+")      # all matches (list)
str_replace(x, "old", "new")      # first replacement
str_replace_all(x, "old", "new")  # all replacements
str_remove_all(x, "\\s")          # replace with ""
str_c("Hello", name, sep = " ")   # concatenate
str_glue("Hello {name}!")         # interpolation
str_trim(x)                       # strip leading/trailing whitespace
str_squish(x)                     # also collapse internal runs of whitespace
str_pad(x, width = 5, pad = "0")  # pad to width
str_length(x)                     # character count
str_split_i(x, ",", 1)            # nth piece, as a vector
```

`str_flatten()` collapses a vector to a single string;
`str_flatten_comma(x, last = " and ")` handles the Oxford comma.

## Case conversion

stringr 1.6 added converters for naming conventions:

```r
str_to_snake("Total Revenue ($USD)")  # "total_revenue_usd"
str_to_camel("Total Revenue")         # "totalRevenue"
str_to_kebab("Total Revenue")         # "total-revenue"

str_to_lower(x); str_to_upper(x); str_to_title(x)
```

`str_to_snake()` is the right tool for normalizing imported column names, and it
matches this skill's snake_case convention:

```r
df %>% rename_with(str_to_snake)
```

## Matching

```r
str_ilike(x, "app%")     # SQL ILIKE — case-insensitive wildcard
str_like(x, "app%")      # case-sensitive as of stringr 1.6
str_equal(x, y)          # unicode-aware comparison
```

`str_like(ignore_case = )` was deprecated in 1.6 — use `str_ilike()` instead.

## Pattern modifiers

By default patterns are regular expressions. Wrap them to change that:

```r
str_detect(x, fixed("a.b"))       # literal, no regex — also much faster
str_detect(x, coll("a", locale = "tr"))  # locale-aware collation
str_detect(x, regex("^ab", ignore_case = TRUE, multiline = TRUE))
```

`str_escape()` escapes regex metacharacters when you must build a pattern from
user-supplied text.

## Regex reference

| Pattern | Matches |
|---|---|
| `\\d` / `\\w` / `\\s` | digit / word character / whitespace |
| `[[:alpha:]]` | letters (locale-aware) |
| `^` / `$` | start / end of string |
| `\\b` | word boundary |
| `+` / `*` / `?` | one-or-more / zero-or-more / optional |
| `{2,4}` | between 2 and 4 times |
| `(...)` | capture group, referenced as `\\1` in replacements |
| `(?:...)` | non-capturing group |

Remember that R string literals need the backslash doubled: `"\\d"` is the regex `\d`.
Use `str_view(x, pattern)` to see interactively what a pattern matches — it highlights
matches in the console.

## Splitting a column

For splitting one column into several inside a data frame, use tidyr's
`separate_wider_delim()` / `separate_wider_regex()` rather than `str_split()` — they
handle naming and ragged input. See
[reshaping.md](reshaping.md).
