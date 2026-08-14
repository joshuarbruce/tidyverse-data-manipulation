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

## Pass 1 — 2026-08-14 (IN PROGRESS)

**Status: 6 of 14 files read. 8 remaining.** Class A is green; everything below was
found by reading.

### Where to resume

Read the remaining files in this order, applying the four-step protocol above:

1. `factors-and-dates.md` (187 lines)
2. `performance.md` (154)
3. `reshaping.md` (117)
4. `tidy-eval.md` (124)
5. `strings.md` (110)
6. `iteration.md` (105)
7. `import.md` (118)
8. `visualization.md` (112) — partly touched this pass (deprecation table), not fully read

Then re-read `data-table.md` and `sources.md` in the final zero-defect pass.

### Deferred: reference token budget

The references now total **25,063 tokens against the validator's 25,000 advisory**
(hard error is 50,000). The simulated posit layout therefore exits 2, and because
posit's `validate-skills.sh` runs under `pipefail`, that would fail their CI.

Decision taken: **keep prioritizing correctness, trim once at the end** with full
knowledge of what earned its place. Each pass adds roughly 300–400 tokens, so the gap
widens — the final trim is real work, not a rounding adjustment. `data-table.md` (4,911
tokens) is nearly twice the next largest file and is the obvious first place to look;
whether `sources.md` (2,731) should be slimmed was explicitly deferred until the whole
picture is visible.

`run-audit.sh` reports this advisory as a known deferred item rather than failing, so
the run does not sit permanently red. Any *other* validator warning still fails.

### Findings

| File | Read | Findings |
|---|---|---:|
| `data-table.md` | ✓ | 6 |
| `SKILL.md` | ✓ | 6 |
| `filtering-and-recoding.md` | ✓ | 5 |
| `grouping.md` | ✓ | 4 |
| `sources.md` | ✓ | 3 |
| `joins.md` | ✓ | 2 |
| `performance.md` | partial | 1 (Class A) |
| 8 files | pending | — |

**Total so far: 27.**

`data-table.md` — superlative ("the fastest tool in R"); "every operation fits one
bracket" contradicted by the same file; stale "the joins above" when joins are below;
two translation-table rows teaching what the rest of the skill says to avoid; rolling
join promised but not demonstrated; `.I` example using non-existent columns.

`sources.md` — `separate()` given as the example of *deprecated* inside the paragraph
teaching deprecated-vs-superseded (it is Superseded); version list conflating "latest
released" with "executed against"; three packages used but never listed.

`SKILL.md` — R ≥ 4.1 never stated though `\(x)` appears in six files; worked example's
"NA-safely" comment demonstrated nothing (`is.na()` cannot return `NA`); unsourced
">1GB" threshold contradicting `performance.md`; ungrouped silent-failure table; a
call-form example among pipe-first ones; stale frontmatter version.

`filtering-and-recoding.md` — fizzbuzz branch unreachable at `x <- 1:20`;
`replace_when()` wrongly offered as a way to avoid evaluating every RHS (it evaluates
them too); `%in%` wrongly lumped into the NA-bug advice; a stated requirement that does
not exist; an empty section heading; a broken sentence.

`grouping.md` — advice to split `mutate()` calls on sequential dependency (wrong: a
later column may reference an earlier one, and it respects `.by`); `arrange()` described
as both inheriting and ignoring grouping without reconciling the two senses; "unlike
every other verb" overclaim; `reframe()` example returning unlabelled quantiles.

`joins.md` — `by = c(...)` called superseded (dplyr's docs list it as accepted);
"join keys must be the same type" false (factor/character coerce silently, discarding
level order).

`performance.md` (Class A) — duckdb example warned about `na.rm`; SQL aggregates always
drop `NULL`, so dbplyr warns rather than silently changing semantics.

**Cross-file:** the deprecation tables in `SKILL.md` and `visualization.md` conflated
deprecated with superseded. Verifying every row against its help page caught
`geom_errorbarh()` — Deprecated, not Superseded, where it had been grouped with
`coord_flip()`.

### Harness changes forced by real failures

- **Allowlist matched by index.** Inserting one block into `grouping.md` shifted every
  later index and turned an intentional demo into a false failure. Now matched by
  content substring, and a stale entry (matching zero or several blocks) is itself
  reported — an allowlist that quietly stops matching is a hole in the audit.
- **The venv copied into the audit directory was broken** (absolute paths from its
  original location). Rebuild with `python3 -m venv venv && ./venv/bin/pip install
  tiktoken frontmatter` if `count-skill-tokens.py` fails to launch.

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
