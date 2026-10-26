# Responsible AI Readiness in Africa

**Gaps, patterns and opportunities for improvement: a GIRAI analysis of 39 African countries**

DSN 2026 AI Bootcamp In-House Hackathon, Data Analysis: African Responsible AI Challenge

This project uses the Global Index on Responsible AI (GIRAI) to show how ready 39 African countries are for responsible AI, highlight the biggest gaps, and recommend practical ways to close them.

---

## Table of contents

1. [Project overview](#project-overview)
2. [Key questions](#key-questions)
3. [Data](#data)
4. [Tools](#tools)
5. [Phase 1: Data cleaning](#phase-1-data-cleaning)
6. [Phase 2: Data modelling](#phase-2-data-modelling)
7. [Phase 3: DAX measures](#phase-3-dax-measures)
8. [Phase 4: Dashboard build](#phase-4-dashboard-build)
9. [Key findings](#key-findings)
10. [Recommendations](#recommendations)
11. [How to reproduce](#how-to-reproduce)
12. [Validation checks](#validation-checks)
13. [Limitations](#limitations)
14. [Team and credits](#team-and-credits)

---

## Project overview

African countries are building AI strategies and adopting AI across sectors, but their readiness to manage AI risks varies widely. This project analyses the GIRAI dataset to assess responsible AI readiness across African countries and delivers:

- An **interactive Power BI dashboard** (exported as PDF)
- A **3 to 5 page summary document** with the top insights and recommendations

The analysis covers **African countries only**. The dataset includes 39 of Africa's 54 countries, and every figure in the dashboard reflects those 39.

## Key questions

1. Which African countries show the strongest and weakest responsible AI readiness?
2. Which dimensions of responsible AI present the biggest gaps?
3. What factors appear to influence responsible AI performance?
4. How do countries compare across inclusion, ethics, skills, trust and public-sector AI?
5. What practical interventions could improve readiness?

## Data

**Source:** Global Index on Responsible AI (GIRAI) 2026, from <https://www.global-index.ai>. Check the data owner's terms of use before redistributing the raw files.

| Sheet | Rows | Content |
|---|---|---|
| `ranking_and_scores` | 135 countries | Global rank, GIRAI score, raw score, penalty multiplier, 5 dimension scores, 3 pillar scores (two-row header) |
| `pillar_scores_by_dimension` | 135 | 15 scores, one per pillar and dimension pair |
| `all_indicators` | 135 | 37 individual indicator scores |
| `country_classifications` | 135 | Region, subregion, income group, GDP per capita |

All scores are on a 0 to 100 scale. `ISO3` links the four sheets.

### Dimensions and pillars

- **5 dimensions (the topics):** Inclusion & Diversity, Ethics & Sustainability, Labour & Skills, Trust & Safety, AI in Public Service
- **3 pillars (the type of evidence):** AI Policy, CSO Engagement, Enabling Conditions

### How the GIRAI score is built (verified on all 135 countries)

1. Dimension score = 0.6 x AI Policy + 0.1 x CSO Engagement + 0.3 x Enabling Conditions
2. Raw GIRAI = simple average of the five dimension scores
3. Official GIRAI = raw GIRAI x `urai_penalty` (Kenya, Ghana and Uganda carry a 0.96 multiplier)
4. Each pillar score is the average of its five cells

AI Policy carries 60% of every dimension, so its link to GIRAI is partly built in.

## Tools

- Power BI Desktop (data modelling, DAX, dashboard)
- Power Query (cleaning and reshaping)
- Figma (dashboard page backgrounds, 1920 x 1080)
- Excel (source data)

---

## Phase 1: Data cleaning

Done in Power Query. Load with **Transform Data** (not Load), remove blank rows in every query (each sheet is padded to about 1,000 rows), and set data types last.

| Query | Key steps | Check |
|---|---|---|
| `Country` | Rename columns; replace subregion `South Africa` with `Southern Africa` (match entire cell); add `Is_Africa` flag | 135 rows, 39 in Africa |
| `Scores` | Promote header; rename the last 8 columns from the first data row; remove that row; drop `country` and `region`; set types | 135 rows, 13 columns |
| `Dimension_Scores` | Reference `Scores`; keep `ISO3` and the 5 dimensions; unpivot | 675 rows |
| `Pillar_Cells` | Unpivot 15 columns; strip ` (0-100)`; split on the **left-most** ` in `; relabel dimensions | 2,025 rows, 3 pillars, 5 dimensions |
| `Indicators` | Drop duplicate score columns; rename `iso3` to `ISO3` | 135 rows, 38 columns (`ISO3` + 37 indicators) |

**Pitfalls**

- The `ranking_and_scores` sheet has a two-row header: the dimension and pillar names sit in the first data row.
- Split `Pillar_Cells` at the left-most ` in ` only, because "AI Use in Public Service" also contains the word "in".
- The subregion "South Africa" collides with the country name, so rename it.
- Do not filter to Africa in Power Query. Use a report-level filter instead.

## Phase 2: Data modelling

A star-style model with `Country` as the central lookup table.

| Table | Rows | Role |
|---|---|---|
| `Country` | 135 | Lookup: region, subregion, income group |
| `Scores` | 135 | Wide table: GIRAI, 5 dimensions, 3 pillars |
| `Dimension_Scores` | 675 | Long table: country x dimension |
| `Pillar_Cells` | 2,025 | Long table: country x pillar x dimension |
| `Indicators` | 135 | 37 indicator scores |
| `Dim_Dimension`, `Dim_Pillar` | 5, 3 | Lookups with sort order |
| `Map_Metric`, `Enabling_Indicator` | 6, 4 | Disconnected selector tables |
| `Measures` | none | Holds all measures |

**Relationships** (all one-to-many, single direction):

```
Country[ISO3]             -> Scores[ISO3]
Country[ISO3]             -> Dimension_Scores[ISO3]
Country[ISO3]             -> Pillar_Cells[ISO3]
Country[ISO3]             -> Indicators[ISO3]
Dim_Dimension[Dimension]  -> Dimension_Scores[Dimension]
Dim_Dimension[Dimension]  -> Pillar_Cells[Dimension]
Dim_Pillar[Pillar]        -> Pillar_Cells[Pillar]
```

**Model settings**

- Sort `Dimension` and `Pillar` by their `Sort_Order` columns
- Set `Country[Country]` to the Country/Region data category
- Add a report-level filter: `Country[Region] = Africa`
- Add two calculated columns in `Country`: `Weakest Dimension` and `Strongest Dimension`

**Design choices**

- Long tables feed matrices, heatmaps and bar charts. The wide `Scores` table feeds maps, tables and the scatter plot.
- Disconnected tables drive selectors without filtering the rest of the model, so a 5 x 5 matrix never collapses when a metric is chosen.

## Phase 3: DAX measures

All measures live in the `Measures` table. Benchmarks use `ALL(Country)` and re-apply `Region = "Africa"`, so they stay fixed whatever the slicers.

```DAX
Country GIRAI = AVERAGE ( Scores[GIRAI] )

Africa GIRAI Avg =
CALCULATE ( AVERAGE ( Scores[GIRAI] ), ALL ( Country ), Country[Region] = "Africa" )

Gap to Avg = [Country GIRAI] - [Africa GIRAI Avg]

Africa Rank =
RANKX ( FILTER ( ALL ( Country ), Country[Region] = "Africa" ), [Country GIRAI], , DESC, DENSE )

Selected Metric Score =
SWITCH (
    SELECTEDVALUE ( Map_Metric[Metric], "Overall GIRAI" ),
    "Overall GIRAI", AVERAGE ( Scores[GIRAI] ),
    "Inclusion & Diversity", AVERAGE ( Scores[Inclusion & Diversity] ),
    "Ethics & Sustainability", AVERAGE ( Scores[Ethics & Sustainability] ),
    "Labour & Skills", AVERAGE ( Scores[Labour & Skills] ),
    "Trust & Safety", AVERAGE ( Scores[Trust & Safety] ),
    "AI in Public Service", AVERAGE ( Scores[AI in Public Service] )
)
```

The full measure set is in the project documentation. Groups:

| Group | Measures |
|---|---|
| Core | `Country GIRAI`, `Africa GIRAI Avg`, `Gap to Avg`, `Status`, `Bar Colour`, `Africa Rank`, `Dimension Score`, `Africa Dimension Avg`, `Pillar Cell Score` |
| Page 1 | `Highest/Lowest Country and Score`, `Countries Above/Below Avg`, `Above vs Below`, `Leaders and Laggards Flag` |
| Page 2 | `Selected Metric Score`, `Africa Selected Avg`, `Region Min`, `Region Max`, `Countries in Region` |
| Page 3 | `Best Country Score/Name`, `Weakest Count`, `Strongest Count`, `Indicator Africa Avg`, `Indicator r with GIRAI`, `r Enabling vs Policy` |

## Phase 4: Dashboard build

Every page is **1920 x 1080** so that all 39 countries fit without scrolling in the PDF export. Backgrounds are designed in Figma and set via Format page > Canvas background.

| Page | What it shows |
|---|---|
| **Performance** | KPI cards (African average, highest, lowest, above vs below), country ranking bar with average line, performance table |
| **Geography & Regions** | Filled map of GIRAI scores with a metric selector, regional bar chart, subregion x dimension matrix, weakest-dimension map |
| **Dimensions & Drivers** | Dimension comparison, weakest vs strongest counts, country x dimension heatmap, pillar bar, enabling-factor and policy analysis |
| **Country Explorer** | Table of all 39 countries, switchable between dimension and pillar views (bookmarks) |
| **Country Profile** | Drill-through page for a single country |
| **Tooltip** (hidden) | Pop-up with rank, score, strongest and weakest dimension |

**Interactivity:** subregion and country slicers (synced), cross-filtering, drill-through, tooltips, and bookmark view switching.

**Map note:** the Filled map has no colour saturation field. Colour it through Format > Colors > fx > Gradient on `Selected Metric Score`. Use `ISO3` as the location if any country stays grey.


## Key findings

| Finding | Evidence |
|---|---|
| Readiness is low and unequal | African average 21.8; 18 countries above, 21 below; top five average 40.4 against 7.5 for the bottom five |
| Strongest and weakest | Nigeria (45.9), Egypt, Kenya, Ghana, Benin; South Sudan (4.0), Chad, Burundi, Congo, Central African Republic |
| Central Africa trails | 9.9 average, against 27.1 in North Africa |
| Biggest dimension gap | Inclusion & Diversity averages 16.7 and is the weakest dimension in 18 countries |
| Policy, not foundations | AI Policy 13.7 and CSO Engagement 9.3, against Enabling Conditions 42.5 |
| Cybersecurity matters most | Cybersecurity has the strongest link to GIRAI (r = 0.70), ahead of rule of law, digital skills and equality |
| Foundations do not guarantee policy | Enabling Conditions and AI Policy correlate at only r = 0.34 |

Correlations show association, not causation.

## Recommendations

1. **Make national AI policy the first priority**, including rules for public-sector AI.
2. **Invest in cybersecurity, rule of law and digital skills**, the enabling factors most linked to performance.
3. **Build inclusion safeguards** on gender, language and non-discrimination.
4. **Strengthen civil society** and turn existing activity into formal policy dialogue.
5. **Organise regional peer learning**, with targeted support for Central Africa.

## How to reproduce

1. Download the GIRAI data files and read the dictionary.
2. Open Power BI Desktop and load the four sheets with **Transform Data**.
3. Clean each query as in [Phase 1](#phase-1-data-cleaning), checking row counts.
4. Create `Dim_Dimension`, `Dim_Pillar`, `Map_Metric`, `Enabling_Indicator` and `Measures` using Enter Data.
5. Close and apply, then build the relationships in [Phase 2](#phase-2-data-modelling).
6. Add the report-level filter `Country[Region] = Africa`.
7. Add the calculated columns and measures from [Phase 3](#phase-3-dax-measures).
8. Set each page to 1920 x 1080, add the backgrounds, and build the pages in order.
9. Build the Country Profile (drill-through) and Tooltip pages, and test with Nigeria, Kenya and South Sudan.
10. Reset the slicers and export to PDF (File > Export > PDF).

## Validation checks

| Check | Expected |
|---|---|
| African average GIRAI | 21.79 |
| Countries covered | 39 |
| Above / below average | 18 / 21 |
| Highest / lowest | Nigeria 45.93 / South Sudan 4.04 |
| Row counts | Country 135, Scores 135, Dimension_Scores 675, Pillar_Cells 2,025 |
| Subregion averages | Central 9.9, East 21.0, North 27.1, Southern 23.7, West 24.8 |
| Pillar averages | AI Policy 13.7, CSO 9.3, Enabling Conditions 42.5 |
| Nigeria drill-through | GIRAI 45.9, rank 1, strongest Trust & Safety, weakest AI in Public Service |

## Limitations

- The index covers 39 of Africa's 54 countries, so findings do not describe the whole continent.
- Central Africa (5), North Africa (5) and Southern Africa (4) are small samples.
- Correlations are not proof of cause, and AI Policy's 60% weight makes its link to GIRAI partly built in.
- The data is a single snapshot with no trend over time.


## Team and credits

- **Team:** [The Bias Busters]
- **Members:** [Bhadmus Mariam, Oluwafemi Oyediwura]
- **Data:** Global Index on Responsible AI (GIRAI)
- **Event:** DSN 2026 AI Bootcamp In-House Hackathon
