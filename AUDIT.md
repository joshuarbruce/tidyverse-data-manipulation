# Audit log

How this skill is verified, and what each pass found.

The harness lives outside this repo at `~/Documents/claude_work/tidyverse-skill-audit/`
so it neither ships with the skill nor trips posit-dev's validator, which recognizes
only `scripts/`, `references/` and `assets/`. Run it with:

```bash
~/Documents/claude_work/tidyverse-skill-audit/run-audit.sh
```

## Two classes of defect

**Class A — mechanically detectable.** Automated, run every pass, exits non-zero on any
finding.

| # | Defect | Detector |
|---|---|---|
| 1 | Unparseable code | `parse()` every block |
| 2 | Execution failure | run blocks cumulatively per file |
| 3 | Unexpected warning | capture warnings, not just errors |
| 4 | Wrong asserted output | eval every line carrying a value-claiming comment |
| 5 | Degenerate result | flag 0-row / all-NA / length-0 returns |
| 6 | Dead link | HTTP HEAD every URL; resolve every relative `.md` link |
| 7 | Self-contradiction | grep blocks for patterns the skill itself bans |
| 8 | Orphans / size limits | `skill-validator` + `count-skill-tokens.py` |
| 9 | Unlisted package | cross-ref `pkg::` / `library()` against `sources.md` |

**Class B — prose only.** No script finds these. They require reading each file end to
end, and every one below was found that way *after* Class A was fully green.

| # | Defect | Example found |
|---|---|---|
| 10 | Wrong lifecycle classification | `separate()` called deprecated; it is superseded |
| 11 | Unqualified superlative | "the fastest in-memory tool in R" |
| 12 | Stale cross-reference | "the joins **above**" when joins were below |
| 13 | Adjacent-content contradiction | one table row using `group_by()`, the next using `.by` |
| 14 | Promised but not demonstrated | "non-equi, rolling" showing only non-equi |
| 15 | Unsourced number | "~489× speedup" (the vignette says ~215×) |
| 16 | Reversed semantics | `na_matches` default; `unmatched=` guarding the wrong side |
| 17 | Overstated supersession | `replace_values()` "supersedes" `replace_na()`; it does not |
| 18 | Provenance drift | version list conflating "latest released" with "executed against" |
| 19 | Misleading despite running | an example whose comment claims more than it shows |

## Protocol

Per file, in order — not batched, because batching reproduces the shallow scanning that
let these through in the first place:

1. Read end to end against the Class B checklist.
2. Classify every factual claim as *executable* (run it), *doc-checkable* (`formals()`,
   lifecycle badge, `packageVersion()`), *sourced* (fetch and confirm), or
   *unsupported* — then source it, hedge it, or delete it.
3. Check for contradiction against `SKILL.md` and sibling references.
4. Record here and fix.

A pass is **clean** only when Class A exits 0 *and* every file has been read with zero
Class B findings. Repeat until a pass is clean.

---

## Pass 1 — 2026-08-14

Class A: **0 findings** across all nine checks at the start of the pass (123 blocks,
116 executing, 92 URLs, validator exit 0). One Class A finding surfaced while building
the harness and is recorded below.

| File | Read | Findings |
|---|---|---|
| `data-table.md` | ✓ (prior pass) | 6 — superlative, "every operation" overclaim, stale "joins above", two translation rows contradicting `grouping.md`/`joins.md`, rolling join promised but not shown |
| `sources.md` | ✓ (prior pass) | 3 — `separate()` misclassified as deprecated, version list conflating two meanings, three packages used but unlisted |
| `SKILL.md` | ✓ | 6 — see below |
| `grouping.md` | pending | |
| `joins.md` | pending | |
| `filtering-and-recoding.md` | pending | |
| `factors-and-dates.md` | pending | |
| `performance.md` | pending | 1 (Class A) — duckdb example warned about `na.rm`; SQL aggregates always drop `NULL`, so dbplyr warns rather than silently changing semantics |
| `reshaping.md` | pending | |
| `tidy-eval.md` | pending | |
| `strings.md` | pending | |
| `iteration.md` | pending | |
| `import.md` | pending | |
| `visualization.md` | pending | |

### `SKILL.md` findings

1. **R ≥ 4.1 was never stated.** The skill uses `\(x)` lambdas in six files; that is a
   syntax error on older R. The only R version mentioned anywhere was 4.3.0, in the
   context of the base pipe. Now stated up front alongside the package minimums.
2. **The worked example's NA-safety comment demonstrated nothing.** It used
   `filter_out(is.na(bmi))` under a comment reading "NA-safely" — but `is.na()` can
   never return `NA`, so the NA-handling property never came into play, and
   `filter(!is.na(bmi))` gives the identical 59 rows. Replaced with
   `filter_out(species == "droid")`, which keeps 81 rows where
   `filter(species != "droid")` keeps 77 — a visible four-row gap that is exactly the
   characters whose species is unrecorded.
3. **Unsourced size threshold.** "roughly >1GB, or millions of rows" was a bare
   heuristic, and inconsistent with `performance.md`, where the equivalent "1–10 GB
   range" claim had already been replaced with advice to benchmark. Now says there is
   no useful threshold and to measure.
4. **Silent-failure table was not grouped.** The `unmatched = "error"` row sat at the
   bottom, separated from the two other join rows. Reordered so every reference's rows
   are adjacent.
5. **Style inconsistency.** `rename_with(df, str_to_snake)` written in call form while
   every other example is pipe-first. Restored to `df %>% rename_with(str_to_snake)`.
6. **Stale version.** Frontmatter still read `2.0` after the restructure plus ten
   correction passes. Bumped to `2.1`.
