# Shadow Zone Module

**Live app:** [sdm-commits.github.io/shadow-zone](https://sdm-commits.github.io/shadow-zone/)

Pitch sequence effects, platoon edge vulnerability, and ABS challenge impact across MLB seasons (2021-2025). Built from Statcast pitch-level tracking data.

## What It Does

Analyzes every called pitch in the buffer zone around the true strike zone edge to quantify:

- **Umpire Stolen Strikes (USS):** Called strikes on pitches actually outside the zone, weighted by game-state run expectancy (RE288). These are framing gains that "survive" ABS because challenging them has negative expected value.
- **Umpire Lost Strikes (ULS):** Called balls on pitches actually inside the zone, also RE288-weighted. Under ABS, the defense can challenge these to recover value.
- **Net ABS Impact:** ULS Runs minus USS Runs. Positive means the defense benefits from ABS (recovers more than it loses). Negative means elite framing value gets eroded.
- **Sequence effects:** How the previous pitch type (fastball, breaking ball, offspeed) shifts umpire perception on borderline calls.
- **Platoon splits:** Inside/outside/top/bottom edge called strike rates broken out by batter-pitcher handedness matchup.
- **Pitcher-catcher synergy:** Battery-level framing analysis with deployment recommendations.

## App Tabs

| Tab | Content |
|-----|---------|
| **Overview** | League-wide sequence CS rates, sequence pair matrix, ABS impact summary, catcher USS/ULS leaders |
| **Platoon Splits** | Edge CS rate heatmaps by matchup (RHP vs RHB, etc.), edge vulnerability analysis |
| **Pitcher Profiles** | Per-pitcher search with sequence profile, platoon edges, battery splits, shadow zone distribution, catcher synergy, scouting notes |
| **Team View** | Staff rankings by CS rate/USS/ULS/Net ABS, team matchup edge tables |
| **Catchers** | Per-catcher search with edge profiles, pitcher battery splits, zone profiles, synergy scores, framing notes, zone comparison tool |
| **Scouting Card** | Exportable pitcher scouting reports with leverage tier analysis |
| **Player Dev** | Catcher deployment recommendations based on synergy scores |
| **Front Office** | Trade value and roster construction analysis using ABS impact projections |
| **ABS Value** | ABS tier classification (unchallengeable / uncontested / exposed), per-pitcher breakdown with challenge exposure rates |

## Shadow Zone Definition

Our shadow zone is computed geometrically:

1. **Ball radius adjustment.** Statcast tracks the ball center (`plate_x`/`plate_z`). Under ABS, a pitch is a strike if *any part* of the ball touches the zone. We expand zone edges by ball radius (~1.5 in) to model this — matching how ABS determines whether a challenged pitch is overturned or confirmed.

2. **Distance-based shadow band.** The shadow zone is all pitches within 3 inches (0.25 ft) of the true zone edges, from either side. This captures pitches just outside (where catchers frame) and just inside (where umpires miss calls).

3. **Per-batter zones.** We use Statcast's operator-measured `sz_top`/`sz_bot` for each pitch, not fixed zone heights. All years (2021-2025) use operator-measured values. Hawk-Eye tracked strike zone data is expected in 2026.

### Key Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `BALL_RADIUS` | 0.125 ft (1.5 in) | Ball radius for ABS true zone expansion |
| `PLATE_HALF_WIDTH` | 0.708 ft (8.5 in) | Half of 17-inch plate width |
| `SHADOW_MAX_DIST` | 0.25 ft (3 in) | Shadow band width from zone edge |
| `HAWKEYE_SD` | 0.014 ft | Hawk-Eye tracking standard deviation |
| `CHALLENGE_VALUE` | 0.20 runs | Cost per challenge attempt |

### USS/ULS Pipeline

For each called strike outside the zone (USS) or called ball inside the zone (ULS):

1. Compute `zone_dist` — distance past the nearest zone edge, accounting for ball radius.
2. Compute Hawk-Eye confidence: `norm.cdf(zone_dist, loc=0, scale=0.014)` — probability ABS confirms the call.
3. Compute challenge threshold: `min(CHALLENGE_VALUE / re_delta, 1.0)` — where `re_delta` is the RE288 run value swing for that count/bases/outs state.
4. If confidence < threshold, the pitch is **uncontested** (not worth challenging) and counts toward USS or ULS.

## Data Pipeline

### Input

Statcast parquet files with pitch-level tracking data:
- **2021-2024:** `statcast_cache/statcast_{year}.parquet`
- **2025:** `statcast-ingest/store/cumulative/pitches_season.parquet`

### Generation

```bash
# Single year
python generate_shadow_zone_json.py --year 2025

# All historical years
python generate_shadow_zone_json.py --year 2021 2022 2023 2024
```

### Output

`shadow_zone_{year}_full.json` — one file per season containing:

```
summary              League-wide totals (pitches, shadow zone size, CS rate, USS/ULS runs)
pitcherMeta          Per-pitcher metadata + USS/ULS/Net ABS
pitcherSequences     Per-pitcher pitch-type sequence CS rates
pitcherPlatoon       Per-pitcher matchup x edge CS rates
leagueMatchupEdges   League-wide matchup x edge CS rates
teamMatchupEdges     Per-team matchup x edge CS rates
catcherMeta          Per-catcher metadata + USS/ULS/Net ABS
catcherPlatoon       Per-catcher edge profiles + USS/ULS
pitcherCatcherPairs  Battery splits + USS
teamUSS / teamULS / teamNetABS   Team-level aggregates
pitcherZoneProfiles  8-zone grid CS rates per pitcher
catcherZoneProfiles  8-zone grid CS rates per catcher
leagueAvgZones       League average 8-zone CS rates
synergyScores        Pitcher-catcher synergy ratings
deploymentRecs       Optimal catcher deployment recommendations
leverageTiers        Count/situation leverage tier analysis
```

### Qualification Thresholds

| Role | Minimum |
|------|---------|
| Pitcher | 30 shadow zone called decisions |
| Catcher | 100 shadow zone called decisions |
| Battery pair | 30 decisions together |
| USS/ULS pitcher | 10 stolen/lost strike decisions |
| USS/ULS catcher | 50 stolen/lost strike decisions |

## File Structure

```
shadow-zone/
  index.html                      Single-page app (HTML + CSS + JS, no build step)
  shadow_zone_2021_full.json      Season data
  shadow_zone_2022_full.json
  shadow_zone_2023_full.json
  shadow_zone_2024_full.json
  shadow_zone_2025_full.json
  README.md
```

The app is a single self-contained HTML file with no external dependencies beyond Google Fonts. Year selector loads JSON files on demand via fetch.

## Methodology Credits

- **Tom Tango** — Challenge expected value framework and RE288 run expectancy tables
- **Baseball Savant** — Zone probability models and Statcast tracking data
- **Statcast** — Pitch-level tracking data (`plate_x`, `plate_z`, `sz_top`, `sz_bot`)
