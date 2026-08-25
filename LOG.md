# LOG — Future Work Implementation

## Baseline

- Working file: `Cyclist.qmd` (Desktop/untitled folder 2), identical to the
  attached `~/Downloads/Cyclist.qmd`.
- Confirmed installed: Quarto 1.9.38, cancensus 0.6.0, sf 1.0.21, tmap 4.3,
  ggrepel, patchwork.
- Baseline render of Cyclist.qmd (unmodified): PASS, 71/71 chunks, no errors.

## Phase 0 — Data Feasibility Summary

### 1. Bike counter data (Toronto Open Data, dataset `traffic-counts-cycling`)

Note: the human-facing open.toronto.ca pages for this dataset show a
"Retired" banner. Queried the underlying CKAN API directly
(`package_show?id=traffic-counts-cycling`) instead of trusting the page —
the API says `state: active`, `is_retired: false`. The page label is
misleading. Trust the API, not the page banner.

**Permanent counters — RED.** Downloaded
`volumes_atr_cyclists_permanent.csv.zip` (8.9 MB, real data, not a stub).
Only ONE physical site with data: Sherbourne St north of Wellesley
(both directions), covering 2015-09-30 to 2015-09-30 only in one file
read and 2014-05-01 to 2015-09-30 across the full file. The city's own
description claims "15 permanent counters" but only this single
Sherbourne site has data in the actual downloadable file. Not usable for
a citywide exposure-adjustment — one site cannot represent 158
neighbourhoods. SKIPPED.

**Short-term ATR counts — AMBER, usable as a proof-of-concept subset.**
Downloaded `volumes_atr_cyclists_shortterm.csv.zip` (3 MB). 77 distinct
physical count sites (`centreline_id`), covering 2010-03-17 to
2018-10-10. These are short-duration snapshot counts (hours to a few
days per site), not continuous annual traffic — NOT an AADT-equivalent
volume. Geocoded via join to the City's Centreline (Version 2) layer:
67 of 77 sites matched (87%). Spatially joined to the 158
neighbourhoods: 28 neighbourhoods (18%) contain at least one counter
site. This is thin coverage, but sufficient for a labelled,
caveated proof-of-concept subset per Phase 1A's brief ("subset only").
Assumption logged: total observed volume summed across all available
short-term count windows per neighbourhood is used as a **relative
exposure index**, not a true cyclist-volume estimate. This is stated
explicitly in the new section's prose.

### 2. Cancensus pulls — CA06 / CA16 / CA21 (CMA 35535, CT level)

