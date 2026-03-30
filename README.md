# WRU — Wide Receiver Grading Model

A play-level NFL wide receiver grading engine built in Python, designed to replicate and improve upon PFF's grading methodology using publicly available data.

---

## Overview

WRU evaluates wide receiver performance by combining multiple advanced metrics into a single composite grade, scaled to a **40–99 range** (consistent with PFF's published scale). Grades are computed at the play level and aggregated per player per season, with sample-size stabilization via √targets weighting.

The model is validated against PFF's published grades, with particular attention to contested catch specialists, route runners, and YAC-dependent receivers.

---

## Metric Weights

| Metric | Weight | Source |
|---|---|---|
| YPRR (Yards Per Route Run) | 25% | nflreadpy / NGS |
| Separation | 22% | NFL Next Gen Stats |
| YAC (Yards After Catch) | 18% | nflfastR play-by-play |
| Catch Point | 15% | nflfastR (air yards, CPOE) |
| Down & Distance | 10% | nflfastR play-by-play |
| Explosive Plays | 7% | nflfastR play-by-play |
| Snap Adjustment | 3% | nflreadpy snap counts |

---

## Data Sources

- **[nflreadpy](https://github.com/nflverse/nflreadpy)** — NGS separation, snap counts, player stats (2016–2025)
- **nflfastR** play-by-play — air yards, CPOE, raw YAC, down & distance context
- **NFL Next Gen Stats** — separation at catch, YAC above expectation

---

## Grading Pipeline

The pipeline is structured as a Jupyter notebook (`wr_data_pipeline.ipynb`) with the following cells:

| Cell | Purpose |
|---|---|
| Cell 1 | Config, imports, weight definitions |
| Cell 2 | Load & merge play-by-play data |
| Cell 3 | QB quality adjustment (`adj_cpoe = cpoe - qb_avg_cpoe`) |
| Cell 4 | Composite play grade computation |
| Cell 5 | Player-level aggregation & grade scaling |
| Cell 6 | Output & visualization |

> **Note:** Cell 3 (QB adjustment) is a hard dependency for Cell 4. Always run in order.

---

## Composite Grade Formula

```
play_grade = (
    sep_grade   * W_SEPARATION  +
    cp_grade    * W_CATCH_POINT +
    yac_grade   * W_YAC
) * dd_multiplier + exp_bonus
```

Final scores are rescaled to a **40–99 range**.  
Grade distribution targets: mean ≈ 0.215, min ≈ –1.260.

---

## Requirements

```
python >= 3.13
pandas
numpy
matplotlib
seaborn
nflreadpy
```

Install dependencies:

```bash
pip install pandas numpy matplotlib seaborn nflreadpy
```

---

## Usage

1. Clone the repo:
```bash
git clone https://github.com/YOUR_USERNAME/WRU.git
cd WRU
```

2. Launch the notebook:
```bash
jupyter notebook wr_data_pipeline.ipynb
```

3. Run cells in order (Cell 1 → Cell 6). Cell 3 must run before Cell 4.

---

## Configuration

Key parameters are defined in **Cell 1**:

```python
SEASONS     = list(range(2016, 2026))   # Season range
MIN_TARGETS = 50                        # Minimum target threshold
OUTPUT_DIR  = './'                      # Output directory
```

Weight variables (`W_YPRR`, `W_SEPARATION`, `W_YAC`, etc.) are also defined in Cell 1 and referenced throughout Cell 4.

---

## Validation

Model grades are benchmarked against **PFF's published WR grades** as the primary external reference. Known edge cases under active review:

- Contested catch specialists (e.g., Terry McLaurin, Drake London)
- Route running efficiency vs. target volume tradeoffs
- Small sample size stabilization (minimum 50 targets enforced)

---

## Design Decisions

- **Tier multipliers removed** — tiers (100+, 70–99, 50–69 targets) are display labels only; no grade inflation by volume
- **√targets weighting** — stabilizes grades for receivers with smaller sample sizes
- **QB quality adjustment** — applied before play grading to isolate receiver contribution from quarterback accuracy
- **Play-level grading** — avoids aggregation bias; grades roll up from individual plays

---

## Status

🚧 Active development — currently integrating YPRR and snap adjustment metrics, resolving `nflreadpy` API compatibility for `load_player_stats()` and `load_snap_counts()`.
