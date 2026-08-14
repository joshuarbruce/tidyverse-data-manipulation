# Factors and Dates

forcats for categorical variables, lubridate for dates and times. Grouped together
because both solve the same class of problem: turning a messy vector into one with
meaningful, ordered structure.

## forcats — factors

The most common use is controlling order in a plot. Factor level order determines bar
order, legend order, and facet order, so reordering the factor is how you reorder a
chart:

```r
fct_reorder(f, x)                 # order levels by another variable (median by default)
fct_reorder(f, x, .fun = max)     # or another summary function
fct_reorder2(f, x, y)             # order by y value at the largest x — good for line plots
fct_infreq(f)                     # order by frequency, most common first
fct_rev(f)                        # reverse current order
fct_relevel(f, "ref", after = 0)  # move a level to the front (e.g. a reference level)
```

Reshaping the levels themselves:

```r
fct_lump_n(f, n = 5)              # keep the 5 most common, lump the rest into "Other"
fct_lump_prop(f, prop = 0.05)     # lump anything below 5% of the data
fct_recode(f, new = "old")        # rename levels
fct_collapse(f, north = c("ME", "VT", "NH"))  # merge several levels into one
fct_expand(f, "unused")           # add a level with no data
fct_drop(f)                       # remove levels with no data
```

Missing values:

```r
fct_na_value_to_level(f, level = "(Missing)")  # make NA an explicit level
fct_na_level_to_value(f, extra_levels = "unknown")  # turn a level back into NA
```

`fct_explicit_na()` is deprecated — use `fct_na_value_to_level()`.

Convert character to factor with `factor()` (alphabetical levels) or `as_factor()`
(levels in order of first appearance). `as_factor()` is usually what you want when the
data already arrives in a meaningful order.

### Two factor traps that produce wrong numbers silently

**`as.numeric()` on a factor returns the level codes, not the values.** This is the
classic R data-loss bug and nothing warns:

```r
f <- factor(c("10", "20", "30"))
as.numeric(f)                  # 1 2 3   <- positions in the level table
as.numeric(as.character(f))    # 10 20 30
```

Always go through `as.character()`. The bug is most common when a numeric column was
read in as a factor or character, "converted", and then averaged — producing a
plausible-looking mean of level indices.

**Filtering rows does not drop the levels.** A factor keeps every level it was created
with, even when no rows use them any more:

```r
d <- tibble(f = factor(c("a", "b", "c")), v = 1:3) %>% filter(f != "c")
levels(d$f)                    # "a" "b" "c" — "c" survives
```

Those phantom levels reappear as empty bars in a plot, empty facets, and zero-count
rows in `count(f, .drop = FALSE)`. Call `fct_drop()` (or `droplevels()`) after
filtering when the removed categories should be gone for good. The reverse is also
worth knowing: `count()` **drops** empty levels by default, so a category that legitimately
has zero rows disappears from a summary table unless you pass `.drop = FALSE`.

## lubridate — dates and times

### Parsing

Function names describe the component order in the *input*, which makes ambiguous
formats unambiguous:

```r
ymd("2024-01-15")
dmy("15/01/2024")
mdy("January 15, 2024")
ymd_hms("2024-01-15 13:45:00", tz = "America/New_York")
```

`today()` and `now()` give the current date and datetime.

**Unparseable input becomes `NA` with only a warning**, so a format the parser does not
recognize turns into missing data rather than an error:

```r
ymd(c("2024-01-01", "not a date"))   # 2024-01-01, NA  + "1 failed to parse"
```

After parsing any date column, check `sum(is.na(parsed))` against what you expected. A
mixed-format column — some `01/02/2024`, some `2024-01-02` — will partially parse and
leave a plausible-looking result that is quietly missing half its rows.

### Extracting components

```r
year(d); month(d); day(d)
month(d, label = TRUE, abbr = FALSE)   # "January" rather than 1
wday(d, label = TRUE, week_start = 1)  # weekday, weeks starting Monday
yday(d); isoweek(d); quarter(d)
```

### Rounding

Rounding is the standard way to aggregate a time series to a coarser grain:

```r
floor_date(d, "month")     # down to the start of the month
ceiling_date(d, "week")    # up to the end of the week
round_date(d, "hour")      # to the nearest hour
```

Combine with grouping to build a monthly summary — see
[grouping.md](grouping.md):

```r
df %>%
  mutate(month = floor_date(date, "month")) %>%
  summarise(total = sum(amount), .by = month)
```

`.by` takes a column selection, not an expression, so the rounded column has to exist
before you group on it.

### Arithmetic

lubridate distinguishes three kinds of time span, and the difference matters across
daylight-saving boundaries and leap years:

- **Periods** (`days()`, `months()`, `years()`) track human clock time — `d + months(1)`
  lands on the same day of the next month.
- **Durations** (`ddays()`, `dyears()`) track exact seconds — unaffected by DST.
- **Intervals** (`interval(start, end)`) are a specific span between two instants.

```r
d + days(30)
interval(start, end) / days(1)      # length in days
time_length(interval(dob, today()), "years")   # age in whole years
```

Use `%within%` to test whether an instant falls inside an interval.
