# NWSL-kickoffs

## Requirements

- [Quarto](https://quarto.org/docs/get-started/) — to build and preview the site
- Python 3.10+ — to run `update_schedule.py`
- Python dependency: `pip install requests`

## Usage

Update schedule data:

```bash
python3 -m venv .venv
source .venv/bin/activate 
python3 update_schedule.py
```

Preview the site:
```bash
quarto preview
```

## Basic development flow

- Files to edit
  - `update_schedule.py`
  - `styles.css`
  - `about.qmd`
- Run `python3 update_schedule.py` to generate `index.qmd` and `schedule.md`
  about.qmd

## Troubleshooting

ModuleNotFoundError: No module named 'requests' 

```bash
python3 -m venv .venv
source .venv/bin/activate 
pip install requests ## one time
```
