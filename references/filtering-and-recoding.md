# Filtering, Recoding, and Replacing Values

dplyr 1.2.0 expanded the `filter()` family and completed recoding into a family of
four. Read this file when a task involves dropping rows, combining conditions, mapping
values, conditional replacement, lookup tables, or migrating away from `case_match()`
/ `recode()`.

## Dropping rows: `filter()` vs `filter_out()`

Use `filter()` to say which rows to **keep**, `filter_out()` to say which to **drop**.

Both treat `NA` like `FALSE`. Because they keep opposite sides of that test,
`filter()` *discards* rows where the condition is `NA` while `filter_out()` *retains*
them. That is what removes the `is.na()` guard a negated condition normally needs:

```r
df %>% filter(count != 0 | is.na(count))   # negated condition + explicit NA guard
df %>% filter_out(count == 0)              # same result, states the intent directly
```

Both take `.by` for per-operation grouping:

```r
df %>% filter_out(n() < 3, .by = region)   # drop small groups
```

Reach for `filter_out()` whenever you catch yourself writing `!=`, `!`, or
`%in%` with a leading `!` — the positive form is nearly always clearer, and it removes
a whole class of NA bugs.

### Conditions that return `NA` silently drop rows

Any predicate returning `NA` counts as `FALSE` in `filter()`, so those rows vanish with
no warning. This bites most often with `str_detect()`, which propagates `NA`:

```r
s <- c("apple", NA, "banana")
str_detect(s, "an")                    # TRUE, NA, TRUE

tibble(s) %>% filter(str_detect(s, "an"))      # 1 row — the NA row is gone
tibble(s) %>% filter_out(str_detect(s, "an"))  # 2 rows — the NA row is kept
```

Whenever a filter shrinks the data more than expected, count the `NA`s in the columns
the condition touches before assuming the logic is wrong.

**`%in%` never returns `NA`** — it returns `FALSE`, because it asks about set
membership rather than equality:

```r
NA %in% c(1, 2)   # FALSE
NA == 1           # NA
```

That makes `x %in% values` safe from NA propagation, but it also means
`filter(x %in% c(1))` silently drops `NA` rows. If missing values should be kept, say
so: `filter(x %in% c(1) | is.na(x))`, or invert with `filter_out()`.

## Combining conditions: `when_any()` / `when_all()`

These are elementwise versions of `any()` and `all()`: `when_any(x, y, z)` is
`x | y | z`, and `when_all(x, y, z)` is `x & y & z`.

Their real value is inside `filter()` / `filter_out()`, where comma-separated
arguments are otherwise combined with `&`. `when_any()` lets you write OR conditions
at the same indentation level, without wrapping parentheses:

```r
countries %>% filter(when_any(
  name %in% c("US", "CA") & between(score, 200, 300),
  name %in% c("PR", "RU") & between(score, 100, 200)
))
```

By default `NA` propagates as it does through `|` and `&`. Pass `na_rm = TRUE` to
force a `TRUE`/`FALSE` result.

## Recoding and replacing

## Choosing the right function

Two questions decide it: are you matching on **conditions** or on **values**, and are
you building a **new** vector or **updating** an existing one?

| | Match on conditions | Map old values to new |
|---|---|---|
| **Create** a new vector | `case_when()` | `recode_values()` |
| **Update** an existing vector | `replace_when()` | `replace_values()` |

The distinction that matters most: the `case_*`/`recode_*` pair needs to account for
every input (via a default), while the `replace_*` pair leaves anything unmatched at
its original value. Reaching for a `replace_*()` function is usually what people
actually mean when they write a `case_when()` ending in `.default = x`.

## Signatures

```r
case_when(..., .default = NULL, .unmatched = "default", .ptype = NULL, .size = NULL)
replace_when(x, ...)
recode_values(x, ..., from = NULL, to = NULL, default = NULL, unmatched = "default", ptype = NULL)
replace_values(x, ..., from = NULL, to = NULL)
```

**Argument prefixes are inconsistent between the two halves, and this is a common
error.** `case_when()` uses dot-prefixed arguments (`.default`, `.unmatched`,
`.ptype`, `.size`). `recode_values()` and `replace_values()` use undotted ones
(`default`, `unmatched`, `ptype`, `from`, `to`). The two `replace_*()` functions take
no default at all — unmatched values keep their original value by definition.

