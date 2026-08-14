# Joins Reference

Use `join_by()` for the `by` argument — it is clearer than a character vector and it
is the only form that supports inequality and rolling joins:

```r
left_join(orders, customers, by = join_by(customer_id))
left_join(events, windows, by = join_by(time >= start, time < end))  # inequality
```

The older `by = c("cust_id" = "id")` character form still works but is superseded;
`join_by(cust_id == id)` reads better and is checked at the call site.

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
# Simple equality
left_join(orders, customers, by = join_by(customer_id))

# Different column names
left_join(orders, customers, by = join_by(cust_id == id))

# Inequality (non-equi) join
left_join(events, windows, by = join_by(time >= start, time < end))

# Rolling join — nearest match
left_join(readings, calibrations, by = join_by(closest(time >= cal_time)))
```

## Cardinality checking

Declare the expected relationship to catch surprises early:

```r
inner_join(a, b, by = join_by(id),
           relationship = "many-to-one")   # many a rows per b row

# Options: "one-to-one", "one-to-many", "many-to-one", "many-to-many"
```

## Handling unmatched rows

```r
left_join(a, b, by = join_by(id), unmatched = "error")  # error if any b key has no match in a
```

## Pitfalls

**`NA` keys match each other by default.** dplyr's default is
`na_matches = "na"`, so a row with `NA` in the key joins to another row with `NA`
in the key — as if `NA` were an ordinary value:

```r
left_join(a, b, by = join_by(k))                      # NA row matches the NA row
left_join(a, b, by = join_by(k), na_matches = "never") # NA row gets NA columns
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
left_join(a, b, by = join_by(k), relationship = "many-to-one")
```

Check row counts before and after any join. A join that changes `nrow()` when you did
not expect it to is the single most common source of wrong analysis results.

**Other traps:**

- **Factor vs character** — join keys must be the same type. Use `as.character()`
  or `as_factor()` to align.
- **Column name conflicts** — if both tables have a non-key column with the same
  name, dplyr adds `.x` and `.y` suffixes. Use `suffix = c("_orders", "_items")`
  to make them meaningful, or rename before joining.

## Checking for unmatched rows after a join

```r
# Rows in orders with no matching customer
anti_join(orders, customers, by = join_by(customer_id))

# See which keys are duplicated
orders %>% count(customer_id) %>% filter(n > 1)
```
