# Quant_Analysis — Plan

## Where the repo stands (read 2026-10-01)

- 3 commits, last one `80f10d5 Rename R to time_series` dated **2017-12-05**. Branch `claude/magical-lovelace-1u1xu3` is identical to `master`. No open or closed PRs, no issues.
- `README.md` — 0 lines of content (just the `# Quant_Analysis` heading).
- `time_series` — 1206 lines, **no file extension**, plain R. It is a concatenation of DataCamp exercise answers from two courses:
  - lines 1–988: *Introduction to R for Finance* (vectors, matrices, data frames, factors, lists, dates, logic, loops, `apply` family, a `tidyquant` pull).
  - lines 989–1206: *Manipulating Time Series Data in R with xts and zoo* (subsetting, merge/join, `na.locf`, `lag`/`diff`, `period.apply`, `to.period`, `rollapply`, index attributes, timezones).
- Nothing else: no `.gitignore`, `.Rproj`, `DESCRIPTION`, `renv.lock`, tests, or CI.

### Concrete defects found

| Line | Problem |
|---|---|
| 470 | `dates <- ` — empty right-hand side. **Parse error; the whole file fails to source.** |
| 316 | `plot(drop_level)` but the comment says plot `keep_level`, and `drop_level` is not defined until line 321. |
| 155, 162, 173 | `cbind(apple, ibm, micr)`, `apple_micr_matrix`, `cor(apple, ibm)` — these objects do not exist yet; `apple` is first defined at line ~848. |
| 292 | `cut(x = AAA_rank, ...)` — `AAA_rank` never defined. |
| 1144, 1147, 1150, 1162 | `indexClass()`, `indexTZ()`, `indexFormat()` — removed/deprecated in xts ≥ 0.12. Use `tclass()`, `tzone()`, `tformat()`. |
| 891 | `tq_mutate(ohlc_fun = Ad, ...)` — `ohlc_fun` was dropped from tidyquant; now `select = adjusted`. |
| 25, 47, 848 | `apple_stock`/`apple` and `ret` are redefined with different shapes (scalar then vector), so later chunks silently depend on which chunk ran last. |
| throughout | ~25 variables the DataCamp sandbox injected are never created here: `ibm`, `micr`, `panera`, `stocks`, `stock`, `corr`, `stock_return`, `x`, `dates`, `irreg`, `temps`, `temps_june30`, `temps_july17/18`, `edhec`, `eq_mkt`, `usd_eur`, `AirPass`, `a`, `b`, `z`, `times`. |

So today the repo is a write-only notes dump: not runnable, not navigable, not reusable.

## Fork in the road

**Track A — archive it properly.** Treat the notes as a learning artifact: make them runnable, organised, and documented. ~Phases 1–2.
**Track B — turn it into a real quant project.** Keep the notes as an appendix and build a small, tested analysis toolkit plus reproducible notebooks on top. Phases 1–5.

Recommendation: **do Track A first (Phases 1–2), then Track B.** Phase 1–2 is cheap, makes everything after it verifiable, and is worth doing regardless of whether Track B happens.

## Phase 1 — Hygiene and structure

1. `git mv time_series notes/01-intro-r-for-finance.R` and split lines 989+ into `notes/02-xts-zoo-time-series.R` (keeps history, gives the files an extension so editors, linters and `source()` work).
2. `.gitignore` for R: `.Rhistory`, `.RData`, `.Rproj.user/`, `renv/library/`, `*.html` build output, `data/cache/`.
3. `README.md` with real content: what the repo is, directory layout, how to install dependencies, how to run the notes and the analyses.
4. `Quant_Analysis.Rproj` so the repo opens as a project with the correct working directory.
5. `LICENSE` — MIT unless you want something else.

**Done when:** `ls` tells a stranger what this repo is, and `README.md` answers "how do I run this".

## Phase 2 — Make the notes actually run

