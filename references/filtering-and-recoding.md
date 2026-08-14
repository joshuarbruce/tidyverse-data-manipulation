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
library(dplyr)

# 5 starwars characters have NA hair_color
starwars %>% filter(hair_color != "blond" | is.na(hair_color))  # negation + NA guard
starwars %>% filter_out(hair_color == "blond")                  # same 84 rows, clearer
```

Both take `.by` for per-operation grouping:

```r
starwars %>% filter_out(n() < 3, .by = species)   # drop species with fewer than 3
```

Reach for `filter_out()` whenever you catch yourself negating a condition. Two
different benefits, worth keeping straight:

- With `!=` or `!` on a comparison, it **fixes an NA bug**: `filter(x != 1)` drops the
  `NA` rows, `filter_out(x == 1)` keeps them.
- With `%in%`, it is **only clearer, not safer** — `%in%` returns `FALSE` rather than
  `NA` for missing values, so `filter(!(x %in% v))` and `filter_out(x %in% v)` return
  exactly the same rows.

### Conditions that return `NA` silently drop rows

Any predicate returning `NA` counts as `FALSE` in `filter()`, so those rows vanish with
no warning. This bites most often with `str_detect()`, which propagates `NA`:

```r
s <- tibble(x = c("apple", NA, "banana"))
stringr::str_detect(s$x, "an")            # FALSE, NA, TRUE

s %>% filter(stringr::str_detect(x, "an"))      # 1 row — the NA row is gone
s %>% filter_out(stringr::str_detect(x, "an"))  # 2 rows — the NA row is kept
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
starwars %>% filter(when_any(
  species == "Droid" & between(height, 90, 200),
  species == "Human" & between(mass, 70, 90)
))
```

By default `NA` propagates as it does through `|` and `&`. Pass `na_rm = TRUE` to
force a `TRUE`/`FALSE` result.

## Recoding and replacing

Two questions decide which of the four functions you want: are you matching on
**conditions** or on **values**, and are you building a **new** vector or **updating**
an existing one?

| | Match on conditions | Map old values to new |
|---|---|---|
| **Create** a new vector | `case_when()` | `recode_values()` |
| **Update** an existing vector | `replace_when()` | `replace_values()` |

The distinction that matters most is what happens to inputs you did not mention. The
`case_*`/`recode_*` pair returns `NA` for them unless you supply a default — no error,
just missing values appearing where you did not expect them. The `replace_*` pair
leaves them at their original value. Reaching for a `replace_*()` function is usually
what people actually mean when they write a `case_when()` ending in `.default = x`.

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
x <- 1:70   # needs to reach 35 for the first branch to fire at all

# condition-based — first match wins, so order the branches most specific first
case_when(
  x %% 35 == 0 ~ "fizz buzz",
  x %% 5  == 0 ~ "fizz",
  x %% 7  == 0 ~ "buzz",
  .default = as.character(x)
)

# value-based
c(1, 2, 3) %>% recode_values(1 ~ "low", 2 ~ "mid", 3 ~ "high")
```

Group several inputs to one output by passing a vector on the left:

```r
x <- c("UNC", "Duke", "NCSU")
# note: `default`, not `.default` — see the argument-prefix warning above
x %>% recode_values(c("UNC", "Chapel Hill") ~ "UNC Chapel Hill", default = x)
```

## Updating an existing vector

`replace_when()` and `replace_values()` are type stable, size stable, pipe well, and
state intent more clearly than a `case_when()` with a catch-all default:

```r
# condition-based update
starwars %>%
  mutate(sex = replace_when(sex, is.na(sex) ~ "unknown")) %>%
  count(sex)

# value-based update — everything not named passes through untouched
c("UNC", "Duke University", "NCSU") %>% replace_values(
  c("UNC", "Chapel Hill")      ~ "UNC Chapel Hill",
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
lookup <- tibble(abbr = c("NC", "SC"), full = c("North Carolina", "South Carolina"))
state  <- c("NC", "SC", "NC")

recode_values(state,  from = lookup$abbr, to = lookup$full)
replace_values(state, from = lookup$abbr, to = lookup$full)
```

This is the main reason these functions replace both `case_match()` and `recode()`.
It also composes with `across()` for multi-column recoding:

```r
key <- tibble(code = c(1, 2), label = c("yes", "no"))
tibble(q1 = c(1, 2), q2 = c(2, 1)) %>%
  mutate(across(starts_with("q"), \(x) recode_values(x, from = key$code, to = key$label)))
```

## Failing loudly on unmatched values

When you believe every case is handled, prefer `unmatched = "error"` over supplying a
default. A default silently buckets values you did not anticipate; an error surfaces
them while you can still fix the mapping:

```r
# errors because 3 is unmapped
recode_values(c(1, 2, 3), 1 ~ "low", 2 ~ "mid", unmatched = "error")
```

`case_when()` offers the same guard, spelled `.unmatched = "error"` with the dot. Use
it wherever an unmapped category is a data-quality signal rather than an expected case.

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
x <- c(4, -1)
case_when(x > 0 ~ sqrt(x), .default = 0)   # warns "NaNs produced" — sqrt(-1) still runs
```

The result is correct — unmatched rows take the default — but the warning is real and
signals wasted computation on values you thought the condition excluded.
**`replace_when()` behaves the same way**; it is not a way out. The fix is to guard the
*input* so every branch is safe to evaluate everywhere:

```r
case_when(x > 0 ~ sqrt(pmax(x, 0)), .default = 0)   # no warning
```

Or filter the offending rows out before recoding.

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
