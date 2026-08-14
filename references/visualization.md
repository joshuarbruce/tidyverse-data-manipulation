# Visualization with ggplot2

Read this for building plots, and for the ggplot2 4.0 deprecations that make older
example code fail or warn.

## Build plots layer by layer

```r
ggplot(df, aes(x = year, y = revenue, color = region)) +
  geom_line(linewidth = 0.8) +
  geom_point(size = 2) +
  scale_y_continuous(labels = scales::comma) +
  labs(title = "Revenue by region", x = NULL, y = "Revenue ($)") +
  theme_minimal()
```

Key patterns:

- Put shared aesthetics in `ggplot()`; override per layer as needed.
- Aesthetics that **map data to visual properties** go inside `aes()`; fixed values go
  outside it. `color = "blue"` inside `aes()` creates a one-level legend labelled
  "blue" — a common and confusing mistake.
- `facet_wrap()` / `facet_grid()` for small multiples.
- `scale_*_*()` to control axes, colors, and sizes; the `scales` package supplies
  label formatters (`comma`, `percent`, `dollar`, `label_number()`).
- `labs()` sets every title, axis label, and legend name in one call.
- `theme_minimal()` / `theme_bw()` as clean starting points; `theme()` for fine control.

Note the size argument for lines is **`linewidth`**, not `size` — this changed in
ggplot2 3.4 and old code using `size` for lines now warns.

**`geom_bar()` counts rows; `geom_col()` plots the value.** Both render a bar chart, so
the wrong one produces a valid-looking chart with wrong heights and no warning:

```r
df <- tibble(cat = c("a", "b"), n = c(5, 10))

ggplot(df, aes(cat)) + geom_bar()        # every bar height 1 — it counted the rows
ggplot(df, aes(cat, n)) + geom_col()     # heights 5 and 10 — what you wanted
```

Use `geom_col()` when the data is already aggregated, `geom_bar()` when you want
ggplot2 to do the counting. `geom_bar(stat = "identity")` is equivalent to `geom_col()`
but the latter says it more clearly.

## Reshape before plotting

ggplot2 wants long data: one row per observation, with the variable that distinguishes
series in its own column. Most "how do I plot these five columns" questions are
reshaping questions — see [reshaping.md](reshaping.md):

```r
df_long <- df_wide %>%
  pivot_longer(cols = Jan:Dec, names_to = "month", values_to = "sales") %>%
  mutate(month = factor(month, levels = month.abb))

ggplot(df_long, aes(x = month, y = sales, group = region, color = region)) +
  geom_line() +
  theme_minimal()
```

Converting `month` to a factor with explicit levels is what stops the x-axis from
sorting alphabetically. Controlling categorical order is a forcats job — see
[factors-and-dates.md](factors-and-dates.md), especially
`fct_reorder()` for ordering bars by value.

## Deprecated and superseded forms

ggplot2 4.0 formalized several long-standing deprecations. Avoid these:

| Old | Current |
|---|---|
| `..var..` or `stat(var)` inside `aes()` | `after_stat(var)` |
| `qplot()` | `ggplot()` + geoms |
| `coord_flip()` | `orientation = "y"` on the geom |
| `geom_errorbarh()` | `geom_errorbar(orientation = "y")` |
| `size` for line width | `linewidth` |
| `trans` in scales | `transform` |
| `fortify()` on model objects | `broom::augment()` |

`coord_flip()` and `geom_errorbarh()` still work but are superseded; the `..var..`
notation and `qplot()` are deprecated and warn.

## Saving

```r
ggsave("plot.png", plot = p, width = 8, height = 5, dpi = 300)
```

`ggsave()` defaults to the last plot displayed, but naming the plot explicitly is
safer in a script. Size in inches is set by `width`/`height`; text size does **not**
scale with them, so a plot saved small will have proportionally large text.

## Accessibility

Default categorical palettes are not colorblind-safe. Use
`scale_color_viridis_d()` / `scale_fill_viridis_d()`, or supply a colorblind-safe
palette explicitly, and do not rely on color as the only channel distinguishing series
— pair it with linetype, shape, or direct labels.
