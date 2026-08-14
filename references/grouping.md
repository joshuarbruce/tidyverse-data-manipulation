# Grouping and Column-wise Operations

`.by` / `group_by()`, `across()`, `pick()`, `reframe()`, and row-wise work. Read this
for any task that aggregates, computes within groups, or applies a function across
several columns.

## Per-operation grouping with `.by`

`.by` arrived in dplyr 1.1 and became **stable in dplyr 1.2.0**. Prefer it whenever
the grouping applies to a single verb:

```r
library(dplyr)

starwars %>% summarise(avg_height = mean(height, na.rm = TRUE), .by = species)
starwars %>% summarise(n = n(), .by = c(species, gender))   # multiple grouping columns
```

**`.by` always returns ungrouped data.** Never write `ungroup()` after it — there is
nothing to undo:

```r
# correct
starwars %>% summarise(n = n(), .by = species)

# redundant — the result was never grouped
starwars %>% summarise(n = n(), .by = species) %>% ungroup()
```

Unlike `group_by()`, `.by` does not sort the result. Add `arrange()` if order matters.

### When `group_by()` is still right

Use `group_by()` when the grouping must **persist across several verbs**:

```r
starwars %>%
  group_by(species) %>%
  filter(!is.na(height)) %>%
  mutate(height_vs_species_max = height - max(height)) %>%
  ungroup()
```

With `.by` this would repeat the grouping on every verb. Rule of thumb: one verb →
`.by`; several verbs sharing a grouping → `group_by()` + `ungroup()`.

### Grouping persists until you remove it

`group_by()` attaches grouping to the data frame itself. **Every subsequent verb
inherits it** — `filter()`, `mutate()`, `arrange()` and `select()` all return a still-
grouped result — and it travels with the object out of a function and into whatever
runs next. `ungroup()` is the only thing that clears it.

