# Joins Reference

Examples use dplyr's `band_members` and `band_instruments`, which are built for this:
`Mick` appears only in members, `Keith` only in instruments, so every unmatched-row
behavior is visible.

```r
library(dplyr)

band_members       # name, band   — Mick, John, Paul
band_instruments   # name, plays  — John, Paul, Keith
```

Use `join_by()` for the `by` argument — it is clearer than a character vector and it
is the only form that supports inequality and rolling joins:

```r
left_join(band_members, band_instruments, by = join_by(name))
```

The character form `by = c("name" = "name")` is **not** deprecated or superseded —
dplyr's docs list it as an accepted value alongside `join_by()`. Prefer `join_by()`
anyway: it reads better, it is checked at the call site, and it is the only form that
expresses anything beyond simple equality.

## Join types

| Function | Rows kept |
|---|---|
| `inner_join()` | Only rows with matches in both tables |
| `left_join()` | All rows from left; NAs for unmatched right |
| `right_join()` | All rows from right; NAs for unmatched left |
| `full_join()` | All rows from both; NAs where no match |
| `semi_join()` | Left rows that have a match in right (no right columns added) |
| `anti_join()` | Left rows that have NO match in right |

## join_by() syntax

```r
# simple equality
left_join(band_members, band_instruments, by = join_by(name))

# different column names on each side
left_join(band_members, rename(band_instruments, who = name), by = join_by(name == who))
```

Inequality and rolling joins take comparison operators rather than `==`:

```r
events  <- tibble(id = 1:2, time = c(3, 12))
windows <- tibble(window = c("early", "late"), start = c(0, 10), end = c(10, 20))

# non-equi: match each event to the window containing it
left_join(events, windows, by = join_by(time >= start, time < end))

# rolling: match to the nearest earlier reading
readings <- tibble(t = c(1, 5, 9), value = c(10, 20, 30))
probes   <- tibble(t = c(4, 8))
left_join(probes, readings, by = join_by(closest(t >= t)))
```

## Cardinality checking

Declare the expected relationship so a surprise errors instead of silently changing
the row count:

```r
left_join(band_members, band_instruments,
          by = join_by(name), relationship = "many-to-one")

# Options: "one-to-one", "one-to-many", "many-to-one", "many-to-many"
```

## Handling unmatched rows

**`unmatched` guards the side whose rows would be dropped — not the side that gets
`NA`s.** This is the opposite of what most people assume, and it is the difference
between catching a broken lookup table and missing it entirely.

In a `left_join()`, every row of `x` is kept no matter what, so `unmatched = "error"`
says nothing about them; it errors only when a row of `y` finds no match and is
therefore discarded:

```r
# Mick has no instrument (gets NA); Keith is in no band (gets dropped)
left_join(band_members, band_instruments, by = join_by(name))
#> Mick  Stones  NA
#> John  Beatles guitar
#> Paul  Beatles bass

left_join(band_members, band_instruments, by = join_by(name), unmatched = "error")
#> Error: Each row of `y` must be matched by `x`   <- about Keith, not Mick
```

So the common worry — "did my lookup table cover every row of my main table?" — is
**not** what `unmatched = "error"` checks in a left join. Mick still silently becomes
`NA`. Use `anti_join()` for that:

```r
anti_join(band_members, band_instruments, by = join_by(name))
#> Mick  Stones     <- the real check for rows that found no match
```

`inner_join()` drops from both sides, so `unmatched = "error"` there checks both.
`right_join()` checks `x`. Match the tool to the question: `unmatched` for rows being
dropped, `anti_join()` for rows being `NA`-filled, `relationship` for row counts.

## Pitfalls

**`NA` keys match each other by default.** dplyr's default is
`na_matches = "na"`, so a row with `NA` in the key joins to another row with `NA`
in the key — as if `NA` were an ordinary value:

```r
x <- tibble(k = c("a", NA), vx = 1:2)
y <- tibble(k = c("a", NA), vy = c("A", "MISSING"))

left_join(x, y, by = join_by(k))                       # NA row matches -> "MISSING"
left_join(x, y, by = join_by(k), na_matches = "never") # NA row gets NA
```

**This is the opposite of SQL**, where `NULL` never equals `NULL` and such rows never
match. If you are translating a query or reasoning from SQL habits, set
`na_matches = "never"` explicitly. Missing keys are rarely meaningful entities, so
joining them together usually creates spurious matches.

**Row counts can silently grow.** If both sides have duplicate key values you get a
Cartesian product for those keys — 2 rows joined to 2 rows returns 4. dplyr does warn
about an unexpected many-to-many relationship, but a warning is easy to lose in a long
script. Declare the expectation so it errors instead:

```r
dup_x <- tibble(k = c("a", "a"), vx = 1:2)
dup_y <- tibble(k = c("a", "a"), vy = c("A", "B"))

nrow(left_join(dup_x, dup_y, by = join_by(k)))   # 4, from 2 x 2
left_join(dup_x, dup_y, by = join_by(k), relationship = "many-to-one")   # errors
```

Check row counts before and after any join. A join that changes `nrow()` when you did
not expect it to is the single most common source of wrong analysis results.

**Other traps:**

- **Factor vs character keys coerce silently.** Joining a factor key to a character
  key does not error or warn — vctrs finds a common type and the result comes back
  `character`. That quietly discards the factor's level order, so anything downstream
  relying on it (bar order in a plot, an ordered comparison) changes behavior with no
  signal. Align the types deliberately with `as.character()` or `as_factor()` before
  joining. Two factors with *different levels* also join fine; the result keeps the
  union of the levels.
- **Column name conflicts** — if both tables have a non-key column with the same
  name, dplyr adds `.x` and `.y` suffixes. Use `suffix = c("_left", "_right")`
  to make them meaningful, or rename before joining.

## Checking a join

Two cheap checks catch most join bugs:

```r
# 1. which rows found no match?
anti_join(band_members, band_instruments, by = join_by(name))
#> Mick  Stones

# 2. which keys are duplicated? duplicates are what multiply rows
inst_dup <- tibble(
  name  = c("John", "Paul", "Paul"),
  plays = c("guitar", "bass", "piano")
)

inst_dup %>% count(name) %>% filter(n > 1)
#> Paul  2

nrow(left_join(band_members, inst_dup, by = join_by(name)))   # 4, not 3
```

Run the first before trusting any enrichment join, and the second before joining
anything you did not create yourself. The duplicate check is the more valuable of the
two: an unmatched row is visible as an `NA`, whereas a duplicated key quietly adds
rows that then inflate every count and sum computed downstream.
