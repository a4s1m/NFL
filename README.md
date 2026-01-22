# NFL
# Analyzing Player Movement When the Ball Is in the Air 

## Project Overview
This project analyzes **defensive player pursuit behavior during the ball-in-air phase** of NFL passing plays using high-frequency player-tracking data. By constructing an optimal pursuit path toward the ball’s landing location and measuring deviations from this ideal trajectory, the project introduces a novel metric — the **Pursuit Efficiency Index (PEI)** — to quantify defensive movement efficiency.

The analysis evaluates **timing accuracy, directional discipline, and distance efficiency**, providing insight into player performance that traditional football statistics do not capture.

---

## Research Objectives
- Model an **optimal pursuit path** for defenders when the ball is in the air
- Quantify how players deviate from optimal pursuit in:
  - Timing (longitudinal progress)
  - Direction (lateral drift)
  - Distance to ball (radial excess)
- Develop a unified efficiency metric (**PEI**) combining all deviation components
- Incorporate **football context rewards** (completion, interception, tight coverage)
- Aggregate PEI across an entire season to evaluate consistent defensive performance

---

## Dataset Overview

### Tracking Data
- **Source:** NFL player-tracking data derived from game film
- **Granularity:** 10 frames per second
- **Scope:** 18 weeks of regular-season games
- **Structure:**  
  - Each row represents **one player at one frame of a specific play**
  - Positional coordinates: `(x, y)`

### Input / Output Split
- **Input files:** Player movement before the ball is thrown  
- **Output files:** Player movement after the ball is thrown  
- Output data is restricted to **players of interest** (targeted defenders)

---

## Key Definitions

- **Player to Predict:** Defensive player assigned to coverage for a given play  
- **Total Frames:** All frames from ball release to catch / incompletion / interception  
- **Ball Landing Coordinates:** Final location of the ball  
  `(ball_land_x, ball_land_y)`  
- **Optimal Path:** Straight-line Euclidean path from player position at throw to ball landing point  

---

## Key Assumptions
- **Straight-line optimality:** Best pursuit is a direct path to the ball landing point  
- **Uniform progress:** Expected progress is linear over time  
- **Frame independence:** Deviations are measured per frame, aggregated later  
- **Contextual realism:** Rewards and bonuses reflect real football outcomes  

---

## Data Merging & Feature Enrichment
To enable meaningful analysis, multiple NFL tables were merged into a unified dataset:

- **Tracking Table (Backbone):**
  - `(game_id, play_id, nfl_id, frame_id)`
- **Play-Level Metadata:**
  - Pass result (complete, incomplete, interception)
  - Ball landing coordinates
  - Route and pass characteristics
- **Player Context:**
  - Role (defender / targeted receiver)

This multi-stage merge allows raw motion data to be interpreted as **football-specific pursuit behavior**.

---

## Optimal Path Construction
The **Optimal Path** represents the ideal straight-line trajectory from the player’s position at ball release to the ball’s landing location.

- Initial distance:
- L = sqrt((x₀ − x_b)² + (y₀ − y_b)²)
- Progress is parameterized using:
- α(t) ∈ [0, 1]

where α increases linearly from throw frame to landing frame.

This constructs an **ideal spatial schedule** for comparison against actual movement.

---

## Path Deviation (Longitudinal Deviation)
Path Deviation measures **how early or late** a player is relative to the optimal schedule.

- Player displacement projected onto optimal direction:
- along(t) = r(t) · u
- Expected progress:
- along_opt(t) = α(t) · L
- Longitudinal deviation:
- path_dev(t) = along(t) − along_opt(t)

**Interpretation:**
- Positive → ahead of schedule  
- Negative → behind schedule  
- Zero → perfect timing  

---

## Directional Deviation (Lateral Deviation)
Directional Deviation measures **steering accuracy** — how well the player stays aligned with the optimal pursuit angle.

Using the 2-D cross product:
dir_dev(t) = u_x · r_y(t) − u_y · r_x(t)

**Interpretation:**
- Positive → drifting left  
- Negative → drifting right  
- Zero → perfect directional alignment  

Magnitude reflects **lateral inefficiency**.

---

## Radial Excess (Distance Inefficiency)
Radial Excess measures how much **farther a player is from the ball** than required under optimal pursuit.

- Actual distance:
- d_actual(t)
- Optimal distance schedule:
- d_opt(t) = (1 − α(t)) · L
- Radial excess:
- rad_excess(t) = max(0, d_actual(t) − d_opt(t))

Players are **not penalized** for arriving earlier than expected.

---

## Pursuit Efficiency Index (PEI)
The **Pursuit Efficiency Index (PEI)** combines:
- Path Deviation
- Directional Deviation
- Radial Excess

Each component is:
- Normalized by total pursuit distance
- Averaged
- Inverted

**PEI Scale:**
- `1.00` → Perfect pursuit
- `0.00` → Highly inefficient pursuit

This creates a **stable, interpretable, and comparable** metric across players and plays.

---

## Contextual Rewards & Adjustments

- **Completion Reward:**  
If the receiver catches the ball → reduced penalties (0.3× MAE)
- **Interception Bonus:**  
Defender rewarded identically to a perfect completion
- **Tight Coverage Bonus:**  
If defender is closer than receiver during final 10 frames of an incomplete pass → 50% penalty reduction

These adjustments align PEI with **real football outcomes**, not just geometry.

---

## Seasonal Aggregation
To evaluate consistency, PEI values are **averaged across the entire season** for each player.

This reduces play-level noise and highlights:
- Sustained pursuit discipline
- Recognition speed
- Route anticipation quality

---

## Interactive Dashboard
**Average PEI for 2023 Season by Position and Route**

🔗 https://lookerstudio.google.com/reporting/c22eb038-8776-4402-a5c4-22cbd19d4f5c/page/mQ8hF

---

## Key Insights
- PEI captures **movement quality** not reflected in tackles or pass breakups
- High-PEI defenders:
- Recognize ball trajectory early
- Maintain clean pursuit angles
- Avoid unnecessary lateral drift
- Low-PEI defenders:
- Take indirect paths
- Arrive late despite speed
- Lose efficiency through over-correction

---

## Conclusion
The Pursuit Efficiency Index (PEI) provides a **comprehensive, interpretable evaluation of defensive pursuit behavior** by integrating timing, directional discipline, and distance efficiency. By grounding the metric in real football outcomes and aggregating performance across a season, PEI offers a powerful alternative lens for evaluating defensive performance beyond traditional statistics.

---

## Tools & Techniques
- Python 
- Vector geometry
- Linear & beta normalization
- Time-series frame analysis
- Data visualization
- Looker Studio dashboards

---
