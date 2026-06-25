# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Quarto static website showing NWSL match kickoff times and streaming info, published to GitHub Pages. Content for the two main pages is **generated**, not hand-written: `update_schedule.py` fetches data from the ESPN scoreboard API and rewrites `index.qmd` and `schedule.qmd`.

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

**Data flow:** ESPN scoreboard API → `update_schedule.py` parses events → rewrites `.qmd` files → `quarto render` produces `_site/` → GitHub Pages.

**`update_schedule.py`** is the core. It produces two outputs from the same parsed game data (`parse_game` → dicts of `home`, `away`, `et_dt`, `networks`):

- `index.qmd` — current week only (`DAYS_AHEAD` window). Built by `build_index_content` and spliced into the existing file by `update_index_qmd`, which **preserves the front matter and everything from the `## How to watch` heading down** (the hand-written FAQ/credits) and replaces only the games block in between. If you edit `index.qmd` by hand, only touch the front matter and the `## How to watch` section onward — anything above it is overwritten on every run.
- `schedule.qmd` — full season (`SEASON_START`..`SEASON_END`). Built and **fully overwritten** by `write_schedule_qmd`. Never hand-edit this file; it has no preserved regions.

Both pages embed HTML tables inside Quarto `` ```{=html} `` raw blocks. Markdown `##`/`###` month/day headings are intentionally kept as real markdown so Quarto picks them up for the TOC.

**Times are rendered client-side.** Python emits each kickoff as a `<td class="time-cell">` carrying UTC timestamps in `data-announced` / `data-approx` attributes; `scripts.html` (loaded site-wide via `include-after-body` in `_quarto.yml`) converts them to the selected timezone in the browser. The timezone toggle and the team filter on `schedule.qmd` are both pure client-side JS in `scripts.html`. The `?timezone=` query param persists the user's choice.

**Kickoff time model:** ESPN gives an "announced" time. Some streamers consistently start late, so `NETWORK_BUFFERS` adds per-network minutes to produce an "approx" time; both are shown when a buffer applies. `STREAM_LINKS` maps network short names to linked streaming destinations, and `SHORT_NAMES` maps full team names (including historical aliases like `OL Reign`→`Seattle`) to display names. These three dicts are the main things to update when networks/teams change.

## Editing surfaces

| File | Hand-edited? |
|------|--------------|
| `update_schedule.py` | Yes — logic + the config dicts at top |
| `index.qmd` | Front matter + `## How to watch` section onward only |
| `schedule.qmd` | No — fully generated |
| `styles.css`, `head.html`, `scripts.html` | Yes |
| `_quarto.yml` | Yes — site config |
| `_site/` | No — build output |

## CI / publishing

`.github/workflows/update-schedule.yml` runs weekly (Mon 13:00 UTC) and on manual dispatch: it runs `uv run --frozen update_schedule.py`, commits the updated `index.qmd`/`schedule.qmd` to `main` (with `[skip ci]`), then renders and publishes to the `gh-pages` branch via the Quarto publish action. The uv version and Python 3.13 are pinned in the `setup-uv` step to match the lockfile.