1. Fix the parse error at line 470 (`dates <- as.Date(char_dates, format = "%d%b%y")`) and the `plot(keep_level)` mix-up at line 316.
2. Add `notes/setup-data.R`, sourced at the top of each notes file, that creates every injected variable reproducibly:
   - price vectors and the `stocks` data frame from a small committed CSV under `data/` (not a live download), so the notes run offline;
   - `AAA_rank`, `corr`, `stock_return` from `set.seed()`-fixed draws;
   - the xts objects from packaged sample data — `edhec` from **PerformanceAnalytics**, `AirPass` from `datasets::AirPassengers`, `temps`/`sample_matrix` from **xts** — with one documented substitution note where DataCamp's exact dataset isn't public.
3. Resolve the shadowing: rename the scalar-vs-vector collisions (`apple_stock` vs `apple_prices`, `ret_12m` vs `ret_portfolio`) so each chunk is independent of execution order.
4. Modernise the deprecated calls listed in the table, leaving a one-line comment naming the old form so the notes still match the course material.
5. Put the network-dependent `tq_get()` block behind `if (interactive())` plus a cached `.rds` under `data/cache/`, so a fresh clone runs with no API call.
6. Verify: `Rscript -e 'source("notes/01-intro-r-for-finance.R", echo = FALSE)'` exits 0, same for file 02.

**Blocker to clear first:** R is **not installed** in this container (`which R` → nothing), so none of this can be validated here yet. I'd install R plus `xts`, `zoo`, `PerformanceAnalytics`, `tidyquant`, `testthat` and `lintr` for the session, and the same commands belong in the environment's setup script so every future session has them.

**Done when:** both notes files source cleanly from a fresh clone with no network access.

## Phase 3 — A small tested toolkit

Promote the useful functions out of the notes into `R/`:

- `R/time-value.R` — `percent_to_decimal()`, `pv()`, `npv()`, `irr()`.
- `R/risk.R` — `sharpe()`, `annualize_return()`, `annualize_vol()`, `max_drawdown()`.
- `R/portfolio.R` — `portfolio_return()`, `portfolio_vol()` (needs the covariance matrix, which the notes never do), `rebalance()`.

Each with roxygen docs, input validation (weights sum to 1, non-negative periods), and `testthat` tests in `tests/testthat/`: `pv(1200, 7, 3)` against the hand-computed value, `sharpe()` against a known mean/sd, `portfolio_return()` against the Microsoft/Sony example already in the notes, `max_drawdown()` on a crafted series with a known trough.

**Done when:** `Rscript -e 'testthat::test_dir("tests/testthat")'` is green.

## Phase 4 — Reproducibility and CI

1. `renv.lock` (or `install.R` if renv is overkill) pinning the dependency set.
2. `.github/workflows/R-CMD-check.yml` using `r-lib/actions`: install deps from the lockfile, run `testthat`, run `lintr`, and source both notes files as a smoke test.
3. `.lintr` config (line length, object naming) so lint failures are about real issues.
4. A `SessionStart` hook so cloud sessions install R and the packages automatically.

**Done when:** a push to this branch produces a green CI run.

## Phase 5 — The actual quant analysis

This is the part the repo's name promises and currently has none of.

1. `analysis/01-returns-and-risk.Rmd` — pull a fixed ticker set and date range, cache to `data/`, then: adjusted-price series, daily and cumulative returns, rolling 21-day volatility, drawdown curve, return distribution vs normal, correlation matrix.
2. `analysis/02-portfolio-construction.Rmd` — equal-weight vs market-cap-ish vs minimum-variance weights on the same universe; portfolio return, volatility, Sharpe; quarterly rebalancing vs buy-and-hold; a two-asset efficient-frontier plot.
3. Each notebook ends with a short written conclusion, not just plots — the point is the finding, not the chart.
4. Knit to `docs/` and enable GitHub Pages so the output is readable without running R.

**Done when:** both notebooks knit from a clean clone against cached data and the rendered output is published.

## Sequencing

Phase 1 → 2 in one commit series (they are not independently useful). Phase 3 and 4 can go in parallel once 2 lands. Phase 5 depends on 3.

## Open questions

1. Track A only, or Track A then Track B?
2. Phase 5 universe and date range — which tickers, and over what window?
3. Data source for Phase 5: `tidyquant`/Yahoo (free, occasionally flaky) or something you already have credentials for? Note this container's outbound traffic goes through a proxy, so a live pull needs testing before the notebooks depend on it.
4. MIT licence, or leave it unlicensed?
