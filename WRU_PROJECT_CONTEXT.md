# WRU — Wide Receiver Grading Model
## Project Context Document
> Read this file at the start of every Claude Code session to understand the full context of the WRU project.

---

## Project Overview

**Goal:** Build a play-level WR grading model in Python that replicates and improves upon PFF's grading methodology. The model evaluates WRs across weighted composite metrics, rescaled to a 40–99 range.

**Environment:**
- Python 3.13, Windows, Git Bash
- Jupyter Notebook (`wr_data_pipeline.ipynb`) located at `~/Desktop/WRU/`
- GitHub repo: `https://github.com/BOOYABOY07/wr-grading-model`

---

## Data Sources

| Source | Package | What It Provides |
|---|---|---|
| NFL Next Gen Stats (NGS) | `nflreadpy` | Separation, YAC above expectation, cushion |
| nflfastR Play-by-Play | `nflreadpy` | Catch point, raw YAC, CPOE, EPA |
| Seasonal Player Stats | `nflreadpy` | YPRR proxy (wopr, racr, target share) |
| Snap Counts | `nflreadpy` | Snap adjustment / injury context |

**Important:** We migrated from `nfl_data_py` to `nflreadpy` because `nfl_data_py` was archived in September 2025 and no longer supports 2025 season data. All data pulls must use `nflreadpy`.

**Seasons covered:** 2016–2025 (NGS only goes back to 2016)

---

## Current Metric Weights

| Rank | Metric | Weight | Source |
|---|---|---|---|
| 1 | YPRR (route running proxy) | 25% | `nflreadpy` seasonal stats |
| 2 | Separation | 22% | NGS `avg_separation` |
| 3 | YAC | 18% | NGS `avg_yac_above_expectation` |
| 4 | Catch Point | 15% | nflfastR `adj_cpoe` (QB-adjusted) |
| 5 | Down & Distance Context | 10% | nflfastR play-by-play |
| 6 | Downfield Explosive Plays | 7% | nflfastR play-by-play |
| 7 | Snap Adjustment | 3% | `nflreadpy` snap counts |

---

## Key Architectural Decisions

### 1. Play-Level Grading (Not Seasonal Averages)
We moved from seasonal average normalization to **play-level grading** — every targeted pass is graded individually, then grades accumulate to a seasonal score. This mirrors PFF's methodology and eliminates small-sample inflation.

### 2. QB Quality Adjustment (adj_cpoe)
Raw `cpoe` contaminates WR grades with QB quality. We calculate each QB's seasonal avg_cpoe, then subtract it from each play:
```python
adj_cpoe = play_cpoe - qb_avg_cpoe
```
This ensures WRs aren't penalized for bad QBs (e.g. Justin Jefferson with JJ McCarthy) or inflated by elite QBs.

**Must be computed BEFORE Cell 4 (play grading).** This is a critical cell-order dependency.

### 3. Separation Proxy (CP-based)
We don't have per-play separation distance data. We use `cp` (completion probability) as a separation proxy:
- `cp >= 0.75` → receiver was wide open
- `cp 0.50–0.75` → moderately open
- `cp 0.30–0.50` → tight coverage
- `cp < 0.30` → heavily contested

**Key rule:** When `cp >= 0.50`, the separation grade is **positive regardless of whether the pass was caught** — because the receiver did their job getting open. Only in tight coverage does the outcome affect the separation grade.

### 4. Tiers as Display Labels Only (No Multiplier)
Tiers exist purely for display context — no grade multiplier is applied:
- **Tier 1:** 100+ targets — Featured Starter
- **Tier 2:** 70–99 targets — Role Starter
- **Tier 3:** 50–69 targets — Rotational

Sample size is handled by `weighted_avg_grade = avg_grade × √targets` in Cell 6.

### 5. √Targets Weighting
To stabilize grades for smaller sample players:
```python
weighted_avg_grade = avg_grade * np.sqrt(targets)
```
A player with 185 targets (√185 = 13.6) carries more weight than one with 65 targets (√65 = 8.1). This is the primary defense against small-sample inflation.