## Creating a new vector

```r
# condition-based
case_when(
  x %% 35 == 0 ~ "fizz buzz",
  x %% 5  == 0 ~ "fizz",
  x %% 7  == 0 ~ "buzz",
  .default = as.character(x)
)

# value-based
score %>% recode_values(1 ~ "low", 2 ~ "mid", 3 ~ "high")
```

Group several inputs to one output by passing a vector on the left:

```r
# note: `default`, not `.default` — see the argument-prefix warning above
name %>% recode_values(c("UNC", "Chapel Hill") ~ "UNC Chapel Hill", default = name)
```

## Updating an existing vector

`replace_when()` and `replace_values()` are type stable, size stable, pipe well, and
state intent more clearly than a `case_when()` with a catch-all default:

```r
# condition-based update
pets %>% mutate(type = replace_when(type, type == "dog" & age <= 2 ~ "puppy"))

# value-based update — everything not named passes through untouched
name %>% replace_values(
  c("UNC", "Chapel Hill")   ~ "UNC Chapel Hill",
  c("Duke", "Duke University") ~ "Duke"
)
```

`replace_when()` is effectively an enhanced `base::replace()`, and is the right tool
for conditionally rewriting part of a column inside `mutate()`.

## Using a lookup table

The `from` / `to` arguments take vectors, so an existing crosswalk can drive the
recode directly — no hand-written formulas, and the mapping stays data rather than
code:

```r
recode_values(state, from = lookup$abbr, to = lookup$name)
replace_values(state, from = lookup$abbr, to = lookup$name)
```

This is the main reason these functions replace both `case_match()` and `recode()`.
It also composes with `across()` for multi-column recoding:

```r
df %>% mutate(across(starts_with("q"), \(x) recode_values(x, from = key$code, to = key$label)))
```

## Failing loudly on unmatched values

When you believe every case is handled, prefer `unmatched = "error"` over supplying a
default. A default silently buckets values you did not anticipate; an error surfaces
them while you can still fix the mapping:

```r
# errors if score contains anything other than 1 or 2
recode_values(score, 1 ~ "low", 2 ~ "mid", unmatched = "error")
```

`case_when()` has the same argument as `.unmatched = "error"`. Use it in pipelines
where an unmapped category is a data-quality signal rather than an expected case.

## Traps

**Never use base `ifelse()` — it strips attributes, including the class.** A `Date`
comes back as the raw number of days since 1970, silently:

```r
d <- as.Date(c("2024-01-01", "2024-06-01"))

ifelse(d > as.Date("2024-03-01"), d, NA)   # 19875 — a number, class lost
if_else(d > as.Date("2024-03-01"), d, NA)  # 2024-06-01 — still a Date
```

The same applies to factors, times, and any vctrs-backed type. `dplyr::if_else()` is
type-stable and checks that both branches have the same type, so a mismatch errors
instead of coercing. Use it everywhere; `data.table::fifelse()` behaves the same way
inside data.table code.

**`case_when()` evaluates every right-hand side for every row**, not just the rows that
match. Branches that are invalid outside their own condition still run:

```r
case_when(x > 0 ~ sqrt(x), .default = 0)   # warns "NaNs produced" if x has negatives
```

The result is correct — unmatched rows take the default — but the warning is real and
signals wasted computation on values you filtered against. When a branch genuinely
cannot be evaluated on all rows, guard the input instead of the output, or use
`replace_when()` so untouched values are never recomputed.

## Migrating from older functions

| Old | Replacement | Status |
|---|---|---|
| `case_match()` | `recode_values()` / `replace_values()` | Deprecated in dplyr 1.2.0 — warns on use |
| `recode()` | `recode_values()` | Superseded |
| `recode_factor()` | `recode_values()` then `factor()` | Superseded |
| `case_when(..., .default = x)` used to patch one value | `replace_when()` / `replace_values()` | Still valid, but the `replace_*` form states intent better |

`case_match()` emits a lifecycle warning naming `recode_values()` as its replacement,
so existing code keeps working — but new code should not use it.

## Further reading

- `vignette("recoding-replacing")` — the authoritative walkthrough of all four
- [Tidyup 7](https://github.com/tidyverse/tidyups/blob/main/007-tidyverse-recoding-and-replacing.md)
  — the design rationale for why these four exist and `case_match()` did not survive
