# Dubai Real Estate Transactions — Key Findings & Recommendations

## Overview
This dashboard analyzes ~1.75 million Dubai Land Department transaction records (1966–2026), covering Sales, Mortgages, and Gifts across residential, commercial, and land property types.

## Key Findings

**Market Growth**
Transaction activity remained minimal through the 1990s, accelerating from the late 1990s onward alongside Dubai's real estate market opening to foreign ownership. Volume peaked in 2009, pulled back through 2010–2011, recovered steadily through 2019, dipped in 2020, and has surged to record highs in 2024 and 2025 (AED 682bn in Sales value for 2025 alone).

**Geographic Concentration**
Marsa Dubai leads both total sales value (AED 283.9bn) and transaction volume (138,024) among all areas. Burj Khalifa commands the highest price per sqm (AED 23,521) — nearly double the group average — while off-plan activity varies sharply by area, from 14% to over 68%, indicating distinct buyer profiles across neighborhoods.

**Property Composition**
Residential transactions dominate at AED 2.42T (65% of total value). Units lead by property type (AED 1.74T), followed by Land, Villa, and Building. Price scales sharply with unit size — 9-bedroom units average AED 92K/sqm, nearly 6x the 1-bedroom rate.

**Off-Plan vs. Existing**
Off-plan financing applies to 60% of Unit sales and 27% of Villa sales, while Land and Building transactions occur almost exclusively as existing-property sales (near 0% off-plan) — consistent with off-plan structures applying only to under-construction residential developments.

## Data Quality Notes
- Standardized missing/unknown values across all dimension keys using a consistent "-1 / Unknown" convention rather than leaving them blank, preserving analytical accuracy while flagging incomplete source records.
- Corrected cross-language contamination in the `property_usage` field (English text appearing in Arabic values and vice versa) by cross-referencing the reliable sibling column.
- Backfilled missing sub-type classification for Villa records using the property type field, verified against existing populated records to confirm accuracy.
- Excluded 4 records (out of ~1.75M) containing implausible transaction dates consistent with unconverted Hijri calendar years — immaterial to overall analysis but noted for completeness.

Full documentation of these decisions is available in [`data_model_documentation.md`](./power_bi_model_documentation.md).

## Recommendations
- **Validate `property_usage_ar`/`en` and `project_number` fields at the source** — both showed inconsistent or incomplete values during this analysis and may benefit from a data quality review upstream.
- **Consider a unique property/unit-level identifier** across transactions — the current dataset links repeat transactions on the same physical unit only loosely (via building/area), which limits any "same-property history" analysis.
- **If maintained going forward**, this dashboard could be extended with a live refresh connection and additional geographic granularity if coordinate-level location data becomes available.

## Dashboard Structure
Six pages: Overview, Time-Based Analysis, Geographic Analysis, Property/Segment Analysis, Drill-Through (transaction-level detail), and Tooltip (contextual hover insights) — with consistent navigation, area-level drill-through, and dynamic year-over-year comparisons throughout.