Inheriting the grouping and *acting* on it are two different things, and `arrange()`
is the verb where they come apart: it stays grouped but sorts the whole table anyway
unless you pass `.by_group = TRUE`. See
[`arrange()` ignores grouping](#arrange-ignores-grouping) below.

Inspect it when unsure:

```r
g <- group_by(starwars, species)
group_vars(g)   # "species" — empty character vector if ungrouped
n_groups(g)     # how many groups
```

The trap is that grouped results are usually *valid*, just computed over the wrong
scope, so nothing errors:

```r
totals <- starwars %>%
  group_by(species, gender) %>%
  summarise(total = sum(height, na.rm = TRUE))   # still grouped by species!

totals %>% mutate(pct = total / sum(total))      # pct within species, not overall
```

**`summarise()` removes only the last grouping variable**, so grouping by two columns
leaves the result grouped by the first. This is documented, not a quirk — `?summarise`
states that `.groups` defaults to `"drop_last"`, described as "drops the last level of
grouping. This was the only supported option before version 1.0.0."

Be explicit rather than relying on the default:

```r
starwars %>%
  group_by(species, gender) %>%
  summarise(total = sum(height, na.rm = TRUE), .groups = "drop")
```

`.groups` (still marked experimental) accepts:

| Value | Result |
|---|---|
| `"drop"` | Fully ungrouped — usually what you want |
| `"drop_last"` | Drops the last grouping level (the default) |
| `"keep"` | Same grouping as the input |
| `"rowwise"` | Each row its own group |

**Do not rely on the message to catch this.** dplyr informs you how the result is
grouped, but the docs list three cases where it stays silent — when the result is
already ungrouped, when `options(dplyr.summarise.inform = FALSE)` is set, and **when
`summarise()` is called from a function inside a package**. The last two are exactly
the situations where a silently grouped result travels furthest before anyone notices,
and plenty of projects set that option globally to quiet their logs.

None of this applies to `.by`, which never returns grouped data — that is the main
reason to prefer it:

```r
starwars %>% summarise(total = sum(height, na.rm = TRUE), .by = c(species, gender))
```

Always `ungroup()` when finished with a `group_by()` chain. A silently grouped data
frame returned from a function is a classic source of wrong results downstream.

### Grouped mutate (window functions)

`.by` inside `mutate()` computes per group and writes back to every row:

```r
starwars %>% mutate(
  species_mean = mean(height, na.rm = TRUE),
  pct_of_max   = height / max(height, na.rm = TRUE),
  .by = species
)
```

Combine several mutations into one `mutate()` when they share a `.by`. A later column
may reference an earlier one in the same call — that is supported and idiomatic, and it
respects the grouping:

```r
starwars %>% mutate(
  species_mean = mean(height, na.rm = TRUE),
  vs_mean      = height - species_mean,   # uses the column defined just above
  .by = species
)
```

Split into separate `mutate()` calls only when the `.by` differs between them.

### Grouped filter

```r
starwars %>% filter(height == max(height, na.rm = TRUE), .by = species)  # tallest per species
starwars %>% filter(n() >= 3, .by = species)                             # groups with 3+ rows
```

`slice_max()` / `slice_min()` are clearer when you want a fixed number of rows and
need explicit tie handling. Note this family takes `by`, not `.by`:

```r
starwars %>% slice_max(height, n = 1, by = species, with_ties = FALSE)
```

**`with_ties` defaults to `TRUE`, so `n = 1` can return more than one row per group.**
Nothing warns — a "top 1 per group" that quietly returns two rows for a tied group will
inflate every downstream count. Set `with_ties = FALSE` when you need exactly `n`, and
add a deterministic tiebreak (`arrange()` first, or include a unique column) so the row
kept is not arbitrary.

### `arrange()` ignores grouping

Where `mutate()`, `filter()`, `summarise()` and `slice_*()` all compute within groups,
`arrange()` does **not** sort within them unless you ask:

```r
starwars %>% group_by(species) %>% arrange(height)                   # sorts whole table
starwars %>% group_by(species) %>% arrange(height, .by_group = TRUE) # sorts within species
```

This is a documented deliberate choice, not a bug, but it surprises people who assume
grouping applies uniformly. It matters whenever sort order is load-bearing — before
`slice_head()`, before `fill()`, or when computing `lag()`/`lead()` within a group.

`arrange()` also always sorts `NA` to the end, **even under `desc()`**:

```r
d <- tibble(v = c(2, NA, 1))
arrange(d, v)$v        # 1, 2, NA
arrange(d, desc(v))$v  # 2, 1, NA   <- not NA first
```

That is usually what you want, but it means "the last row" is not reliably the largest
value when the column has missing data.

## `across()` — one function, many columns

```r
starwars %>% mutate(across(where(is.character), stringr::str_trim))
starwars %>% summarise(across(c(height, mass), \(x) mean(x, na.rm = TRUE)), .by = species)
```

**Pass extra arguments with a lambda, not through `...`.** The `...` argument of
`across()` was deprecated in dplyr 1.1.0:

```r
across(a:b, mean, na.rm = TRUE)        # deprecated — warns
across(a:b, \(x) mean(x, na.rm = TRUE)) # current
```

A bare function name with no extra arguments (`across(where(is.character), str_trim)`)
is still fine.

Several functions at once, with control over output names:

```r
starwars %>% summarise(
  across(c(height, mass), list(mean = mean, sd = sd), .names = "{.col}_{.fn}"),
  .by = species
)
```

Pass a character vector of column names through `all_of()`:

```r
cols <- c("height", "mass")
starwars %>% summarise(across(all_of(cols), \(x) mean(x, na.rm = TRUE)))
```

## `pick()` — select columns inside a verb

`pick()` returns a data frame of the selected columns, for functions that need several
columns at once. It replaces the older `cur_data()` / `cur_data_all()`:

```r
starwars %>% mutate(n_missing = rowSums(is.na(pick(everything())))) %>% select(name, n_missing)
```

## `reframe()` — summaries returning multiple rows

`summarise()` expects one value per group. When a computation returns a vector, use
`reframe()` (stable as of dplyr 1.2.0):

```r
probs <- c(.25, .5, .75)

starwars %>%
  filter(!is.na(height)) %>%
  reframe(p = probs, height = quantile(height, probs), .by = gender)
```

Carry the labels through as their own column. `reframe(q = quantile(x, probs))` alone
returns three unlabelled numbers per group, and nothing in the result says which is the
median — a table that is correct and unusable.

Fighting a "must return size 1" error from `summarise()` is the signal to switch.

## Row-wise operations

`rowwise()` treats each row as its own group. Correct but slow — prefer a vectorized
alternative whenever one exists:

```r
# rowwise — clear, but slow on large data
mtcars %>% rowwise() %>% mutate(total = sum(c_across(c(mpg, hp)))) %>% ungroup()

# vectorized — much faster
mtcars %>% mutate(total = rowSums(pick(c(mpg, hp))))
```

Base R already vectorizes several of these: `rowSums()`, `rowMeans()`, `pmin()`,
`pmax()`. Reserve `rowwise()` for genuinely non-vectorizable work, such as calling a
function that only accepts scalars.

## Counting

```r
starwars %>% count(species)                        # shorthand for group + tally
starwars %>% count(species, sort = TRUE)           # most frequent first
starwars %>% count(species, wt = height)           # weighted
starwars %>% summarise(n = n(), .by = species)     # equivalent, more explicit
```

`n_distinct(x)` counts unique values. `count()` on the key columns is the quickest way
to check for duplicates before a join — see
[joins.md](joins.md).

**`n_distinct()` counts `NA` as one of the distinct values** — `n_distinct(c(1, NA, NA))`
is 2, not 1. Pass `na.rm = TRUE` when you mean "how many real values are there". The
same applies to `count()`, which gives `NA` its own row; that row is easy to overlook
in a long result and inflates any total computed from it.