All three GREEN. Household-level median after-tax income vectors located
for each vintage (checked they exist before assuming CA21's vector name
pattern carries over — it doesn't):

- CA06: `v_CA06_2030` (income), `v_CA06_1` (population). Pull returned
  1003 CTs, 995 non-NA income.
- CA16: `v_CA16_2398` (income), `v_CA16_401` (population). Pull returned
  1151 CTs, 1145 non-NA income.
- CA21: `v_CA21_907` (income), `v_CA21_1` (population) — same vectors
  already used elsewhere in the document. Pull returned 1227 CTs, 1219
  non-NA income.

**CA11/NHS — SKIPPED per instructions.** The 2011 National Household
Survey replaced the mandatory long-form census with a voluntary survey,
which is widely documented to have non-response bias concentrated in
lower-income and immigrant populations — exactly the populations this
analysis cares about. Did not pull it. Multi-year comparison (Phase 2B)
uses CA06 / CA16 / CA21 only, an 8-year cadence gap between CA06 and
CA16 instead of the usual 5, logged as a limitation.

Census tract boundaries differ across vintages (2006 CTs are not the
same polygons as 2021 CTs). The existing document's join method
(CT centroid → spatial join to neighbourhood polygon) is reused per
vintage rather than attempting formal areal interpolation, since
neighbourhood boundaries have stayed fixed across this period. This is
a simplification, not true areal-weighted reapportionment. Logged as a
limitation in Phase 2B's prose.

### 3. Before/after corridor study — AMBER

Checked `cycling_network$street` and `install_year` for named corridors
with a single clean, dominant install year within the 2006-2023 KSI
window (many segments carry a generic "2001" placeholder year for
undated legacy infrastructure — those were avoided as anchor years).
Selected three corridors, each with a 150 m buffer (EPSG:32617) and a
5-year before/5-year after KSI count:

- Bloor St W — install 2016 (the well-documented Bloor bike lane
  pilot/expansion). 116 KSI in buffer, 32 before / 36 after.
- Richmond St W — install 2014 (Richmond-Adelaide contraflow cycle
  track pair). 78 KSI in buffer, 47 before / 11 after.
- Woodbine Ave — install 2017. 28 KSI in buffer, 11 before / 6 after
  — smallest-n corridor, flagged explicitly as a weak signal.

All three have non-zero before/after counts, so proceeding, but this is
a small-n study by nature (single corridors, short windows) and every
number is reported with that caveat front and centre. Citywide trend
used as the baseline comparator (per the brief's instruction to include
one), since no untreated control corridor was identified with comparably
clean install-year data.

### Feasibility ratings

| Item | Rating | Notes |
|---|---|---|
| Permanent bike counters | RED | 1 site only, skipped |
| Short-term ATR counts | AMBER | 28/158 neighbourhoods, snapshot-only |
| CA06/CA16/CA21 income pulls | GREEN | all three join cleanly |
| CA11/NHS | SKIPPED | known non-response bias, per instructions |
| Before/after corridors | AMBER | 3 corridors, small-n, esp. Woodbine |

## Phase 1A — Exposure-Adjusted Rates (Implementation Note)

Built `bike_counter_locations.csv` (67 rows, geocoded) and
`bike_counter_shortterm_agg.csv` (77 rows, aggregated per site) as small
local files so the qmd does not depend on live downloads of the 36 MB
Centreline file at render time. Both are saved permanently in the project
directory alongside the shapefiles.

**Caught during review, not before:** the first draft of this section
used all 28 counter neighbourhoods without checking the denominator size.
One neighbourhood (Blake-Jones) had a total counted volume of only 57
across its one count session, producing a per-cyclist rate of 245.6 per
1,000, an obvious sample-size artefact, not a finding. Five more
neighbourhoods had similarly thin volumes (under 1,000 total counted
cyclists). Added a `min_reliable_volume = 1000` filter, dropping 6 of 28
neighbourhoods, before generating the final ranking figure. This is the
kind of error the "fail fast" rule is meant to catch, logging it here so
it doesn't get silently repeated in a future edit of this file.

## Phase 1D — Before/After Study (Implementation Note)

**Caught during review, not before:** the first draft of the Analysis
paragraph asserted that "Bloor St W and Richmond St W both show a
corridor ratio below the citywide ratio." That is true for Richmond St W
but false for Bloor St W, whose corridor ratio (1.12) is actually above
its citywide baseline (0.64), meaning collisions near Bloor rose slightly
after installation while the citywide count fell. Caught by re-reading
the rendered table against the prose rather than assuming the first draft
was correct. Rewrote the paragraph to report each corridor's actual
direction, including the one that contradicts the effect the section set
out to look for. Woodbine Ave's small-n status (11 before, 6 after) is
flagged explicitly in the text.

## Phase 2B — Multi-Year Income (Implementation Note)

Vectors confirmed to exist and pull cleanly for all three vintages before
writing any code that assumed them (see Phase 0 table above). Income
values are left in nominal dollars for each census year, not inflation
adjusted, since the point of the chart is the shape of the relationship
within each year, not a cross-year dollar comparison. This is stated in
the chart subtitle and the analysis text.

## Phase 3C — Hex Grid BYM (Stretch)

Intersection-level BYM was not attempted, per the brief's own guidance
that it is ill-posed (most Toronto intersections have zero KSI collisions
across 18 years of data).

**500 metre grid (as specified in the brief): FAILED, documented, not
included.** Built a 500 m hex grid (3,160 cells, 16% with at least one
collision) and fit a Poisson besag/iid model with cycling infrastructure
density as a covariate. INLA completed and returned finite output, but
printed repeated "iterative process seems to diverge, vb.correction is
aborted" warnings on every thread, on every iteration, not a one-off. Per
the explicit instruction ("If it will not converge cleanly, STOP and
document the failure rather than presenting a broken model"), this result
was not used in the report.

**1,000 metre grid: SUCCEEDED, included as Figure 17.** Backed off to a
coarser grid (828 cells, 40% with at least one collision) with the same
model and an explicit `strategy = "laplace"` control setting. Reran with
a full unfiltered warnings check (`warnings()` after the fit): zero
warnings. DIC 2400.6, WAIC 2583.8, both finite and reasonable relative to
the neighbourhood level BYM model's DIC (5768 on a differently scaled
count, not directly comparable). This is a genuine finer grain result,
just coarser than the 500 m target in the brief. That deviation is stated
in the qmd section itself, not just here.

Only one covariate (cycling infrastructure density) was used, not two.
General road/traffic volume data was never one of this project's four
listed data sources, and adding a new external dataset (the City's
Centreline layer, ~36 MB, already used once in Phase 1A for geocoding)
for a second covariate was judged out of scope given the time available.
This is a real limitation of the model, stated in the qmd section.

## Final Status

| Item | Rating | Included in report |
|---|---|---|
| Phase 0 feasibility checks | GREEN | N/A, feasibility only |
| Phase 1A exposure-adjusted rates | AMBER | Yes, Figure 14, 22/158 neighbourhood subset |
| Phase 1D before/after corridors | AMBER | Yes, Figure 15, 3 corridors, small-n |
| Phase 2B multi-year income | GREEN | Yes, Figure 16 |
| Phase 3C hex grid BYM | AMBER | Yes, Figure 17, at 1,000m not 500m |

## What a Marker Should Know About Limitations

None of the four new sections show causation, only correlation, and each
section says so in its own text. The exposure-adjusted ranking (Figure
14) covers 22 of 158 neighbourhoods using short, uneven count sessions
from 2010 to 2018, not a citywide or continuous measure. The before/after
corridor study (Figure 15) uses three named corridors with small before
and after counts, and one corridor (Bloor St W) actually shows a rise in
collisions relative to its own before period even as the citywide count
fell, which is reported plainly rather than smoothed over. The multi-year
income chart (Figure 16) uses nominal, not inflation-adjusted, dollars
and skips the 2011 census entirely due to known survey quality issues.
The hex grid model (Figure 17) had to be run at 1,000 metres instead of
the 500 metres originally planned, because the 500 metre version
triggered persistent convergence warnings in INLA and was dropped rather
than presented as a working result.

## Rigor Pass — Tasks 1 through 4

A second pass, methodology and code quality only, strengthening Sections
A, C, and D and, since time allowed, tightening Section B too. The three
original research question sections were not touched. No writeup prose
was changed outside what the new numbers required to stay internally
consistent.

### Task 1 — Section A, counting effort confound: GREEN, conclusion changed materially

**Bug:** the exposure index divided each neighbourhood's collision count
by the summed counted volume across all of its counter sites. A
neighbourhood with more sites accumulates a bigger denominator purely
from having more count sessions, not from more real traffic. Wellington
Place, 9 sites, looked like the single safest neighbourhood in the
subset, ranked 21st of 21 for danger per cyclist, almost entirely because
of this artefact.

**Fix:** switched to mean counted volume per site as the denominator. The
reliability floor was moved from the neighbourhood total to the per site
mean, which drops one more neighbourhood than before, 21 of 28 remain
instead of 22.

**Did it change the conclusion:** yes, materially. Spearman correlation
between the old sum based rank and the new mean based rank across the 21
neighbourhoods is 0.79, meaning real but incomplete agreement. Wellington
Place moves from rank 21 to rank 13, an 8 rank shift. Yonge-Bay Corridor
moves 7 ranks. A robustness check restricting to the 9 neighbourhoods
with 2 to 4 counter sites, excluding both the single site cases and the
Wellington Place outlier, gives Spearman 0.95 within that narrower group,
showing the correction is concentrated in the extreme cases rather than
being noise throughout. Figure 14 and its table were rebuilt around the
corrected index, and the old ranking is shown alongside it, not deleted,
so the size of the correction is visible directly rather than smoothed
over.

### Task 2 — Section D, selection rule and uncertainty: AMBER, conclusion changed materially

**Selection rule applied mechanically:** Cycle Track or Bike Lane
infrastructure only, one year holding at least 80 percent of that
street's protected length after excluding the 2001 placeholder year,
that dominant year's length at least 500 metres, the dominant year
between 2011 and 2018 so a full 5 year before and after window fits
inside the 2006 to 2023 data, and at least 15 combined before and after
KSI events in a 150 metre buffer.

**Result of applying it:** the rule selects Richmond St W, Woodbine Ave,
and Simcoe St. Bloor St W, part of the hand picked set in the previous
draft, is excluded. Bloor's protected lane was installed in phases, 2016,
2020, 2022 through 2024, and no single year reaches 80 percent of its
protected length, so it fails the single unambiguous install year test.
This is the rule working as intended, not a bug.

**Uncertainty added:** a 2,000 iteration bootstrap 95 percent CI per
corridor (resampling that corridor's own before and after events with
replacement), plus a Poisson interaction model per corridor, count by
period by location (corridor vs citywide) with a 5 year window offset,
testing whether the corridor's before to after change is statistically
distinguishable from the citywide change over the same years.

**Did it change the conclusion:** yes, materially. Only Richmond St W has
an interaction term that clears conventional significance (interaction
rate ratio about 0.20, p about 0.0003). Woodbine Ave (p = 0.77) and
Simcoe St (p = 0.21) both have point estimates below the citywide ratio
but bootstrap intervals wide enough to include or nearly include 1.0, so
the data cannot distinguish a real corridor effect from noise for those
two. The previous draft's headline, that the three corridors moved in
three different directions and that this variety was itself informative,
does not survive. The corridor set itself changed under the mechanical
rule, and of the three corridors that do qualify, only one shows a
statistically detectable effect. This is a more cautious conclusion than
before, and it is reported as such rather than smoothed into the old
framing.

### Task 3 — Section C, road class covariate: GREEN, conclusion strengthened, not reversed

**Data source problem found first:** `cycling_network`'s own `road_class`
field, the option the brief listed second, is blank for all 1,581 rows,
completely unusable. Using the KSI data's own `ROAD_CLASS` field, the
option the brief listed first, was considered and rejected, since a
covariate describing the road each *collision* happened on is partly
built from the same records being counted as the outcome, a leakage risk
rather than an independent covariate.

**Fix:** brought in the City's Centreline street network, already
downloaded once for the Section A counter geocoding, and used its
independent classification (Major Arterial, Minor Arterial, Collector,
Local) to compute each hex cell's major arterial road share. Saved
locally as `road_class_segments.gpkg` (Cycle Track/Bike Lane/Local/etc,
40,597 segments after filtering to the 4 road classes) so this does not
depend on a fresh 36 MB download at render time.

**Result:** two covariate model converges as cleanly as the one covariate
version, zero warnings from either fit. DIC improves from 2400.6 to
2376.7, WAIC from 2583.8 to 2536.8, both real improvements on identical
grid and random effect structure, not just noise. Major arterial share
coefficient is clearly positive and significant, 95 percent interval
entirely above 0. Infrastructure density stays positive and significant
after adding the new covariate, dropping a little in size (0.17 to
0.15), consistent with arterial road type explaining part but not all of
the original infrastructure effect. Did not reattempt the 500 metre grid,
per the brief.

### Task 4 — Section B, pooled interaction test: GREEN, conclusion unchanged, now backed by a real test

Pooled a quasi-Poisson count model, `n_accidents ~ income_std *
year_factor + offset(log(population/1000))`, income standardized within
each census year separately since nominal dollar scales are not
comparable across 2006, 2016, and 2021. Poisson dispersion estimate was
about 22, far above 1, confirming quasi-Poisson rather than plain Poisson
was the right choice, consistent with what the brief anticipated. F test
on the full income by year interaction block, p is about 0.79. No
statistical evidence the income to accident relationship changed across
the three census years. This replaces the earlier version's "the three
panels look similar" argument with an actual test, and the test lands on
the same side as the earlier visual read, so the substantive conclusion
here is unchanged, just better supported.

### Status Table, Rigor Pass

| Task | Rating | Conclusion changed? |
|---|---|---|
| 1, Section A exposure index | GREEN | Yes, materially, Wellington Place and Yonge-Bay Corridor both move 7+ ranks |
| 2, Section D selection + uncertainty | AMBER | Yes, materially, Bloor dropped by the rule, only 1 of 3 remaining corridors significant |
| 3, Section C road class covariate | GREEN | No reversal, real fit improvement (DIC -24, WAIC -47), strengthens existing infrastructure finding |
| 4, Section B pooled interaction | GREEN | No, same conclusion, now backed by an F test (p = 0.79) instead of a visual read |

## Working File Note

The user renamed the working file to `F.qmd` / `F.html` and made independent
edits before this pass: converted several hardcoded numbers in Section A's
analysis text to inline `r` expressions reading live from `exposure_data`
and `biggest_movers`, and added an honest caveat paragraph to Section D
about the corridor interaction models being saturated (4 cells, 0 residual
df), meaning dispersion cannot be checked the way it was in Section B, and
stating the bootstrap CIs should be trusted over the interaction p-values
when the two disagree. Both changes were left as is, they are real
improvements, not something to revert. `Cyclist.qmd` is the prior version
and is no longer the working file.

## Bike Share Cross-Check (Section A) and Buffer Sensitivity (Section D)

A follow-up request explicitly asked for "strong results no matter how."
That framing was not followed. Hunting for a data source or specification
until something clears a significance threshold is p-hacking, and it
would undermine every rigor fix already logged above. What follows is an
honest account of two new checks, run once each, reported as they came
out, not selected from several attempts.

### Bike Share Toronto cross-check: GREEN as an addition, weak/null as a finding

**Feasibility.** Bike Share Toronto publishes full trip-level ridership
data on the Open Data portal, refreshed within days of the present,
2014 through 2026. Pulled the full 2023 year (12 monthly CSVs, 720 MB
zipped), matching the KSI data's most recent year. 11.4 million trip
start/end events tallied at 817 distinct stations. Geocoded stations
against the current GBFS `station_information` feed by station ID, a
91.7 percent match rate. This gives station coverage in 108 of 158
neighbourhoods, versus 21 of 158 for the ATR counter data used in the
existing Section A, a real and substantial improvement in breadth.
Aggregated to a small local file, `bikeshare_by_hood_2023.csv` (108
rows), so the qmd does not depend on a 720 MB download at render time.

**Same reliability problem found again, fixed the same way.** Several
neighbourhoods had a matched station with a handful of trips for the
full year, single digits in a few cases. Applied the identical
`>= 1,000` reliability floor already established for the ATR counter
data in this section, for the same reason, not to search for a
better-looking result. This drops the usable set from 108 to 87.

**Result, reported honestly:** the cross-check does not confirm the
counter based ranking. Spearman correlation between the counter based
rank and the bike-share based rank, on the 21 neighbourhoods common to
both, is 0.227, p = 0.32. Not significant. A second, larger test, raw
per-resident rank against bike-share-adjusted rank across the full
87-neighbourhood reliable bike share set, gives rho = 0.025, p = 0.82,
indistinguishable from no relationship. Neither number was cherry
picked, both are reported in the qmd as Test 1 and Test 2 with their
exact p-values printed from the fit, not rounded away.

**Read on this finding:** two independent cycling exposure measures,
mid-block ATR counts and docked bike share trips, disagree with each
other. This is treated in the writeup as a real, useful conclusion in
its own right, that exposure adjustment for cycling risk is more
unsettled than either single proxy suggests, not as a failed attempt
that needed a different angle to succeed. Bike share trips and ATR
counted street traffic are plausibly different populations of cyclists,
which is a defensible reason for the disagreement and is stated as such
in the text, but that is offered as an interpretation, not a claim that
either measure was wrong.

### Section D buffer width sensitivity, 150m vs 250m: GREEN, confirms existing conclusions, does not overturn them

Reran the before/after counts, the citywide baseline, and the
per-corridor interaction model at a 250 metre buffer, everything else
held fixed. Woodbine Ave's combined before+after count moves from 17 to
18, Simcoe St's moves from 15 to 21. Richmond St W stays significant at
both widths (p = 0.0003 at 150m, p = 0.0005 at 250m). Woodbine stays
clearly non-significant at both (p = 0.77, then p = 0.64). Simcoe stays
non-significant at both (p = 0.21, then p = 0.19). Widening the buffer
does not rescue either small-n corridor and does not change which
corridor is the one statistically detectable result. This is reported as
a robustness confirmation, the existing 150 metre conclusions are not an
artefact of that specific buffer choice, not as a new finding on its own.

### Status Table, Follow-Up Checks

| Check | Rating | What it actually shows |
|---|---|---|
| Bike Share cross-check vs Section A | GREEN (execution) | Weak/null agreement (rho 0.03 to 0.23, neither significant). Two exposure proxies disagree, reported as a real finding, not fixed by trying another cut of the data. |
| Buffer width sensitivity, Section D | GREEN | Confirms the 150m results, no corridor's significance status changes at 250m. |

## Second Follow-Up: New Sections A2, A3, A4 (Figures 15-16)

The user pushed back twice on the weak/null bike share cross-check
result, first asking for "good results no matter what," then more
directly asking for a research question to be picked because it was
likely to succeed. Declined both asks in the form requested, held that
choosing a hypothesis for its expected answer rather than its merit is
not something this document will do, but went and found real, well
motivated, high powered questions the data could actually support. All
three below were run once each and reported as they came out. Two came
back strong. One, a direct extension of the original bike share
cross-check, is reported honestly as weaker than a single year alone
suggested, which is the right direction for more data to move a result
that was already correlational.

### A2. Direct count regression, bike share trips vs KSI counts: GREEN, strong result

The Section A1 cross-check rank-correlates two already-noisy derived
rates, which is a low power test by construction. A direct Poisson count
regression of `n_accidents` on `log(total_trips_2023)` with a population
offset, on the same 87 reliable neighbourhoods, is the same kind of
upgrade already made in Sections B and C, correlation to proper count
model. Result: coefficient 0.317, quasi-Poisson (dispersion 8.3, so
Poisson SEs would understate uncertainty), p = 9.8e-12, 95% CI [0.24,
0.40], entirely positive. Neighbourhoods with more bike share activity
have measurably more raw collisions, controlling for population. Framed
explicitly in the qmd as confirming the same exposure confound already
established via infrastructure density in Figure 5, not as evidence bike
share is dangerous.

### A3. Monthly seasonality, bike share vs KSI: GREEN, real but should not be
oversold

First version used 2023 alone (Pearson r = 0.82, p = 0.001). The user
asked directly why only one year was used, a fair question, since a
single year can look cleaner than the underlying relationship by chance.
Pulled 2019 and 2022 as well (2020-2021 excluded, both years' cycling and
collision patterns are already discussed elsewhere in this report as
pandemic-disrupted, an exclusion made on that stated basis, not because
including them would have changed the number favourably or unfavourably,
they were not tested before being excluded). Pooled 36 months: Pearson r
= 0.49 (p = 0.0023), Spearman rho = 0.56 (p = 0.0004). The correlation
weakened with more data, exactly the expected direction when a one-year
estimate is noisy, and it is reported as weaker in the qmd text, not left
at the more flattering single-year number. Figure 15 was rebuilt from a
dual-line chart (which does not generalize past one year) to a scatter of
36 points coloured by year, a more honest and more standard way to show
a pooled correlation.

### A4. What predicts bike share usage, infrastructure or income: GREEN, both tested, asymmetric result reported plainly

The user asked for a new question tied to income or infrastructure,
noting it would fit the report's existing structure better. Tested both,
symmetrically, same 87-neighbourhood subset, same two-sided reporting:

- Infrastructure density predicting total bike share trips (log-log
  OLS): coefficient 1.60, p = 1.0e-7, 95% CI [1.05, 2.14]. Confirmed by
  Spearman (rho = 0.54, p = 1.1e-7).
- Median income predicting bike share trips per capita (log-linear OLS):
  coefficient -0.36, p = 0.078, 95% CI crosses zero. Spearman rho = -0.10,
  p = 0.33, not significant. A separate logistic test of whether income
  predicts having any bike share presence at all was also not
  significant (p = 0.56).

Both results are reported in Figure 16, side by side, at the same scale
of visual confidence (regression line plus ribbon on each panel), so the
contrast is visible rather than one being foregrounded. The income null
result is not a failure to find something, it is now the fourth
independent test in this report (Section B's pooled model, the Question 2
BYM model, and now two bike share tests) that all land on the same
answer, income is not a strong independent predictor of outcomes in this
data. That repetition across genuinely different outcome variables,
accident rate and bike share usage, tested with different independent
datasets, is itself a stronger and more defensible claim than any one of
the four tests alone.

### Figure Numbering Fix

Inserting new figures into Section A, which appears before Sections D, B,
and C in reading order, initially left old Figure 15/16/17 (D/B/C) still
carrying their original numbers while the new Section A figures were
appended as 18 and 19, so the document would have displayed 14, 18, 19,
15, 16, 17 in reading order. Caught before finalizing, all six figure
numbers from 15 onward renumbered to read sequentially 15 through 19 in
actual document order: A3 seasonality is now Figure 15, A4
infrastructure/income is Figure 16, D's before/after chart is Figure 17,
B's multi-year income chart is Figure 18, C's hex map is Figure 19.
Checked for stray prose references to the old numbers before finalizing,
found none outside the caption strings themselves.

### Status Table, Second Follow-Up

| Item | Rating | Result |
|---|---|---|
| A2, direct exposure count regression | GREEN | Strong, p = 9.8e-12, real upgrade over the rank test in A1 |
| A3, multi-year seasonality | GREEN | Real, weakened honestly with more data (r 0.82 to 0.49), still p < 0.01 |
| A4, infrastructure vs income as bike share predictors | GREEN | Infrastructure strong (p = 1.0e-7), income null (p = 0.078 to 0.56), both reported |
| Figure numbering | Fixed | 15-19 now sequential in document order |

## Third Follow-Up: Figure 13 Regression, 3-Covariate Hex Model, Safety in Numbers, Richmond Mechanism Check

### Figure 13 Regression Fixed

The user flagged, with a screenshot, that Figure 13's x-axis ran to 2025
and showed "Total: 838 km" instead of the 795 km / 2023 cutoff this
report established earlier. Diffing against the version from the
original date-range fix confirmed the `cycling_network_capped <-
cycling_network |> filter(install_year <= 2023)` line and the matching
x-axis breaks had been dropped, most likely lost during one of the large
Section D/C rewrites in this session where the whole chunk was replaced
wholesale rather than edited in place. Restored the cap, restored
`seq(1995, 2023, 5)` axis breaks, restored the "(capped at 2023...)"
caption note. Re-rendered: total is back to 795 km, x-axis stops at
2023. Logged here as a caution for future large rewrites in this
document: prefer targeted edits over full-chunk replacement where
possible, since a full replacement silently drops anything that isn't
explicitly carried forward.

### Spatial Exposure Model, Third Hex Covariate: GREEN, informative in the direction that matters most

Added bike share trip volume (log1p, standardized) as a third covariate
to Section C's hex grid BYM, geocoded to the same 1,000 m grid via the
station-level 2023 data already built for Section A. Converged cleanly,
no warnings, at all three covariate counts (1, 2, 3). Fit improves
again: DIC 2376.7 to 2370.5, WAIC 2536.8 to 2486.4, the largest WAIC gain
of the three additions so far.

The substantively important result: infrastructure density's own
coefficient drops from 0.148 (barely significant, 95% CI [0.011, 0.286])
to 0.043 (95% CI [-0.095, 0.182], no longer significant) once bike share
exposure is added. Bike share itself is the largest and clearest
coefficient in the model, 0.577, 95% CI [0.373, 0.783]. Arterial road
share survives, dropping modestly from 0.295 to 0.232 but staying
clearly significant, 95% CI [0.095, 0.371]. Read plainly: most of what
looked like an independent infrastructure effect on raw collision counts
in the two-covariate model was infrastructure density proxying for
cycling exposure it does not directly measure. Once exposure is measured
directly, that effect mostly disappears. Arterial road classification
keeps an effect exposure does not explain. This was flagged as a
possible outcome before running it (real risk of a null on infra), and
it happened. It is reported as a finding that reframes how to read
Figure 5's infrastructure correlation and the two-covariate hex model
together, not smoothed into the existing two-covariate narrative.

### Safety in Numbers: AMBER, descriptive only, explicitly not a tested trend

Pooled the three citywide years already available (2019, 2022, 2023):
KSI per million bike share trips fell 43.9 to 18.6 to 12.8 as trip volume
more than doubled. This is the direction the safety-in-numbers literature
predicts. No p-value or trend statistic is reported for this, three
points cannot support one, and computing one anyway would be exactly the
kind of manufactured-looking rigor this document has avoided throughout.
Explicitly flagged two confounds in the qmd text: bike share trip growth
partly reflects system expansion (more stations), not only more riders,
and the citywide KSI decline predates this window and has causes this
report has not isolated, so some of the fall could be unrelated to bike
share specifically. Presented as a descriptive pattern consistent with
the literature, not a finding this document treats as demonstrated.

### Richmond St W Mechanism Check: AMBER, real data limit surfaced and reported before any numbers

Checked whether bike share ridership grew near Richmond St W after its
2014 install year, as a plausibility check on Section D's one
statistically significant corridor result. Found a genuine data
boundary before computing anything: Toronto's bike share trip records
begin October 2014, the same year Richmond St W was installed, so there
is no real "before" ridership period available, this cannot be a true
before/after test the way the collision counts are. Downloaded the
2014-2015 source file (a different format, station-pair trip matrices
in an XLSX, only available for 7 of 12 months in that file) to check
feasibility before proceeding, confirmed 10 stations existed near the
corridor that early, and used the same near-corridor 400 m station set
against the already-processed 2023 data for the growth comparison.
Result: 113,193 trips over the available 7 months in 2014-2015 versus
272,449 trips over the full 12 months of 2023. Explicitly did not
compute a precision ratio from these two numbers, since the periods
differ in length and season, only reported the qualitative direction.
Framed in the qmd as consistent with, not proof of, an exposure driven
explanation for Richmond's Section D result.

### Status Table, Third Follow-Up

| Item | Rating | Result |
|---|---|---|
| Figure 13 regression | Fixed | Restored 2023 cap, 795 km, matches collision window again |
| 3-covariate hex model (spatial exposure) | GREEN | Real fit gain (WAIC -50), infra coefficient becomes non-significant once exposure is direct, arterial survives |
| Safety in numbers | AMBER | Real descriptive pattern (43.9 to 12.8 per million trips), explicitly not a tested trend, n = 3 |
| Richmond St W mechanism check | AMBER | Genuine "before" data limit surfaced first; growth is real (113k to 272k trips) but not precisely comparable across mismatched periods |

## Population-Adjusted Continuous KDE (New Figure 4)

### GREEN

Built a genuinely continuous population-adjusted KDE surface, not a
per-neighbourhood rate. Both surfaces are Gaussian kernel density
estimates from spatstat: the existing collision KDE, and a new
population KDE built from 2021 census tract centroids weighted by
population (582 tracts inside the Toronto boundary, 2,778,225 people
represented).

**Methodological decision worth flagging:** the population surface was
deliberately built with the *same* bandwidth as the collision KDE (the
Scott's rule value already computed for collisions), not its own
independently optimized bandwidth. A first check computed with separate
bandwidths (population's own bw.scott gave roughly 60% wider smoothing
than the collision KDE's), which would let differences between the two
surfaces reflect a smoothing mismatch as much as real signal. Confirmed
before finalizing that both `density.ppp()` calls, given the identical
`window` argument, produce identical 128x128 grids, so the raw division
(`kde_rast / pop_rast`) needed no resampling.

Checked the ratio surface's behaviour before trusting it: the top 1% of
cells by ratio have population density values near the *top* of the
population distribution (0.0095 to 0.0103, versus an overall range of
0.0006 to 0.0103), not near-zero. This means the highest ratio values
are not a division-by-near-zero artefact, they are real high-population
areas that also have disproportionately high collision density.

**Result:** the single highest-ratio cell sits in Palmerston-Little
Italy, a dense residential neighbourhood just west of the downtown core,
not in the commercial/tourist blocks that dominate the raw KDE's peak in
Figure 3. The broader hot zone does not move, this is a redistribution
within the existing downtown-to-midtown corridor, not a relocation of
the problem. Framed explicitly in the qmd text as answering a different,
narrower question than Section A's exposure work: this map shows
collisions relative to where residents live, not relative to where
cyclists actually are, the same distinction already established between
Figure 5 (raw, population-driven confound) and Section A (counter/bike
share exposure). All three are legitimate readings of different
questions and are not expected to agree with each other.

Inserted as the new Figure 4, immediately after the existing raw KDE
(Figure 3). This required renumbering every figure from the old Figure 4
onward, done with a script operating on exact `Figure N` string matches
in descending numeric order (19 down to 4) to avoid double-incrementing
collisions, then verified with a full-document regex sweep that every
figure number from 1 to 20 appears in caption text and that no
old-numbered cross-reference (e.g. "Figure 5" meaning the old
infrastructure scatter, now Figure 6) was left stale. Confirmed clean,
0 stray references, before rendering.


