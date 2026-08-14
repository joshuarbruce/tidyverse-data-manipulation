# Visualization with ggplot2

Read this for building plots, and for the ggplot2 4.0 deprecations that make older
example code fail or warn.

## Build plots layer by layer

```r
library(ggplot2)
library(dplyr)

by_year <- economics %>%
  mutate(year = lubridate::year(date)) %>%
  summarise(unemploy = mean(unemploy), .by = year)

ggplot(by_year, aes(x = year, y = unemploy)) +
  geom_line(linewidth = 0.8) +
  geom_point(size = 2) +
  scale_y_continuous(labels = scales::comma) +
  labs(title = "Mean unemployment by year", x = NULL, y = "Unemployed (thousands)") +
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
counts <- tibble::tibble(cat = c("a", "b"), n = c(5, 10))

ggplot(counts, aes(cat)) + geom_bar()        # every bar height 1 — it counted the rows
ggplot(counts, aes(cat, n)) + geom_col()     # heights 5 and 10 — what you wanted

ggplot(mpg, aes(class)) + geom_bar()         # correct use: let ggplot do the counting
```

Use `geom_col()` when the data is already aggregated, `geom_bar()` when you want
ggplot2 to do the counting. `geom_bar(stat = "identity")` is equivalent to `geom_col()`
but the latter says it more clearly.

## Reshape before plotting

ggplot2 wants long data: one row per observation, with the variable that distinguishes
series in its own column. Most "how do I plot these five columns" questions are
reshaping questions — see [reshaping.md](reshaping.md):

```r
long <- tidyr::billboard %>%
  tidyr::pivot_longer(starts_with("wk"), names_to = "week",
                      values_to = "rank", values_drop_na = TRUE) %>%
  filter(artist %in% c("2Pac", "3 Doors Down"))

ggplot(long, aes(x = week, y = rank, group = track, color = track)) +
  geom_line() +
  theme_minimal()
```

Note that `week` here is character (`"wk1"`, `"wk2"`, …), so the x-axis sorts
alphabetically — `wk10` lands before `wk2`. Whenever a categorical axis comes out in
the wrong order, the fix is to control the factor, not the plot: convert with explicit
`levels`, or use `fct_reorder()` to order by value. See
[factors-and-dates.md](factors-and-dates.md).

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
p <- ggplot(mpg, aes(displ, hwy)) + geom_point()
ggsave(tempfile(fileext = ".png"), plot = p, width = 8, height = 5, dpi = 300)
```

`ggsave()` defaults to the last plot displayed, but naming the plot explicitly is
safer in a script. Size in inches is set by `width`/`height`; text size does **not**
scale with them, so a plot saved small will have proportionally large text.

## Accessibility

Default categorical palettes are not colorblind-safe. Use
`scale_color_viridis_d()` / `scale_fill_viridis_d()`, or supply a colorblind-safe
palette explicitly, and do not rely on color as the only channel distinguishing series
— pair it with linetype, shape, or direct labels.
