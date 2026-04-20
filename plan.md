# CBL Analytics Dashboard — Project Plan

## Overview

A pitch-level analytics platform for the Intercounty Baseball League (CBL), built from manually charted game data collected across all 9 teams during the 2025 season (May 10 – August 30). The project produces three core metrics — **Pitch+**, **xBA**, and **Deception Score** — delivered through an interactive web dashboard designed for coaching staff and portfolio presentation.

---

## Data Source

- **Input:** Shared Google Sheet with one tab per game, charted by analysts across the league
- **Coverage:** ~30 games personally charted, plus access to the full league dataset from all analysts
- **Granularity:** One row per pitch

### Raw Columns

| Column | Description |
|---|---|
| Half Inning | Top/Bottom + inning number |
| Inning | Inning number |
| Outs | Outs at time of pitch |
| Balls | Ball count at time of pitch |
| Strikes | Strike count at time of pitch |
| Count | Current count string (e.g., '1-2') |
| Batter_Team | Team at bat |
| Pitcher_Team | Team in the field |
| Date | Game date |
| Home Team | Home team name |
| Away Team | Away team name |
| Pitcher_R_Allowed | Runs allowed (pitcher) |
| Pitcher_ER_Allowed | Earned runs allowed (pitcher) |
| Batter | Batter name |
| Pitcher | Pitcher name |
| Batter_Side | L or R |
| Pitcher_Side | L or R |
| Pitch_Type | Fastball (FB) or Offspeed (OS) |
| Outcome | Pitch-level result (see full list below) |
| Quality of Contact | Ground Ball, Line Drive, Pop Up, Fly Ball (batted balls only) |
| Spray Chart | Pull, Straightaway, Opposite Field (batted balls only) |
| Runners | Binary encoding of base state (e.g., 000, 100, 010, 001) |
| Pitch_Location_X | Continuous X coordinate on zone grid |
| Pitch_Location_Y | Continuous Y coordinate on zone grid |
| Time To Plate (sec) with Man on First | Pitcher delivery time |

### Outcome Values

**Non-terminal:** Ball, Called Strike, Swinging Strike, Foul

**Terminal (batted ball):** Single, Double, Triple, Home Run, Groundout, Flyout, Lineout, Popout, Double Play, Triple Play, Sacrifice Bunt, Sac Bunt Double Play, Sacrifice Fly, Sac Fly Double Play, Error

**Terminal (non-contact):** Strikeout Looking, Strikeout Swinging, Dropped Third Strike Looking, Dropped Third Strike Swinging, Walk, Intentional Walk, Hit By Pitch, Pickoff, Caught Stealing, Truncated Out

---

## Data Pipeline

### Ingestion

Python script pulls all tabs from the shared Google Sheet via the Google Sheets API, normalizes column names, and loads into a unified dataframe. Export to a local database (SQLite for simplicity, PostgreSQL for portfolio polish). Run weekly or on-demand throughout the season.

### Derived Columns

| Column | Derivation |
|---|---|
| `game_id` | Hash of date + home_team + away_team |
| `at_bat_id` | Hash of game_id + half_inning + batter; incremented on count reset to 0-0 or batter change within same half inning |
| `pitch_seq` | Integer starting at 1 within each at_bat_id, reconstructed from count progression |
| `zone_x` / `zone_y` | Raw X/Y binned into a 5×5 grid (integers 1–5) |
| `is_terminal` | Boolean — pitch ends the plate appearance |
| `is_batted_ball` | Boolean — outcome produces contact |
| `run_value` | Linear weight assigned to the outcome (see below) |

---

## Metric 1 — Pitch+

### Concept

A location-adjusted pitch quality metric. For every pitch thrown, assigns a value based on how effective that pitch type is in that specific zone cell, relative to the league average. Scaled to 100 (league average).

### Run Value Assignments (Pitcher Perspective)

**Terminal batted-ball outcomes:**

| Outcome | Run Value |
|---|---|
| Home Run | −1.40 |
| Triple | −1.07 |
| Double | −0.77 |
| Single | −0.47 |
| Sacrifice Fly | −0.27 |
| Error | −0.27 |
| Groundout | +0.28 |
| Flyout | +0.28 |
| Lineout | +0.28 |
| Popout | +0.28 |
| Double Play | +0.56 |
| Triple Play | +0.84 |

