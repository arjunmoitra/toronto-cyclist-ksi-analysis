# Spatiotemporal Analysis of Cyclist KSI Collisions in Toronto

Where cyclist collisions cluster, whether income predicts risk, and whether Toronto's cycling network expansion is associated with fewer accidents, 2006–2023.

**Authors:** Sanika Rozario, Jose Barral, Arjun Moitra

[**Read the full rendered report →**](analysis.html) (self-contained HTML, open directly in a browser, no R required)

## Research questions

1. **Where** do serious cycling accidents (Killed or Seriously Injured, KSI) cluster spatially in Toronto?
2. **Do lower-income neighbourhoods** have higher accident rates after controlling for population?
3. **Has the cycling network** expansion been associated with reduced accident rates over time?

## Key findings

- **Spatial clustering:** A clear collision hotspot sits in the downtown core along major arterial roads, which account for ~65% of all cyclist KSI collisions. Raw correlation between infrastructure density and accident rate is *positive* (ρ = 0.357, p < 0.001) — a sign of confounding by cyclist volume, not infrastructure causing harm.
- **Income:** A borderline negative relationship between income and accident rate (p ≈ 0.096) disappears once a BYM spatial model accounts for neighbourhood adjacency (coefficient 0.023, 95% CI −0.122 to 0.167). Location, and the traffic environment it implies, predicts risk better than income does.
- **Time trend:** Collisions show a strong seasonal peak in June and a declining trend since ~2013, continuing (per an INLA forecast) through 2027 — temporally consistent with, though not proof of, the effect of cycling network expansion after Vision Zero (2016).
- **Deeper analysis:** A follow-up exposure-adjusted model (bike counter + Bike Share Toronto ridership data as a real exposure measure, at 1 km hex-grid resolution) shows the apparent infrastructure effect above mostly disappears once actual cycling volume is controlled for directly, while arterial road classification keeps an independent, significant association with collision counts.

Full discussion, limitations, and the extended exposure/robustness/before-after analysis are in the report itself.

## Methods

- **Spatial:** kernel density estimation (`spatstat`), population-adjusted hotspot mapping, BYM (Besag–York–Mollié) conditional autoregressive spatial model at the neighbourhood level and a finer 1 km hex-grid level (`INLA`, `spdep`)
- **Temporal:** seasonal decomposition and INLA-based forecasting of monthly collision counts
- **Causal-adjacent:** before/after corridor study around three protected-infrastructure installations, with citywide trend as baseline comparator
- **Exposure adjustment:** bike counter and Bike Share Toronto trip data used as independent cycling-volume proxies to de-confound the infrastructure–accident relationship

## Data sources

| Dataset | Source | Access |
|---|---|---|
| Cyclist KSI collisions (2006–2023) | Toronto Police Service ArcGIS FeatureServer | Live API pull, cached on first render |
| Neighbourhood boundaries (158 polygons) | City of Toronto Open Data (CKAN) | Live API pull, cached on first render |
| Cycling network (type + install year) | City of Toronto Open Data (CKAN) | Live API pull, cached on first render |
| Median after-tax household income, 2006/2016/2021 | Statistics Canada Census, via `cancensus` | Live API pull, requires free API key |
| Bike Share Toronto trip data (2023: 11.4M trips; 2019/2022 monthly totals) | City of Toronto Open Data / GBFS | Pre-aggregated CSVs committed to this repo (see below) |
| Road classification, ATR bike counter locations/counts | City of Toronto Open Data | Pre-aggregated CSVs/GeoPackage committed to this repo |

The three primary geographic layers (collisions, neighbourhoods, cycling network) and the census income data are fetched live from public APIs the first time the report renders and cached locally afterward — no manual download step is needed. The Bike Share, counter, and road-class files are the reduced output of a one-time aggregation against much larger public source files (documented in [LOG.md](LOG.md)) and are committed directly since re-running that aggregation on every render would add minutes for no change in output.

## Reproducing the report

Requirements:

- R (4.2+) and [Quarto](https://quarto.org)
- CRAN packages: `sf`, `terra`, `tidyverse`, `tmap` (v4 syntax), `cancensus`, `httr`, `spatstat`, `spdep`, `ggplot2`, `lubridate`, `gridExtra`, `scales`, `RColorBrewer`
- `INLA`, installed from its own repository:
  ```r
  install.packages("INLA", repos = c(getOption("repos"), INLA = "https://inla.r-inla-download.org/R/stable"), dep = TRUE)
  ```
- A free Statistics Canada `cancensus` API key:
  ```r
  cancensus::set_cancensus_api_key("YOUR_KEY_HERE", install = TRUE)
  ```
  Get one at [censusmapper.ca](https://censusmapper.ca/users/sign_up).

Then, from a clone of this repo:

```bash
quarto render analysis.qmd
```

The first render downloads the live-sourced datasets (a few minutes depending on connection) and caches them; later re-renders reuse the cache and only re-run the models and figures.

## Repository contents

```
analysis.qmd    Full source: data loading, cleaning, models, figures, and prose
analysis.html   Rendered report (self-contained, viewable standalone)
LOG.md          Implementation log — feasibility checks, decisions, and corrections
                made while building the exposure-adjusted / before-after analysis
styles.css      Report theme
*.csv, *.gpkg   Pre-aggregated data files (see Data sources above)
```

## Limitations

- The KSI dataset only captures serious/fatal collisions; minor incidents aren't recorded.
- The headline models don't control for cycling exposure (volume); busier routes see more accidents regardless of safety, which likely drives the counter-intuitive positive infrastructure–accident correlation in Q1 (addressed directly in the deeper exposure-adjusted analysis).
- Census income is a single 2021 snapshot for the headline model and may not reflect trends across 2006–2023 (a multi-year 2006/2016/2021 comparison is included as a stretch analysis).
- The BYM model treats neighbourhoods as the spatial unit, while risk plausibly operates at the level of individual intersections.
- Correlation between infrastructure and accidents does not imply causation.
