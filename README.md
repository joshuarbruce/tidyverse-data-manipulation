# tidyverse-data-manipulation

A Claude Code skill for R data manipulation using the tidyverse ecosystem.

## What it does

When active, Claude will:
- Write idiomatic, modern tidyverse code (dplyr, tidyr, ggplot2, purrr, stringr, forcats, lubridate, tibble, readr)
- Ask your pipe preference (`%>%` vs `|>`) at the start of a session and stay consistent
- Apply current best practices (`.by` grouping, `join_by()`, `list_rbind()`, etc.)
- Reach for reference files on joins, tidy evaluation, and performance when needed

## Installation

Copy the `tidyverse-data-manipulation/` folder into your project's `.claude/skills/` directory (create it if it doesn't exist):

```
your-project/
└── .claude/
    └── skills/
        └── tidyverse-data-manipulation/
            ├── SKILL.md
            └── references/
                ├── joins.md
                ├── tidy-eval.md
                └── performance.md
```

Or install globally under `~/.claude/skills/` so it's available in every project.

## Pipe preference

At the start of each session Claude will ask whether you prefer `%>%` (magrittr,
works in all R versions) or `|>` (native, requires R ≥ 4.1). It then uses your
choice consistently for the rest of the session.

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

## Reference files

Loaded on demand (not every session):

- `references/joins.md` — join types, `join_by()`, cardinality, pitfalls
- `references/tidy-eval.md` — `{{ }}`, `.data[[]]`, `all_of()`, `:=`
- `references/performance.md` — profiling, vroom, dtplyr, duckdb, furrr
- `references/sources.md` — docs site, changelog, and repo for every package

## Keeping it up to date

`references/sources.md` lists the canonical docs, changelog (`/news/index.html`),
and GitHub repo for every package the skill covers. The
[tidyverse blog](https://www.tidyverse.org/blog/) is the central feed for release
announcements. When a release supersedes or deprecates a pattern, update `SKILL.md`
and bump the "Last verified" date in `references/sources.md`.

## Credits

Built from the official [tidyverse](https://www.tidyverse.org/) documentation and
[R for Data Science (2e)](https://r4ds.hadley.nz/). Skill structure and
modern-pattern emphasis were informed by
[sj-io's R tidyverse skills gist](https://gist.github.com/sj-io/3828d64d0969f2a0f05297e59e6c15ad)
and [ab604/claude-code-r-skills](https://github.com/ab604/claude-code-r-skills).
Full attribution in [`references/sources.md`](references/sources.md).

## License

MIT — see [LICENSE](LICENSE).