### 6. Within-Season Normalization
All metrics are normalized to 0–100 **within each season**, not across all seasons. This ensures 2016 grades are comparable to 2025 grades even as the league evolves.

### 7. Final Rescaling (40–99)
The composite score is rescaled from 0–100 to **40–99** to match PFF's grading scale:
- 90–99: Elite
- 80–89: Pro Bowl
- 70–79: Above Average Starter
- 60–69: Average Starter
- 50–59: Below Average
- 40–49: Fringe Starter

---

## Play-Level Grading Engine (Cell 4)

### Separation Grade (`sep_grade`)
Uses `cp` as coverage proxy. Outcome (caught/not) only matters when `cp < 0.50`:
```
cp >= 0.75 → +1.0 (open, receiver did job regardless of outcome)
cp 0.50–0.75 → +0.5 (moderately open)
cp 0.30–0.50 + caught → +1.5 (won in tight coverage)
cp 0.30–0.50 + not caught → -0.5
cp < 0.30 + caught → +2.0 (elite contested catch)
cp < 0.30 + not caught → 0.0 (neutral, too difficult)
Wide open drop (cp >= 0.75 + not caught, high leverage) → -3.0
Wide open drop (cp >= 0.75 + not caught, normal) → -2.0
```

### Catch Point Grade (`cp_grade`)
Uses `adj_cpoe` (QB-adjusted CPOE):
```
caught + adj_cpoe > 10 → +1.0
caught + adj_cpoe >= 0 → 0.0 (expected catch)
caught + adj_cpoe < 0 → +1.5 (harder than expected)
incomplete + adj_cpoe > 10 + high leverage → -3.0
incomplete + adj_cpoe > 10 + normal → -2.0
incomplete + adj_cpoe >= 0 + high leverage → -2.0
incomplete + adj_cpoe >= 0 + normal → -1.0
incomplete + adj_cpoe < 0 → 0.0 (too difficult, neutral)
```

### YAC Grade (`yac_grade`)
Uses `yac_over_expected = yards_after_catch - xyac_mean_yardage`:
```
yac_oe > 3.0 → +1.5
yac_oe 1.0–3.0 → +0.5
yac_oe -1.0 to 1.0 → 0.0
yac_oe -3.0 to -1.0 → -0.5
yac_oe < -3.0 → -1.0
```

### Down & Distance Multiplier (`dd_multiplier`)
Applied to ALL grades (positive and negative):
```
3rd/4th + 5+ yards to go → 1.5x
3rd/4th + <5 yards to go → 1.2x
All other downs → 1.0x
```

### Explosive Play Bonus (`exp_bonus`)
```
air_yards >= 30 + caught → +1.0
air_yards >= 20 + caught → +0.5
Otherwise → 0.0
```

### Composite Play Grade
```python
play_grade = (
    (sep_grade * W_SEPARATION +
     cp_grade  * W_CATCH_POINT +
     yac_grade * W_YAC)
    * dd_multiplier
    + exp_bonus
)
```

---

## Grade Distribution Calibration

The target play grade distribution is:
- **Mean:** ~0.20–0.25 (slightly positive, mirrors real NFL data)
- **Median:** ~0.20
- **Min:** ~-1.3 or lower
- **Max:** ~2.6
- **Std:** ~0.46–0.48

Current achieved: mean=0.215, std=0.478, min=-1.260, max=2.605 ✅

---

## Notebook Cell Structure

