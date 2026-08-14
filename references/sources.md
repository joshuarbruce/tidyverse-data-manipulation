# Sources & Maintenance Reference

This file is the maintenance backbone for the skill: it lists the canonical
documentation site, changelog, and source repository for every package the skill
covers. When you want to verify that a pattern in `SKILL.md` is still current — or
update the skill after a new release — start here.

## How to check for updates

1. **Central feed** — the [tidyverse blog](https://www.tidyverse.org/blog/)
   announces every major release across all core packages. Skim it first; it's the
   fastest way to spot breaking changes, superseded functions, and new idioms.
2. **Per-package changelog** — every tidyverse/r-lib package has a pkgdown
   changelog at `<docs-site>/news/index.html` (e.g.
   <https://dplyr.tidyverse.org/news/index.html>). This is the authoritative list
   of what changed in each version, including deprecations.
3. **Source repo** — the GitHub repo's `NEWS.md` and Releases tab carry the same
   information plus open issues/PRs if you need to dig deeper.

When updating the skill, watch especially for **superseded** functions (still work
but discouraged — e.g. `map_dfr()`) and **deprecated** functions (warn now, removed
later — e.g. `separate()`). These are the patterns most likely to make the skill
feel dated.

## Core tidyverse packages

| Package | Docs (changelog at `/news/index.html`) | Source repo |
|---|---|---|
| ggplot2 | https://ggplot2.tidyverse.org/ | https://github.com/tidyverse/ggplot2 |
| dplyr | https://dplyr.tidyverse.org/ | https://github.com/tidyverse/dplyr |
| tidyr | https://tidyr.tidyverse.org/ | https://github.com/tidyverse/tidyr |
| readr | https://readr.tidyverse.org/ | https://github.com/tidyverse/readr |
| purrr | https://purrr.tidyverse.org/ | https://github.com/tidyverse/purrr |
| tibble | https://tibble.tidyverse.org/ | https://github.com/tidyverse/tibble |
| stringr | https://stringr.tidyverse.org/ | https://github.com/tidyverse/stringr |
| forcats | https://forcats.tidyverse.org/ | https://github.com/tidyverse/forcats |
| lubridate | https://lubridate.tidyverse.org/ | https://github.com/tidyverse/lubridate |

## Pipes & programming

| Package | Docs | Source repo |
|---|---|---|
| magrittr (`%>%`) | https://magrittr.tidyverse.org/ | https://github.com/tidyverse/magrittr |
| native pipe (`\|>`) | https://www.tidyverse.org/blog/2023/04/base-vs-magrittr-pipe/ | base R (R ≥ 4.1) |
| glue | https://glue.tidyverse.org/ | https://github.com/tidyverse/glue |
| rlang (tidy eval) | https://rlang.r-lib.org/ | https://github.com/r-lib/rlang |

## Data import

| Package | Docs | Source repo |
|---|---|---|
| readxl | https://readxl.tidyverse.org/ | https://github.com/tidyverse/readxl |
| haven (SPSS/Stata/SAS) | https://haven.tidyverse.org/ | https://github.com/tidyverse/haven |
| googlesheets4 | https://googlesheets4.tidyverse.org/ | https://github.com/tidyverse/googlesheets4 |
| googledrive | https://googledrive.tidyverse.org/ | https://github.com/tidyverse/googledrive |
| rvest (web scraping) | https://rvest.tidyverse.org/ | https://github.com/tidyverse/rvest |
| httr2 (HTTP) | https://httr2.r-lib.org/ | https://github.com/r-lib/httr2 |
| jsonlite | https://jeroen.r-universe.dev/jsonlite | https://github.com/jeroen/jsonlite |
| xml2 | https://xml2.r-lib.org/ | https://github.com/r-lib/xml2 |
| DBI (databases) | https://dbi.r-dbi.org/ | https://github.com/r-dbi/DBI |

## Wrangling backends

| Package | Docs | Source repo |
|---|---|---|
| dbplyr (SQL) | https://dbplyr.tidyverse.org/ | https://github.com/tidyverse/dbplyr |
| dtplyr (dplyr → data.table) | https://dtplyr.tidyverse.org/ | https://github.com/tidyverse/dtplyr |
| hms (time-of-day) | https://hms.tidyverse.org/ | https://github.com/tidyverse/hms |

## High-performance alternative (see references/data-table.md)

| Package | Docs | Source repo |
|---|---|---|
| data.table | https://r-datatable.com/ | https://github.com/Rdatatable/data.table |

## Performance (see references/performance.md)

| Package | Docs | Source repo |
|---|---|---|
| vroom | https://vroom.r-lib.org/ | https://github.com/tidyverse/vroom |
| duckdb (R client) | https://r.duckdb.org/ | https://github.com/duckdb/duckdb-r |
| furrr (parallel purrr) | https://furrr.futureverse.org/ | https://github.com/futureverse/furrr |
| bench (benchmarking) | https://bench.r-lib.org/ | https://github.com/r-lib/bench |
| profvis (profiling) | https://profvis.r-lib.org/ | https://github.com/r-lib/profvis |
| mirai (backs `purrr::in_parallel()`) | https://mirai.r-lib.org/ | https://github.com/r-lib/mirai |

## Design proposals (tidyups)

Tidyups are the design documents behind cross-package API changes. They explain *why*
an interface changed, which is often more useful than the changelog entry:

| Tidyup | Covers | URL |
|---|---|---|
| Tidyup 7 | Recoding and replacing values (`recode_values()`, `replace_values()`, `replace_when()`) | https://github.com/tidyverse/tidyups/blob/main/007-tidyverse-recoding-and-replacing.md |
| Tidyup 8 | Expanding the `filter()` family (`filter_out()`, `when_any()`, `when_all()`) | https://github.com/tidyverse/tidyups/pull/30 |
| All tidyups | Index of accepted and proposed design changes | https://github.com/tidyverse/tidyups |

The dplyr vignette `vignette("recoding-replacing")` is the practical companion to
Tidyup 7 and the best single explanation of which of the four functions to reach for.

## Meta & learning

| Resource | URL |
|---|---|
| tidyverse home | https://www.tidyverse.org/ |
| tidyverse blog (release announcements) | https://www.tidyverse.org/blog/ |
| tidyverse meta-package repo | https://github.com/tidyverse/tidyverse |
| R for Data Science (2e), Wickham/Çetinkaya-Rundel/Grolemund | https://r4ds.hadley.nz/ |
| tidyverse style guide | https://style.tidyverse.org/ |
| tidyverse design principles | https://design.tidyverse.org/ |

## Acknowledgments & inspiration

This skill was built from the official tidyverse documentation and R4DS (all linked
above). Its structure and emphasis on *modern* tidyverse idioms (native pipe, `.by`
grouping, `join_by()`, `map() %>% list_rbind()`) and its progressive-disclosure
reference-file layout were directly informed by two community resources:

| Resource | What it contributed | URL |
|---|---|---|
| sj-io — "Claude R Tidyverse Expert Skills" (gist) | Modern-pattern guidance and base-R → tidyverse migration paths | https://gist.github.com/sj-io/3828d64d0969f2a0f05297e59e6c15ad |
| ab604 — claude-code-r-skills | Skill architecture, token-optimization approach, and reference-file organization | https://github.com/ab604/claude-code-r-skills |

The R coding patterns themselves are tidyverse canon, documented in the official
package sites and R4DS — these resources shaped *how the skill is organized*, not
proprietary content.

## Last verified

Two different things can be "verified" here, and conflating them is how a skill goes
stale while looking current. Record both:

- **URLs reachable** — every link above resolves: **2026-08-14**
- **Content reconciled** — `SKILL.md` guidance checked against each package's current
  changelog and corrected: **2026-08-14**

Versions current at the last content reconciliation: dplyr 1.2.1, tidyr 1.3.2,
ggplot2 4.0.3, purrr 1.2.2, stringr 1.6.0, readr 2.2.0, forcats 1.0.1, tibble 3.3.1,
lubridate 1.9.5, data.table 1.18.4, dtplyr 1.3.2.

If a docs link 404s, the package may have been renamed, retired, or moved orgs —
check the [tidyverse blog](https://www.tidyverse.org/blog/) or search the
[r-lib](https://github.com/r-lib) and [tidyverse](https://github.com/tidyverse)
GitHub organizations.