**Terminal non-contact outcomes:**

| Outcome | Run Value |
|---|---|
| Strikeout Looking | +0.30 |
| Strikeout Swinging | +0.30 |
| Dropped Third Strike Looking | +0.15 |
| Dropped Third Strike Swinging | +0.15 |
| Walk | −0.33 |
| Intentional Walk | −0.33 |
| Hit By Pitch | −0.35 |

**Non-terminal outcomes:**

| Outcome | Run Value |
|---|---|
| Called Strike | +0.065 |
| Swinging Strike | +0.075 |
| Foul | +0.040 |
| Ball | −0.055 |

### Formula

1. For each zone cell `(zone_x, zone_y)` and pitch type `(FB/OS)`, compute the league-average run value per pitch:

```
league_avg(zone_x, zone_y, pitch_type) = mean(run_value) for all pitches in that cell and type
```

2. A pitcher's Pitch+ on any individual pitch:

```
pitch+ = ((pitcher_run_value − league_avg) / abs(league_avg)) × 100 + 100
```

3. Aggregate across all pitches for overall Pitch+, or split by pitch type for Fastball+ and Offspeed+.

### Minimum Thresholds

- 15 pitches per cell before displaying individual cell grades
- Fall back to a coarser 3×3 grid if data is too thin in any cell

---

## Metric 2 — xBA (Expected Batting Average)

### Concept

An expected outcome model from Quality of Contact + Spray Direction, bypassing fielding, luck, and park effects. Identifies batters who are overperforming or underperforming relative to their contact quality.

### Bucket Matrix

4 contact types × 3 spray directions = 12 buckets:

|  | Pull | Straightaway | Opposite Field |
|---|---|---|---|
| Ground Ball | GB-Pull | GB-Straight | GB-Oppo |
| Line Drive | LD-Pull | LD-Straight | LD-Oppo |
| Fly Ball | FB-Pull | FB-Straight | FB-Oppo |
| Pop Up | PU-Pull | PU-Straight | PU-Oppo |

### Formula

1. Filter to `is_batted_ball = true`
2. For each bucket, compute league-wide batting average:

```
league_ba(contact, spray) = hits / total_batted_balls in that bucket
```

Where hits = Single + Double + Triple + Home Run.

3. A batter's xBA:

```
xBA = mean(league_ba for each of their individual batted balls)
```

4. Extend to xSLG by replacing hit/no-hit with total bases:

```
league_slg(contact, spray) = total_bases / total_batted_balls in that bucket
```

5. Luck Score:

```
luck = actual_BA − xBA
```

Positive = overperforming, negative = underperforming.

---

## Metric 3 — Deception Score (Tunneling)

### Concept

Measures how effectively a pitcher disguises their offspeed by locating it near their fastball. Based on the principle that tighter "tunnel distances" between consecutive pitches of different types produce more deception and better outcomes.

### Formula

1. For each pitcher, identify consecutive pitch pairs within an at-bat where pitch type changes (FB→OS or OS→FB).

2. Compute tunnel distance for each pair:

```
tunnel_distance = sqrt((x2 − x1)² + (y2 − y1)²)
```

3. Measure the run value of the second pitch in each pair.

4. League-wide validation: group tunnel distances into buckets (tight / medium / wide) and compare second-pitch run values to confirm tighter tunnels produce better results.

5. Pitcher's Deception Score:

```
deception_score = mean(run_value of second pitch) for all pitch-type-change pairs where tunnel_distance < threshold
```

6. Scale to 100 like Pitch+.

### Minimum Thresholds

- 20 pitch-type transition pairs before displaying a pitcher's Deception Score

---

## Tech Stack

| Layer | Technology |
|---|---|
| Data Ingestion | Python + Google Sheets API |
| Database | SQLite or PostgreSQL |
| Metric Computation | Python (pandas, numpy) |
| Backend/API | Flask or FastAPI |
| Frontend/Dashboard | React or Next.js |
| Visualizations | Plotly or D3.js |
| Hosting | Vercel (frontend) + Railway (backend) |
| Version Control | GitHub (public repo with documentation) |

