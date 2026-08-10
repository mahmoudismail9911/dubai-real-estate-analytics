# Power BI Data Model Documentation — Dubai Real Estate Transactions

**Project:** Power BI Dashboard — Dubai Real Estate Transactions (self-directed portfolio project)
**Source:** data.dubai (Dubai Land Department Open Data Portal)
**Volume:** ~1,750,283 rows across 2 CSV files
**Architecture:** Star schema — 1 unloaded shared query (`Transactions_Raw`), 1 fact table, 9 dimension tables + 1 date table

---

## 1. Model Architecture

`Transactions_Raw (Load OFF)` holds all shared cleaning logic (type casting, null handling, corrections). It does not load into the data model — every other query references it, so every fix applied there is inherited consistently by the fact table and every dimension, with no duplicated logic.

**Tables loaded into the model:**
- `Fact_Transactions`
- `Dim_Area`, `Dim_Procedure`, `Dim_PropertyType`, `Dim_PropertySubType`, `Dim_RegistrationType`, `Dim_TransactionGroup`, `Dim_Rooms`, `Dim_PropertyUsage`, `Dim_Project/Building`
- `Calendar` (Date table, marked as such, related on the `Date` column, trimmed to ~14 relevant columns)

All dimensions relate to `Fact_Transactions` in a pure star (no snowflaking), one-to-many, single-direction filtering.

---

## 2. Data Quality Findings & Resolutions

### 2.1 Villa Records — Blank Sub-Type Backfilled
Villa-type records (`property_type_id = 4`) sometimes had a blank `property_sub_type`. Backfilled to "Villa"/"فيلا"/id 4, confirmed against existing populated Villa rows that this is the correct pre-existing ID (not invented). Verified across the full dataset that Villa-type rows only ever contain "Villa" or blank as sub-type — no conflicting third value — so the backfill is safe.

### 2.2 Standardized Sentinel Key for Unknown/Missing Values
Every dimension with nullable keys uses **-1** as the unknown/missing sentinel, paired with consistent labels ("NA" / "غير محدد"). Applied to: `property_sub_type_id_clean`, `rooms_key`, `project_building_key`. Text fields were trimmed and blank strings unified with nulls before replacement, to prevent multiple inconsistent "blank" rows surviving deduplication.

### 2.3 `project_number` Excluded from the Model
`project_number`/`project_name` population doesn't reliably align with `reg_type` (Off-Plan vs Existing) as initially assumed — found Off-Plan records with null `project_number` and vice versa. Given inconsistent coverage and no clean independent business meaning beyond what `reg_type` already captures, these columns were dropped entirely from `Fact_Transactions`.

### 2.4 Master Project / Building Modeled as One Combined Dimension
`master_project` and `building_name` form a natural one-to-many hierarchy but neither has a native ID. Combined into a single `Dim_Project/Building` table (4 text columns + surrogate key `project_building_key`), with a Power BI hierarchy (Master Project → Building) built on top for drill-down/drill-through — chosen over snowflaking into two related tables for simplicity, since Power BI hierarchies work best as columns within one table.

### 2.5 `property_usage` — Cross-Language Field Contamination
More extensive than a single corrupted value: investigation found **three separate corruption patterns** in the usage fields:
- `property_usage_ar` sometimes contained the literal English word **"Other"** instead of the Arabic equivalent
- `property_usage_ar` sometimes contained **"مليون"** ("million") — unrelated garbage text, likely a leaked value from another field/process
- `property_usage_en` sometimes contained Arabic text ("اخري") instead of "Other"

**Resolution:** Standardized all three patterns to a single consistent pairing — `property_usage_en = "Other"` / `property_usage_ar = "أخرى"` — using cross-referencing between the two sibling columns as the source of truth (each helped correct the other where one side was reliable and the other corrupted).

**Recommendation for stakeholders:** This level of cross-language contamination (English leaking into the Arabic column and vice versa, plus unrelated garbage values) suggests a data pipeline/import issue upstream, not just isolated typos. Recommend a full field-level audit of `property_usage_ar`/`en` pairing at the source before this field is relied on for other reporting.

### 2.6 `property_type` / `property_sub_type` Inverted Pairing — Immaterial Volume
A small number of records show illogical pairings (e.g. `property_type_en = "Unit"` with `property_sub_type_en = "Building"`). Confirmed volume: **4 records** with sub-type = "Building", **2 records** with sub-type = "Unit" (6 total out of ~1.75M rows). Given the negligible scale and no reliable in-row signal to determine correct values, left unmodified — noted as a minor, immaterial data quality footnote rather than corrected or specially handled in the model.

### 2.7 Alternative Project-Modeling Approach Considered and Rejected
Before settling on the `Dim_Project/Building` (master_project + building_name) structure, a separate approach was evaluated: building a `DimProject` keyed on `project_name_en/ar` with a generated surrogate key, keeping `project_number` only as a secondary reference attribute (rather than excluding project_number/project_name entirely).

