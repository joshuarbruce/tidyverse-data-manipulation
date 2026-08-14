# Recoding and Replacing Values

dplyr 1.2.0 completed recoding into a family of four functions. Read this file when a
task involves mapping values, conditional replacement, lookup tables, or migrating
away from `case_match()` / `recode()`.

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