---

## Dashboard Layout

### View 1 — League Overview

- Leaderboards for Pitch+, xBA, and Deception Score
- Filterable by team
- Season-level snapshot of top performers in each metric

### View 2 — Pitcher Profile

- Overall Pitch+ with 5×5 strike zone heatmap (color-coded above/below average)
- Split by Fastball+ and Offspeed+
- Deception Score with scatter plot (tunnel distance vs. second-pitch outcome)
- Pitch usage breakdown and sequencing tendencies by count

### View 3 — Batter Profile

- Actual BA vs. xBA and xSLG
- Spray chart visualization color-coded by contact quality
- "Damage zone" heatmap (where in the zone they produce hard contact)
- Luck score indicator

### View 4 — Matchup Tool

- Select a pitcher and batter
- Overlay pitcher's zone heatmap with batter's damage zone
- Highlights where the pitcher should attack and where to avoid

---

## Timeline

| Phase | Dates | Deliverables |
|---|---|---|
| Pre-season setup | Now – May 10 | Data pipeline, database schema, metric calculations built with synthetic/placeholder data |
| Early season | May 10 – mid-June | Collect data, establish league baselines, first version of all three metrics running |
| Dashboard build | Mid-June – July | Build interactive dashboard, iterate on visualizations, validate metrics with coaches |
| Polish & present | August | Final polish, methodology documentation, portfolio-ready presentation |

### Key Risk

Tunneling / Deception Score depends on sufficient per-pitcher sample sizes. With only two pitch type categories, individual pitcher scores may be noisy for relievers with limited innings. Build in a minimum pitch threshold before displaying individual metrics, and consider showing league-wide tunneling trends even when individual data is sparse.

---

## Amendments & Design Notes

### 1. Deception Score — Manual Charting Noise

**Risk:** X/Y coordinates are manually clicked, so small analyst errors can inflate tunnel distances, making a tight tunnel appear wide and suppressing a pitcher's Deception Score unfairly.

**Fix:**
- De-emphasize raw `tunnel_distance` as the primary filter. Treat it as a soft weight rather than a hard threshold.
- Primary signal: outcome trends for pitch-type-change sequences (does the second pitch produce better-than-average run values?).
- Add a **Confidence Interval** to each pitcher's Deception Score, derived from the variance across their transition pairs. Wide CI = interpret with caution.
- Display a caveat on the dashboard when a pitcher's score has high uncertainty due to sample size or coordinate variance.

### 2. Pitch+ Scaling — Z-Score Normalization

**Risk:** `league_avg` run values per cell are often very small decimals (e.g., 0.015). Dividing by `abs(league_avg)` produces massive, uninterpretable swings.

**Fix:** Replace the denominator with the standard deviation of run values in that cell/pitch-type bucket:

```
pitch+ = ((pitcher_run_value − league_avg) / stdev(run_values in cell)) × 10 + 100
```

The multiplier (10) is tunable to keep scores within a realistic ~80–120 range. Fall back to the league-wide stdev for that pitch type when a cell has too few pitches for a stable stdev.

### 3. Whiff%

A simple sanity-check metric added to the Pitcher Profile page.

```
whiff% = swinging_strikes / (swinging_strikes + fouls + batted_ball_outcomes)
```

A high Pitch+ pitcher with a low Whiff% is generating value through location, not pure stuff — useful context for coaches.

### 4. Data Integrity Dashboard (View 5)

A dedicated page for portfolio presentation demonstrating analytical maturity.

**Metrics to display per analyst/charter:**
- Called strike rate vs. league average (flags a charter calling the zone tight or loose)
- Swinging strike rate vs. league average (detects systematic pitch-type misclassification)
- Batted-ball type distribution (does one charter log more line drives than others?)
- Games charted count and coverage map

**Implementation:** Requires adding an `analyst` column to the ingestion schema so each pitch row is traceable to its source charter. Compute per-analyst distributions, compare to the league mean via z-scores, and flag analysts >1.5 SD from the mean in any category.