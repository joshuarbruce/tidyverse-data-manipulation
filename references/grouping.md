# Grouping and Column-wise Operations

`.by` / `group_by()`, `across()`, `pick()`, `reframe()`, and row-wise work. Read this
for any task that aggregates, computes within groups, or applies a function across
several columns.

## Per-operation grouping with `.by`

`.by` arrived in dplyr 1.1 and became **stable in dplyr 1.2.0**. Prefer it whenever
the grouping applies to a single verb:

```r
df %>% summarise(mean_val = mean(x), .by = group_col)
df %>% summarise(n = n(), .by = c(region, year))     # multiple grouping columns
```

**`.by` always returns ungrouped data.** Never write `ungroup()` after it — there is
nothing to undo:

```r
# correct
df %>% summarise(total = sum(x), .by = region)

# redundant — the result was never grouped
df %>% summarise(total = sum(x), .by = region) %>% ungroup()
```

Unlike `group_by()`, `.by` does not sort the result. Add `arrange()` if order matters.

### When `group_by()` is still right

Use `group_by()` when the grouping must **persist across several verbs**:

```r
df %>%
  group_by(patient_id) %>%
  filter(visit == max(visit)) %>%
  mutate(days_since_first = as.numeric(visit_date - min(visit_date))) %>%
  ungroup()
```

With `.by` this would repeat the grouping on every verb. Rule of thumb: one verb →
`.by`; several verbs sharing a grouping → `group_by()` + `ungroup()`.

### Grouping persists until you remove it

`group_by()` attaches grouping to the data frame itself. **Every subsequent verb
inherits it** — `filter()`, `mutate()`, `arrange()`, and `select()` all return a still-
grouped result — and it travels with the object out of a function and into whatever
runs next. `ungroup()` is the only thing that clears it.

Inspect it when unsure:

```r
group_vars(df)   # character vector of grouping columns, empty if ungrouped
n_groups(df)     # how many groups
```

The trap is that grouped results are usually *valid*, just computed over the wrong
scope, so nothing errors:

```r
totals <- df %>%
  group_by(region, year) %>%
  summarise(total = sum(sales))     # still grouped by region!

totals %>% mutate(pct = total / sum(total))   # pct within region, not overall
```

**`summarise()` removes only the last grouping variable**, so grouping by two columns
leaves the result grouped by the first. dplyr says so in a message, which is easy to
miss in a long script. Be explicit instead:

```r
df %>% group_by(region, year) %>% summarise(total = sum(sales), .groups = "drop")
```

`.groups` accepts `"drop"` (ungrouped — usually what you want), `"drop_last"` (the
default), or `"keep"`.

None of this applies to `.by`, which never returns grouped data — that is the main
reason to prefer it:

```r
df %>% summarise(total = sum(sales), .by = c(region, year))   # 0 groups
```

Always `ungroup()` when finished with a `group_by()` chain. A silently grouped data
frame returned from a function is a classic source of wrong results downstream.

### Grouped mutate (window functions)

`.by` inside `mutate()` computes per group and writes back to every row:

```r
df %>% mutate(
  group_mean   = mean(value),
  pct_of_group = value / sum(value),
  .by = region
)
```

Combine several mutations into one `mutate()` when they share a `.by`. Keep them
separate only when the `.by` differs, or when a later column depends on an earlier one.

### Grouped filter

```r
df %>% filter(value == max(value), .by = region)   # top row(s) per region
df %>% filter(n() >= 3, .by = region)              # keep groups with 3+ rows
```

`slice_max()` / `slice_min()` are clearer when you want a fixed number of rows and
need explicit tie handling. Note this family takes `by`, not `.by`:

```r
df %>% slice_max(value, n = 1, by = region, with_ties = FALSE)
```

## `across()` — one function, many columns

```r
df %>% mutate(across(where(is.character), str_trim))
df %>% summarise(across(starts_with("val_"), mean, na.rm = TRUE), .by = group)
```

Several functions at once, with control over output names:

```r
df %>% summarise(
  across(c(height, mass), list(mean = mean, sd = sd), .names = "{.col}_{.fn}"),
  .by = species
)
```

Pass a character vector of column names through `all_of()`:

```r
cols <- c("height", "mass")
df %>% summarise(across(all_of(cols), mean))
```

## `pick()` — select columns inside a verb

`pick()` returns a data frame of the selected columns, for functions that need several
columns at once. It replaces the older `cur_data()` / `cur_data_all()`:

```r
df %>% mutate(n_missing = rowSums(is.na(pick(everything()))))
```

## `reframe()` — summaries returning multiple rows

`summarise()` expects one value per group. When a computation returns a vector, use
`reframe()` (stable as of dplyr 1.2.0):

```r
df %>% reframe(q = quantile(x, c(.25, .5, .75)), .by = group)
```

Fighting a "must return size 1" error from `summarise()` is the signal to switch.

## Row-wise operations

`rowwise()` treats each row as its own group. Correct but slow — prefer a vectorized
alternative whenever one exists:

```r
# rowwise — clear, but slow on large data
df %>% rowwise() %>% mutate(total = sum(c_across(starts_with("q")))) %>% ungroup()

# vectorized — much faster
df %>% mutate(total = rowSums(pick(starts_with("q"))))
```

Base R already vectorizes several of these: `rowSums()`, `rowMeans()`, `pmin()`,
`pmax()`. Reserve `rowwise()` for genuinely non-vectorizable work, such as calling a
function that only accepts scalars.

## Counting

```r
df %>% count(region)                          # shorthand for group + tally
df %>% count(region, sort = TRUE)             # most frequent first
df %>% count(region, wt = sales)              # weighted
df %>% summarise(n = n(), .by = region)       # equivalent, more explicit
```

`n_distinct(x)` counts unique values. `count()` on the key columns is the quickest way
to check for duplicates before a join — see
[joins.md](joins.md).
