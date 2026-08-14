# tidyverse-data-manipulation

A Claude Code skill for R data manipulation using the tidyverse ecosystem.

## What it does

When active, Claude will:
- Write idiomatic, modern tidyverse code (dplyr, tidyr, ggplot2, purrr, stringr, forcats, lubridate, tibble, readr)
- Ask your pipe preference (`%>%` vs `|>`) at the start of a session and stay consistent
- Apply current best practices (`.by` grouping, `join_by()`, `list_rbind()`, and the
  dplyr 1.2 `filter_out()` / `recode_values()` families) and steer away from
  deprecated ones (`case_match()`, `coord_flip()`, `..var..`)
- Default to the tidyverse, but switch to **data.table** (or the `dtplyr` bridge) when data is large or speed is the goal — with the reasoning made explicit
- Navigate from a short hub to whichever of the 13 reference files the task actually needs, rather than loading everything every session

## Installation

Copy the `tidyverse-data-manipulation/` folder into your project's `.claude/skills/` directory (create it if it doesn't exist):

```
your-project/
└── .claude/
    └── skills/
        └── tidyverse-data-manipulation/
            ├── SKILL.md
            └── references/
                ├── grouping.md
                ├── joins.md
                ├── filtering-and-recoding.md
                ├── reshaping.md
                ├── import.md
                ├── iteration.md
                ├── strings.md
                ├── factors-and-dates.md
                ├── visualization.md
                ├── tidy-eval.md
                ├── data-table.md
                ├── performance.md
                └── sources.md
```

Or install globally under `~/.claude/skills/` so it's available in every project.

## Pipe preference

At the start of each session Claude will ask whether you prefer `%>%` (magrittr) or
`|>` (native, R ≥ 4.1), then uses that choice consistently for the rest of the
session. With no preference stated it follows the
[tidyverse style guide](https://style.tidyverse.org/pipes.html), which recommends
`|>`. Both are fully supported; the reference files are written in `%>%` and Claude
converts them to your choice.

## Packages covered

| Package | Purpose |
|---|---|
| dplyr | Filter, select, mutate, summarise, join, arrange |
| tidyr | Pivot, nest, separate, complete, fill |
| ggplot2 | Data visualization |
| readr | Fast CSV / delimited file import |
| purrr | Functional iteration (map, walk, reduce) |
| stringr | String manipulation |
| forcats | Factor handling |
| lubridate | Dates and times |
| tibble | Modern data frames |
| data.table | High-performance manipulation for large data (with `dtplyr` bridge) |

## Structure

`SKILL.md` is a navigational hub — a routing table, the core principles, the
tidyverse-vs-data.table decision, and a quick reference of current idioms and the
deprecated forms they replace. Everything else lives in `references/`, loaded on
demand rather than every session:

- `references/grouping.md` — `.by` vs `group_by()`, `across()`, `pick()`, `reframe()`, row-wise work, counting
- `references/joins.md` — join types, `join_by()`, cardinality, pitfalls
- `references/filtering-and-recoding.md` — `filter_out()`, `when_any()`/`when_all()`, the four-function recode/replace family, lookup tables, migrating off `case_match()`
- `references/reshaping.md` — pivoting, nesting, `separate_wider_*()`, missing values
- `references/import.md` — `read_csv()`, `col_types`, parse failures, Excel/SPSS, building tibbles
- `references/iteration.md` — `map_*()`, `walk()`, `list_rbind()`, `in_parallel()`
- `references/strings.md` — `str_*()`, regex, case conversion, cleaning column names
- `references/factors-and-dates.md` — `fct_*()` ordering and lumping, date parsing, rounding, intervals
- `references/visualization.md` — ggplot2 layers, facets, scales, themes, 4.0 deprecations
- `references/tidy-eval.md` — `{{ }}`, `.data[[]]`, `all_of()`, `:=`
- `references/data-table.md` — data.table syntax, dtplyr bridge, tidyverse↔data.table translation table
- `references/performance.md` — profiling, vroom, dtplyr, duckdb, parallelism
- `references/sources.md` — docs site, changelog, and repo for every package

## Keeping it up to date

`references/sources.md` lists the canonical docs, changelog (`/news/index.html`),
and GitHub repo for every package the skill covers, plus the tidyups (design
proposals) behind recent API changes. The
[tidyverse blog](https://www.tidyverse.org/blog/) is the central feed for release
announcements. When a release supersedes or deprecates a pattern, update `SKILL.md`
and bump the "Last verified" dates in `references/sources.md` — that section tracks
*URLs reachable* and *content reconciled* separately, because a link check alone does
not tell you whether the guidance is still current.

## Credits

Built from the official [tidyverse](https://www.tidyverse.org/) documentation and
[R for Data Science (2e)](https://r4ds.hadley.nz/). Skill structure and
modern-pattern emphasis were informed by
[sj-io's R tidyverse skills gist](https://gist.github.com/sj-io/3828d64d0969f2a0f05297e59e6c15ad)
and [ab604/claude-code-r-skills](https://github.com/ab604/claude-code-r-skills).
Full attribution in [`references/sources.md`](references/sources.md).

## License

MIT — see [LICENSE](LICENSE).