| Cell | Purpose | Key Function/Data |
|---|---|---|
| 1 | Imports & Config | Weights, seasons, thresholds |
| 2 | NGS Data | `nfl.load_nextgen_stats()` |
| 3 | Play-by-Play Data | `nfl.load_pbp()` |
| QB Adj Cell | QB Quality Adjustment | Compute `qb_avg_cpoe`, merge onto pbp, create `adj_cpoe` |
| 4 | Play-Level Grading Engine | `grade_separation()`, `grade_catch_point()`, `grade_yac()` |
| 5 | Aggregate + Merge | pbp → seasonal, merge with NGS |
| 6 | Scoring | `weighted_avg_grade`, normalize, rescale 40–99 |
| 7 | Top 25 Output | 2025 season rankings |
| YPRR Cell | YPRR + Snap Data | `nfl.load_player_stats()`, `nfl.load_snap_counts()` |

**Critical cell-order dependency:** QB adjustment cell MUST run before Cell 4. If kernel is restarted, run `Kernel → Restart & Run All`.

---

## Known Issues & Next Steps

### Currently In Progress
- **nflreadpy migration** — Mid-migration. `load_nextgen_stats()` and `load_pbp()` are confirmed working. `load_player_stats()` and `load_snap_counts()` parameter names need verification. Run `print(dir(nfl))` and `help(nfl.load_player_stats)` to check correct syntax.
- **YPRR integration** — Not yet added to scoring. Seasonal data pull is failing due to wrong parameter name in nflreadpy.
- **Snap adjustment** — Not yet added to scoring. Snap count pull is failing for same reason.

### Known Model Limitations
- **No route running grade** — Release, stem/break, and separation-at-break grades require frame-level tracking data not available in any open source dataset. YPRR is our best available proxy.
- **No contested catch metric** — FTN data only goes back to 2022, insufficient for full 2016–2025 model. Decided not to include for consistency.
- **Separation at break point** — NGS measures separation at time of throw, not at route break. These are different moments. Our CP-based proxy partially compensates.

### Validation Findings (vs PFF 2025)
**Strong agreements:** Puka Nacua (#1), Jaxon Smith-Njigba (#2), George Pickens
**Major gaps:**
- Drake London — completely missing from our Top 25 (PFF top 3). Needs YPRR + route running signal.
- Terry McLaurin — missing from Top 25 (PFF top 10). Contested catch specialist not captured.
- Christian Watson — undervalued due to injury/snap issue. Snap adjustment will help.
- Justin Jefferson — ranks #23, should be top 10. QB adjustment helped but catch point still low due to JJ McCarthy.

---

## What NOT To Do

- **Do not overfit to individual players** — We caught ourselves adjusting the model specifically to fix Jefferson's grade. Validate against groups of players, not individuals.
- **Do not use `nfl_data_py`** — Archived, no 2025 data. Use `nflreadpy` exclusively.
- **Do not add tier multipliers** — Tiers are display labels only. Sample size is handled by √targets weighting.
- **Do not use raw `cpoe`** — Always use `adj_cpoe` (QB-adjusted) for catch point grading.
- **Do not normalize across seasons** — Always normalize within each season separately.
- **Do not string together `import nfl_data_py as nfl` and `nflreadpy`** — They have different function names.

---

## PFF Methodology Reference

PFF grades every player on every play on a -2 to +2 scale. For WRs specifically:
- **Route running** is the largest component (~40% of grade) — we approximate with YPRR
- **Catch point** includes contested catches, drops, difficult grabs (~30%)
- **YAC** includes broken tackles, explosive plays (~30%)
- Grades are context-sensitive: same yardage on 3rd and long grades higher than 1st and 10
- PFF measures separation at the **route break point**, not at catch point
- Coverage busts / scheme gifts grade as **neutral (0)** — receiver gets no credit

---

## File Outputs

| File | Description |
|---|---|
| `wr_data_pipeline.ipynb` | Main notebook — all pipeline code |
| `wr_master_dataset.csv` | Seasonal WR grades 2016–2025 |
| `wr_playbyplay.csv` | Every targeted WR pass play |
| `wr_top25_2024.csv` | Top 25 WRs by composite score |
| `WRU_PROJECT_CONTEXT.md` | This file |

---

*Last updated: March 2026*
*Model version: Play-level grading v1 with QB adjustment*
