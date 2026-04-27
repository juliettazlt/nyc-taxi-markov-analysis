# NYC Taxi Zone Importance — Markov Chain Analysis

## Overview

This project applies a **Markov Chain / PageRank-style algorithm** to NYC Yellow Taxi trip data (January 2026, ~3.7M trips) to rank each of the 263 taxi zones by their structural importance in the city's mobility network.

Rather than simply counting trips, the model treats the taxi network as a directed graph and computes the **steady-state probability distribution** — the probability that a random taxi passenger ends up in a given zone after many rides. Zones with high scores are true mobility hubs: they attract and redistribute traffic across the entire network.

---

## Dataset

| Source | Description |
|--------|-------------|
| [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) | Yellow taxi trip records — January 2026 (3,724,889 rows) |
| [Taxi Zone Lookup CSV](https://data.cityofnewyork.us/) | Maps LocationID → Zone, Borough, Service Zone (265 zones) |
| [NYC Taxi Zone Boundaries (Socrata API)](https://data.cityofnewyork.us/Transportation/NYC-Taxi-Zones/d3c5-ddgc) | GeoJSON polygons for choropleth visualization |

---

## Methodology

```
Raw Trips  →  Data Cleaning  →  Origin-Destination Graph  →  Transition Matrix  →  Markov Iterations  →  Importance Scores
```

1. **Data Loading** — Load parquet trip data and zone lookup CSV
2. **Data Cleaning** — Remove outlier trips (e.g., distances > 100 miles), remap duplicate location IDs, drop unmappable zones (264, 265)
3. **Graph Construction** — Aggregate trips into unique (origin, destination) edges with counts
4. **Transition Matrix** — Normalize edge weights by outdegree → row-stochastic matrix `P` (265×265 sparse)
5. **Self-loop removal** — Drop edges where origin == destination before normalizing, preventing zones like EWR from trapping probability mass
6. **Markov Convergence** — Iterate `x = Pᵀ · x` for 50 steps starting from uniform distribution
7. **Geospatial Validation** — Merge scores with NYC zone polygons and render choropleth map
8. **Dynamic Pricing Analysis** — Correlate Markov scores and raw trip counts against zone revenue; compare predictive power via Pearson r and Spearman ρ

---

## Top 10 Most Important Zones

| Rank | Zone | Borough | Importance Score |
|------|------|---------|-----------------|
| 1 | Newark Airport | EWR | 0.0453 |
| 2 | Upper East Side North | Manhattan | 0.0370 |
| 3 | Upper East Side South | Manhattan | 0.0344 |
| 4 | Midtown Center | Manhattan | 0.0279 |
| 5 | Upper West Side South | Manhattan | 0.0234 |
| 6 | Lincoln Square East | Manhattan | 0.0234 |
| 7 | Murray Hill | Manhattan | 0.0232 |
| 8 | Lenox Hill West | Manhattan | 0.0229 |
| 9 | Times Sq/Theatre District | Manhattan | 0.0215 |
| 10 | East Village | Manhattan | 0.0215 |

---

## Business Questions This Analysis Can Answer

### 1. Where should Uber/Lyft prioritize driver supply?
> *Which zones have the highest steady-state importance and therefore the most persistent demand?*

High Markov scores indicate zones where passengers consistently arrive and depart — ideal locations for driver positioning to minimize wait times and idle miles.

### 2. Which zones are underserved relative to their network importance?
> *Are there high-importance zones with low driver availability or high surge pricing?*

Cross-referencing importance scores with surge pricing data or driver density could reveal supply gaps worth addressing.

### 3. Where should the city invest in transit infrastructure?
> *Which zones act as connectors that, if disrupted, would most fragment the mobility network?*

Zones with high importance scores are structural bottlenecks. Improving public transit alternatives there (bus lanes, subway expansions) would have outsized citywide impact.

### 4. How does airport connectivity shape the network?
> *Why does Newark Airport (EWR) rank #1 despite being outside NYC?*

EWR's top ranking reveals the asymmetry of airport trips: passengers arrive from all over the city but most return to the same few destination zones, concentrating flow.

### 5. Can importance scores predict revenue-optimal fare zones for dynamic pricing?
> *Do high-importance zones generate proportionally higher revenue per zone?*

By joining importance scores with `fare_amount` and `total_amount` columns, one could test whether Markov centrality is a better predictor of revenue density than raw trip counts.

### 6. Which boroughs are most isolated from the taxi network?
> *How do the Bronx, Staten Island, and outer Queens compare to Manhattan in network centrality?*

Low-scoring boroughs represent mobility deserts where taxi coverage is structurally weak — useful for policy targeting or ride-share expansion decisions.

---

## Project Structure

```
.
├── main.ipynb          # Full analysis notebook
├── requirements.txt    # Python dependencies
└── uber_report.html    # Rendered HTML report
```

---

## Requirements

```
pandas
numpy
scipy
sodapy
geopandas
shapely
matplotlib
pyarrow        # for reading .parquet files
```

Install with:
```bash
pip install -r requirements.txt
```

---

## How to Run

1. Download `yellow_tripdata_2026-01.parquet` from the [NYC TLC website](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) and place it at the path referenced in the notebook
2. Download `taxi+_zone_lookup.csv` and place it in the project root
3. Open and run `main.ipynb` end-to-end

---

## Key Takeaways

- **Manhattan dominates** the taxi network's steady state — 9 of the top 10 zones are in Manhattan
- **Newark Airport's #1 rank** is a methodological artifact worth investigating: its high self-loop probability (96% of trips stay at EWR) traps probability mass there
- The **sparse 265×265 transition matrix** with 40,256 non-zero edges reflects a densely connected but geographically constrained mobility graph
- Markov convergence is fast — 50 iterations is sufficient for a network of this size
- **Self-loops are removed** before building the transition matrix so zones like EWR (96% self-loop) don't artificially dominate the steady-state distribution