This alternative has merit in principle — project-level detail is a common real estate analysis dimension, and a surrogate key sidesteps the null-coverage problem with `project_number` specifically. However, it was not adopted here because `project_name` coverage was found to have the same fundamental reliability issue as `project_number` — nulls did not correspond to any identifiable business rule (e.g. property type, reg_type) that would justify a clean "Unknown Project" categorization. Since `master_project`/`building_name` had meaningfully broader and more consistent coverage across transaction types, and already forms a legitimate business hierarchy, that pairing was used as the project-level analytical dimension instead, with `project_number`/`project_name` dropped rather than carried as a partially-reliable secondary attribute.

**Semantic note on sentinel labels:** missing values in this dataset more accurately represent "identifier not provided by source" rather than "does not apply" — e.g. a villa with a null master_project likely still belongs to some project, the field was simply not captured, rather than the property genuinely having no project. The Arabic label used ("غير محدد" — "unspecified/not determined") already reflects this nuance correctly. The English label was updated from the initial "NA" to **"Unknown"** for the same reason — "N/A" commonly implies "not applicable," while "Unknown" more accurately signals "not provided by source." This label is used consistently across `Dim_PropertySubType`, `Dim_Rooms`, and `Dim_Project/Building`.

### 2.8 Implausible Transaction Dates (Hijri Calendar Artifacts)
Sorting `instance_date` surfaced 4 records (out of ~1.75M) with years in the 1416–1422 range — implausible as Gregorian dates for Dubai property transactions, but consistent with unconverted Hijri (Islamic) calendar years left un-transformed during a source import. Rather than attempt a Hijri-to-Gregorian conversion (which risks introducing incorrect dates), these 4 records were excluded from the model entirely. All other early dates (1966 onward) were verified as plausible, gradually-increasing transaction volume consistent with Dubai's real market history, and were retained.

### 2.9 Historical Volume Distribution and Market Context
A year-by-year breakdown of the full dataset showed a clear, plausible growth curve rather than random noise: minimal, largely mortgage-classified activity from 1966–1997; steady growth from 1998 through the mid-2000s; an early peak in 2009; a pullback through 2010–2011; sustained growth through 2019; a 2020 dip; and a sustained surge from 2021 onward, with 2024 and 2025 both marking record highs. This pattern was used as validation that the historical data (including the sparse 1966–1999 records, ~2,986 rows in one source file alone) reflects genuine transaction history rather than a data quality artifact, and informed the decision to set the Calendar/date table's range starting at 1966 while defaulting dashboard views to more recent, denser periods (e.g. 2025) for readability.

---

## 3. Modeling Decisions Worth Noting

### 3.1 `Dim_PropertySubType` Deduplication Logic
This dimension deduplicates on `property_sub_type_id_clean` only (not the full 3-column combination), on the assumption that each ID maps to exactly one consistent label pairing. This is a reasonable assumption given the ID list is a fixed, known classification set — but it means if any single ID ever had two different text labels in the source (a labeling inconsistency rather than a missing value), only the first-encountered label would survive silently. No evidence of this was found, but it's a modeling assumption worth being aware of rather than a verified guarantee.

### 3.2 Columns Deliberately Excluded from `Fact_Transactions`
Removed as non-essential for the dashboard's analytical scope: `nearest_landmark_ar/en`, `nearest_mall_ar/en`, `nearest_metro_ar/en`, `Source.Name`, `load_timestamp`, `no_of_parties_role_1/2/3`, `project_number`, and all descriptive text columns superseded by dimension foreign keys. This kept the fact table lean (~60MB full model) given the 1.75M row volume.

### 3.3 Foreign Keys Carried in `Fact_Transactions`
`area_id`, `procedure_id`, `property_type_id`, `property_sub_type_id_clean`, `reg_type_id`, `trans_group_id` (native source IDs), plus generated surrogate keys `project_building_key`, `rooms_key`, `property_usage_key`.

---

## 4. Open Items / Judgment Calls Still to Confirm

- Whether `property_usage_ar`'s other distinct values (Residential/سكني, Commercial/تجاري, etc.) are fully clean, or whether the audit should extend beyond the "Other" pairing specifically.
- Whether the March/April/June 2026 dip observed in monthly Year-over-Year comparisons reflects genuine market activity or a data completeness gap in the most recent months of the extract — not yet resolved with a row-count check.

## 5. Dashboard Structure (Final)

Six pages, built on the model described above:
1. **Overview** — headline KPIs, all-time sales trend, transaction group and property type composition, top 5 areas
2. **Time-Based Analysis** — monthly/yearly trend with transaction count overlay, year-over-year month comparison
3. **Geographic Analysis** — Top 20 areas matrix with conditional formatting, geocoded map (via an "Area Full Name" column with city/country suffix appended for reliable geocoding)
4. **Property/Segment Analysis** — property type, usage, room-type price comparison, off-plan % by property type
5. **Drill-Through** — area-level detail (KPIs, trend, transaction-level table), triggered from Geographic Analysis
6. **Tooltip** — custom hover card (KPIs + off-plan/existing composition), triggered from Overview's Top 5 Areas chart

All pages share consistent DLD branding, a custom page-navigation control, and Year slicers sorted descending (most recent year first) as the default view.

---

*This document is built directly from the implemented M code and project notes, and reflects the authoritative record of data quality and modeling decisions made throughout the build.*
