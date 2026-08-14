# Strings with stringr

All functions share the `str_` prefix and a consistent argument order
`(string, pattern)`, which makes them pipe-friendly in a way base R's
`gsub(pattern, replacement, x)` is not. Prefer `str_*` over `gsub()` / `grep()` /
`paste()` throughout.

## Core set

```r
library(stringr)
x <- fruit[1:5]                     # "apple" "apricot" "avocado" "banana" "bell pepper"

str_detect(x, "an")                 # logical vector
str_subset(x, "an")                 # keep matching elements
str_extract(x, "[aeiou]+")          # first match
str_extract_all(x, "[aeiou]+")      # all matches (list)
str_replace(x, "a", "A")            # first replacement
str_replace_all(x, "a", "A")        # all replacements
str_remove_all(x, "\\s")            # replace with ""
str_c("the", x, sep = " ")          # concatenate
str_glue("a {x}!")                  # interpolation
str_trim("  padded  ")              # strip leading/trailing whitespace
str_squish("  a   b  ")             # also collapse internal whitespace
str_pad(x, width = 12, pad = ".")   # pad to width
str_length(x)                       # character count
str_split_i("a,b,c", ",", 1)        # nth piece, as a vector
```

`str_flatten()` collapses a vector to a single string;
`str_flatten_comma(x, last = " and ")` handles the Oxford comma.

## Case conversion

stringr 1.6 added converters for naming conventions:

```r
str_to_snake("Total Revenue ($USD)")  # "total_revenue_usd"
str_to_camel("Total Revenue")         # "totalRevenue"
str_to_kebab("Total Revenue")         # "total-revenue"

str_to_lower(fruit[1]); str_to_upper(fruit[1]); str_to_title("bell pepper")
```

`str_to_snake()` is the right tool for normalizing imported column names, and it
matches this skill's snake_case convention:

```r
tibble(`Total Revenue` = 1, `Unit Count` = 2) %>% rename_with(str_to_snake)
```

## `NA` propagates through `str_*`

Every `str_*` function returns `NA` for `NA` input — including the predicates:

```r
str_detect(c("apple", NA, "banana"), "an")   # TRUE, NA, TRUE
```

Because `filter()` treats `NA` as `FALSE`, a `str_detect()` condition silently drops
missing rows. Use `filter_out()` for the negative case, or handle `NA` explicitly with
`replace_na()` or `coalesce()` before matching. See
[filtering-and-recoding.md](filtering-and-recoding.md).

## Matching

```r
str_ilike("Apple", "app%")   # TRUE  — case-insensitive wildcard
str_like("Apple", "app%")    # FALSE — case-sensitive as of stringr 1.6
str_equal("a", "a")          # unicode-aware comparison
```

`str_like(ignore_case = )` was deprecated in 1.6 — use `str_ilike()` instead.

## Pattern modifiers

By default patterns are regular expressions. Wrap them to change that:

```r
str_detect("a.b", fixed("a.b"))                  # literal, no regex — also faster
str_detect("a", coll("a", locale = "tr"))        # locale-aware collation
str_detect("AB", regex("^ab", ignore_case = TRUE))
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
