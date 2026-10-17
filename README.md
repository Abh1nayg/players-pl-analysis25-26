# Clustering Premier League Outfield Players (2025-26)

Unsupervised K-Means clustering of Premier League outfield players from FBref season stats, built to compare how **feature scaling** changes the clusters you get.

## Key Findings

**1. Z-score and MinMax scaling give very different clusterings.**
Same data, same features, same algorithm. Only the scaler changed.

| | MinMax | Z-score |
|---|---|---|
| Best silhouette | k=3 (0.2715) | k=2 (0.3629) |
| k used | **3** | **6** |
| Rare behaviours (penalty takers, red cards) | Penalty takers merged with goalscorers; red cards not separated | Each gets its own cluster |

MinMax squeezes every feature into 0-1 using min and max, so a few extreme values (a player with many penalties, a player with several red cards) compress everyone else. Z-score keeps those outliers far from the mean, so K-Means can isolate them.

**2. The two scalers point to different values of k.**
- **MinMax:** silhouette peaks clearly at k=3 (0.2715), and k=3 gives clean, interpretable groups.
- **Z-score:** silhouette is highest at k=2 (0.3629) but that is just a coarse split. After a dip at k=3-4 it recovers at k=6 (0.2330) and k=8 (0.2389), and the inertia curve flattens after about k=6 (2107 at k=5, 1827 at k=6, 1719 at k=7). I used k=6 because it gives the most interpretable groups.

Silhouette scores are low overall (0.17-0.36), so these are loose groupings, not clean natural clusters.

| k | MinMax inertia | MinMax silhouette | Z-score inertia | Z-score silhouette |
|---|---|---|---|---|
| 2 | 120.41 | 0.2273 | 3335.23 | 0.3629 |
| 3 | 84.49 | **0.2715** | 2926.63 | 0.2213 |
| 4 | 75.68 | 0.2252 | 2507.41 | 0.1678 |
| 5 | 69.73 | 0.2340 | 2107.35 | 0.2061 |
| 6 | 66.38 | 0.2066 | 1826.58 | 0.2330 |
| 7 | 62.83 | 0.1840 | 1718.68 | 0.2107 |
| 8 | 57.31 | 0.2356 | 1611.69 | 0.2389 |
| 9 | 53.35 | 0.2019 | 1495.65 | 0.2177 |
| 10 | 50.18 | 0.2077 | 1472.91 | 0.2159 |

## Problem

Positions (DF/MF/FW) say little about *how* a player contributes. Can player stats alone group players into meaningful roles, and how sensitive is the answer to preprocessing choices?

## Data

- Source: [FBref, Premier League 2025-26 standard stats](https://fbref.com/en/comps/9/2025-2026/stats/2025-2026-Premier-League-Stats#all_stats_standard)
- Goalkeeper stats (`data/keeper.csv`, not used in the notebook yet): [FBref keepers page](https://fbref.com/en/comps/9/2025-2026/keepers/2025-2026-Premier-League-Stats)
- Season: **2025-2026**. Stats are a snapshot from when the page was copied, not a final table.
- Copied manually from FBref's table (or its CSV export) into `data/players.csv` (551 rows, ~60 KB), so the file is included in this repo.
- Data belongs to FBref / Sports Reference. Check their [terms of use](https://www.sports-reference.com/termsofuse.html) before reusing or redistributing it.

To refresh the data: open the FBref link above, use *Share & Export → Get table as CSV*, and save as `data/players.csv`.

## Method

1. **Cleaning:** dropped the FBref match-link column, stripped the flag prefix from `Nation` (`us USA` -> `USA`), split joined positions (`MFFW` -> `MF,FW`).
2. **Filtering:**
   - Removed goalkeepers (different role and stats).
   - Removed players with fewer than 180 minutes (too small a sample for per-90 numbers). This leaves **414 players**.
3. **Features (11):** `Age`, `Min`, `PK`, `PKatt`, `CrdY`, `CrdR`, `Gls90`, `Ast90`, `G+A90`, `G-PK90`, `G+A-PK90`.
4. **Why per-90:** raw goals and assists mostly measure minutes played. I dropped the raw `Gls`, `Ast`, `G+A` and `G-PK` and kept the per-90 versions, so a bench player with a high rate is comparable to a regular starter.
5. **Scaling:** both `StandardScaler` (z-score) and `MinMaxScaler`, fit on the 11 features, clustered separately.
6. **Choosing k:** KMeans (`random_state=42`) for k=2..10 on each scaled dataset. I looked at the elbow plot (inertia) and silhouette score together; see the table above.
7. **Profiling:** cluster labels were joined back to player names and per-cluster feature means were computed.

## Results

**MinMax, k=3**

| Cluster | Description | Avg minutes | Notable |
|---|---|---|---|
| 0 | Goalscorers / penalty takers | 1648 | PK 0.69, Gls90 0.38 |
| 1 | High-minutes, card-heavy, low attacking output | 2469 | CrdY 5.95, Gls90 0.08 |
| 2 | Low-minutes squad players | 1001 | Low everything |

**Z-score, k=6**

| Cluster | Description | Avg minutes | Notable |
|---|---|---|---|
| 0 | Younger attacking midfielders and wide players | 1310 | Age 23.6, highest Ast90 (0.17) of the low-scoring groups |
| 1 | Regular starters, mostly defensive | 2579 | CrdY 6.3, low output |
| 2 | Fringe / low-minutes squad players | 988 | Low everything |
| 3 | Penalty takers | 2382 | PK 3.8, PKatt 4.5 |
| 4 | Regular goalscorers | 1561 | Gls90 0.41, G-PK90 0.40 |
| 5 | Red-card outliers | 2148 | CrdR 1.05 |

The cluster names are my interpretation of the feature means, not something the algorithm outputs. Cluster numbers are arbitrary and can change between runs or library versions.

## Limitations

- Features are only goals, assists, cards, penalties, age and minutes. There are no passing, defensive or positional stats, so roles are only roughly captured.
- Low silhouette scores mean clusters overlap.
- A single season and a single `random_state`; I haven't checked stability across seeds.
- `Age` and `Min` are clustered on along with performance stats, so part of each cluster reflects playing time and age.

## How to run

```bash
git clone <this-repo>
cd players-pl-analysis25-26
python -m venv .venv
# Windows: .venv\Scripts\activate    macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook notebooks/players.ipynb
```

Run the notebook from the `notebooks/` folder (the default when you open it from Jupyter), since it reads `../data/players.csv`.

Developed with Python 3.14, pandas 3.0, scikit-learn 1.9.

## Project structure

```
data/players.csv       FBref outfield + GK stats, 2025-26
data/keeper.csv        FBref goalkeeper stats (not yet analysed)
notebooks/players.ipynb  cleaning, scaling, clustering, profiling
requirements.txt
```
