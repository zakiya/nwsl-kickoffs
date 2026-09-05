# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Quarto static website showing match kickoff times and streaming info for the NWSL, the Women's Africa Cup of Nations (WAFCON), and Arsenal's and Chelsea's women's fixtures (WSL, UEFA Women's Champions League, English FA Cup and League Cup), published to GitHub Pages. Content for the two main pages is **generated**, not hand-written: `update_schedule.py` fetches data from the ESPN scoreboard API and rewrites `index.qmd` and `schedule.qmd`.

## Commands

This project uses **uv** to run Python (manages Python + dependencies from `pyproject.toml` / `uv.lock` automatically — no `pip` or manual venv).

- Regenerate page content from the ESPN API: `uv run update_schedule.py`
- Build the site: `quarto render`
- Local preview with live reload: `quarto preview`
- Full local flow: `uv run update_schedule.py && quarto render`
- Run against the committed lockfile (what CI does): `uv run --frozen update_schedule.py`
- Update dependencies to latest: `uv lock --upgrade`

There are no tests or linters configured.

## Architecture

**Data flow:** ESPN scoreboard API → `update_schedule.py` parses events → rewrites `.qmd` files → `quarto render` produces `_site/` → GitHub Pages. Fixtures come from several ESPN feeds defined in `ESPN_SOURCES`, each entry a `(competition label, feed URL, team filter)` triple. A filter of `None` takes every fixture in the feed (NWSL `usa.nwsl`, WAFCON `caf.w.nations`); a set of team names keeps only fixtures involving one of those teams, which is how the English/European feeds (`eng.w.1`, `uefa.wchampions`, `eng.w.fa`, `eng.w.league_cup`) are narrowed to `TRACKED_CLUBS` — Arsenal and Chelsea — rather than pulling in whole leagues. Filtering happens in `_get_events`, before anything downstream sees the event. Feeds are merged and de-duplicated by event id.

**Both pages look forward only.** `index.qmd` covers the next `DAYS_AHEAD` days and `schedule.qmd` a rolling `MONTHS_AHEAD` window from today; there are no season-start/end constants and finished games are never rendered. ESPN answers a dated query spanning much more than a year with an HTTP 400, so `fetch_games` splits the window into `MAX_RANGE_DAYS` chunks (`date_chunks`) and merges the results — don't collapse that back into a single request.

**`update_schedule.py`** is the core. It produces two outputs from the same parsed game data (`parse_game` → dicts of `home`, `away`, `et_dt`, `networks`, `competition`):

- `index.qmd` — current week only (`DAYS_AHEAD` window). Built by `build_index_content` and spliced into the existing file by `update_index_qmd`, which **preserves the front matter and everything from the `## Credits` heading down** (the hand-written credits) and replaces only the games block in between. If you edit `index.qmd` by hand, only touch the front matter and the `## Credits` section onward — anything above it is overwritten on every run. The streaming-service guide lives on its own hand-written page, `watch.qmd`.
- `schedule.qmd` — the rolling `MONTHS_AHEAD` window. Built and **fully overwritten** by `write_schedule_qmd`. Never hand-edit this file; it has no preserved regions. Because the window always spans two calendar years, month headings carry the year (`## March 2027`) and the mobile jump-nav anchors are slugified by `month_anchor`.

Both pages embed HTML tables inside Quarto `` ```{=html} `` raw blocks. Markdown `##`/`###` month/day headings are intentionally kept as real markdown so Quarto picks them up for the TOC.

**Times are rendered client-side.** Python emits each kickoff as a `<td class="time-cell">` carrying UTC timestamps in `data-announced` / `data-approx` attributes; `scripts.html` (loaded site-wide via `include-after-body` in `_quarto.yml`) converts them to the selected timezone in the browser. The timezone toggle and the team filter on `schedule.qmd` are both pure client-side JS in `scripts.html`. The `?timezone=` query param persists the user's choice.

Each row carries a **competition badge** (a `<td class=competition>` with a Bootstrap `badge` span). `comp_badge_html` looks the competition label up in the `COMPETITIONS` dict for its display text + badge color class (Lux theme provides the `bg-*` colors); unknown labels fall back to gray `bg-secondary`.

**Kickoff time model:** ESPN gives an "announced" time. Some streamers consistently start late, so `NETWORK_BUFFERS` adds per-network minutes to produce an "approx" time; both are shown when a buffer applies. `STREAM_LINKS` maps network short names to linked streaming destinations, and `SHORT_NAMES` maps full team names (including historical aliases like `OL Reign`→`Seattle`) to display names — national teams pass through unchanged. `ESPN_SOURCES`, `TRACKED_CLUBS`, `COMPETITIONS`, `STREAM_LINKS`, `NETWORK_BUFFERS`, and `SHORT_NAMES` are the main config to update when competitions/networks/teams change.

## Editing surfaces

| File | Hand-edited? |
|------|--------------|
| `update_schedule.py` | Yes — logic + the config dicts at top |
| `index.qmd` | Front matter + `## Credits` section onward only |
| `watch.qmd` | Yes — hand-written streaming guide |
| `schedule.qmd` | No — fully generated |
| `styles.css`, `head.html`, `scripts.html` | Yes |
| `_quarto.yml` | Yes — site config |
| `_site/` | No — build output |

## CI / publishing

`.github/workflows/update-schedule.yml` runs daily (13:00 UTC) and on manual dispatch: it runs `uv run --frozen update_schedule.py`, commits the updated `index.qmd`/`schedule.qmd` to `main` (with `[skip ci]`), then renders and publishes to the `gh-pages` branch via the Quarto publish action. The uv version and Python 3.13 are pinned in the `setup-uv` step to match the lockfile.
