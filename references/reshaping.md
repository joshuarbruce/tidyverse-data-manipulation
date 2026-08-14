# Reshaping with tidyr

Pivoting, nesting, splitting columns, and handling explicit missing values. Read this
when data arrives in the wrong shape — one row per measurement instead of per
observation, values spread across columns, or several values packed into one column.

## Pivoting

`pivot_longer()` and `pivot_wider()` cover almost all reshaping needs:

```r
# wide → long
df %>% pivot_longer(
  cols      = starts_with("week_"),
  names_to  = "week",
  values_to = "sales"
)

# long → wide
df %>% pivot_wider(names_from = category, values_from = amount)
```

Useful arguments:

- `names_prefix` — strip a common prefix while pivoting longer.
- `names_transform` / `values_transform` — coerce types during the pivot, rather than
  in a follow-up `mutate()`.
- `values_fn` — aggregate when `pivot_wider()` finds duplicate key combinations.
  Without it, duplicates produce list-columns and a warning.
- `cols_vary` — controls output row ordering when pivoting several column sets.

If `pivot_wider()` warns about duplicate identifiers, the fix is almost always to
find them first rather than to silence the warning:

```r
df %>% count(id, category) %>% filter(n > 1)
```

## Splitting and combining columns

`separate_wider_delim()` and `separate_wider_regex()` replace the deprecated
`separate()`:

```r
df %>% separate_wider_delim(full_name, delim = " ", names = c("first", "last"))
df %>% separate_wider_regex(code, patterns = c(region = "[A-Z]{2}", "-", id = "\\d+"))
```

Use `too_few` and `too_many` to control what happens on ragged input — the default is
to error, which is usually what you want. `unite()` is the inverse.

## Nesting and list-columns

`nest()` collapses groups into a list-column of data frames, which is the tidy way to
fit many models or apply a function per group:

```r
df %>%
  nest(.by = group) %>%
  mutate(model = map(data, \(d) lm(y ~ x, data = d)))
```

`unnest()` reverses it. `unnest_longer()` expands a list-column down the rows;
`unnest_wider()` expands it across columns — the usual tool for JSON-shaped data.

## Missing values

```r
df %>% drop_na()                  # drop rows with any NA
df %>% drop_na(amount)            # drop rows missing a specific column
df %>% complete(group, date)      # make implicit missing combinations explicit
df %>% replace_na(list(n = 0))    # substitute a value
```

`complete()` is the tool when a group is missing entirely from the data and its absence
is meaningful — for example, filling in months with no sales before plotting a time
series.

`fill()` carries values forward or backward, and takes `.by` as of tidyr 1.3.2, so
grouping no longer needs `group_by()`:

```r
df %>% fill(value, .direction = "down", .by = station)
```

To replace values *within* an existing column rather than fill gaps, see
[filtering-and-recoding.md](filtering-and-recoding.md) —
`replace_values()` supersedes `replace_na()` and `na_if()` for that purpose.
