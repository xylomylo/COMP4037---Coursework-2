# Annual & Summary NHS Clinical Burden by ICD Levels & Age Groups

**COMP4037 Research Methods — Coursework 2: Data Visualisation**
Anan Faiyaz Radin

An interactive nested treemap of NHS Hospital Episode Statistics (HES) primary
diagnoses spanning 26 financial years (FY 1998–99 → FY 2023–24), drillable across
the full ICD-10 hierarchy (Chapter → Summary Group → 3-character → 4-character)
and age bands. Built from heterogeneous NHS spreadsheets and enriched with the
WHO ICD-10 2019 classification.

---

## Repository Layout

```
.
├── data/
│   ├── raw/           # NHS HES source files (gitignored — see Source section)
│   └── processed/     # Cleaned master CSV output
├── notebooks/
│   └── DataWrangler_4C.ipynb   # End-to-end preparation pipeline
└── output/            # Generated artefacts
```

`data/raw` is intentionally excluded from version control due to size; rebuild
by downloading the NHS HES source files listed below.

---

## Dataset at a Glance

**54 data files across 26 financial years**, split into two format eras:

| Era | Years | Files | Format | Age Granularity | Available Sheets |
|-----|-------|-------|--------|-----------------|------------------|
| **Group A** (Old) | 1998–2011 (14 FYs) | 3 `.xls` files per yearly folder | `.xls`, 1 sheet each | 4 coarse bands (0–14, 15–59, 60–74, 75+) | Primary Diagnosis: Summary, 3-char, 4-char |
| **Group B** (New) | 2012–2024 (12 FYs) | 1 `.xlsx` per year, 6–7 sheets | `.xlsx` at root | 20 fine bands (0, 1–4, 5–9, … 85–89, 90+) | Intro, Field Descriptions, Primary Diagnosis (Summary/3c/4c), All Diagnosis (3c/4c) |

ICD-10 abstraction levels present in the data:

- **Summary** — ~216–221 grouped ICD-10 ranges (broad diagnostic chapters)
- **3-character** — ~1,614 individual codes
- **4-character** — ~8,000–8,270 most granular codes (**~1,565 stable across all 26 years; 49 added, incl. U-chapter for COVID**)

---

## Selected Fields

| Field | Description |
|-------|-------------|
| Diagnosis Code / Description | ICD-10 code and human-readable label |
| FCE | Finished Consultant Episodes |
| FAE / Admissions | Finished Admission Episodes |
| Emergency / Waiting List / Planned | Admission-type breakdown |
| Mean + Median Wait Time | Days waited |
| Mean + Median Length of Stay | Days in hospital |
| Mean Age, Age Group % | Age distribution |
| Day Cases, FCE Bed Days | Utilisation metrics |

---

## Data Preparation Pipeline (NHS HES 4-Character ICD)

Final output: **`data/processed/df_4c_final.csv`** — **237,978 rows × 25 columns**
(26 years × 9,153 WHO 4-character codes).

### 25-Column Schema

`Year, Chapter, Chapter_Name, Group, Group_Name, ICD_Code_3_Char, Code_Name_3C,
ICD_Code_4Char, ICD_Code_Name_4C, FCE, FAE, Emergency, Waiting List,
Mean/Median Waiting Time, Mean/Median Length of Stay, Mean Age,
Age 0-14/15-59/60-74/75+ %, Day Case, Bed Days (FCE), Max Age Group`

### Phases

| # | Phase | Purpose | Key Operation |
|---|-------|---------|---------------|
| **P1** | Load | Read 26 heterogeneous sheets | `HEADER_OFFSETS_4C` (5 / 3 / 10 / 15 / 16 / 17) and `GROUP_A_SHEET_4C` per-year maps; promote header row, drop fully-null rows |
| **P2** | Clean | Canonicalise columns, isolate ICD codes | `CODE4_RE` regex (`.0–.9` and `.X`); strip `‡ † § *` NBSP footnote marks; `_norm` + `COL_RENAME_4C` + `DROP_TOKENS_4C` harmonise 18 variants; row 0 → `'000'` Total |
| **P3** | Typecast + MAG | Enforce nullable dtypes; compute Max Age Group | Placeholders (`* - . .. :`) → NaN; mean/median → `float64`, others → `Int64` if whole; `idxmax` over 4 canonical age buckets (GA direct / GB via `AGE_BUCKETS_GB` rollup) |
| **P4** | Reshape | Make 26 DataFrames schema-identical; concat | Bucket GB fine bands → 4 canonical via `min_count=1`; insert fiscal-year `Year`; reorder to `CANON_ORDER_4C`; assertion guard on schema mismatch |
| **P5** | Trim | Remove privacy-suppressed empty rows | Drop rows where all 15 metric columns are NaN |
| **P6** | Enrich (XML) | Add Chapter / Group / Name columns from WHO | Parse `icd102019en.xml` → `{chapters: 22, blocks: 274, cat3: 2090, cat4: 9153}`; walk `SuperClass` chain; patch U07.0 name |
| **P6.5** | Dense grid | Zero-imputation + pre-introduction guard | Cartesian `Year × 9153 WHO 4-char codes` → left-join NHS observations; counts → 0-fill, means/medians → NULL; U04/U06/U07 revert to NA for pre-intro years |
| **P7** | Final | Derive Age %; write CSV | Age % = share of 4-band sum (NaN propagated for all-NA rows); reorder to 25-col schema; sort by `Year, Chapter, Group, 3C, 4C` |
| **P8** | Validate | 12 automated integrity checks | Row count, column order, hierarchy nulls, codes-per-year, 3C⊂WHO, 3C/4C cross-consistency, Age % sums, zero-imputed count, raw spot-check (`I21.9 2022 = 6512`), file size, `.X` coexistence, MAG ties |

### Key Design Decisions

- **Zero-imputation split** — count metrics filled with `0`; mean/median metrics left `NULL` (mean/median of zero admissions is undefined).
- **Pre-introduction guard** — known post-1994 WHO codes (U04 SARS 2003; U06/U07.x COVID + Vaping 2019) revert count metrics to `NA` for years before the code existed, distinguishing *"no admissions"* from *"code not yet defined"*.
- **Fiscal-year convention** — `Year = N` denotes NHS fiscal year starting April N (e.g., `2022` ≡ Apr-2022 → Mar-2023). Calendar-year joins are the consumer's responsibility.
- **`.X` semantics** — retained as valid 4-character codes (WHO "category-as-whole"); the WHO skeleton filters NHS-only obsolete `.X` codes (A90 / A91 / B59 / I84) during Cartesian expansion.

### Header Location Variance (Group A)

| Years | `iloc` Header | `iloc` Data Start | Notes |
|-------|---------------|-------------------|-------|
| 1998–2004 | `[2]` | `[3:]` | 1998 has "Primary Diagnosis" instead of "Total" |
| 2005–2006 | `[9]` | `[10:]` | More metadata rows |
| 2007 | `[14]` | `[16:]` | Code & description already in separate columns |
| 2008–2011 | `[14]` | `[16:]` | Combined codes like earlier years, blank row between header and data |

Group B (2012–2024) header positions auto-detected per sheet; the 2012–13
outlier sits at header `iloc=17`, the remainder at `iloc=9–14`.

---

## Visualisation

- **Type** — Nested treemap with interactive drill-down and filters.
- **Tooling** — Jupyter Notebook (Python) for preparation; Tableau Public for the
  interactive treemap.
- **Mappings**
  - *Hierarchy:* Year → ICD Chapter → Summary Group → 3-char → 4-char → Age Group (optional)
  - *Colour:* Hue encodes ICD Chapter; tint encodes dominant age group per level
  - *Size:* Rectangle area encodes Σ FCE
  - *Tooltip:* Chapter, Group, 3C / 4C names, dominant age group, mean age-band %s, Σ FCE

**Focus cohort** — All ages across FY 1998–2024 at all four ICD abstraction levels.

### Unique Observation

The largest light-brown box in the upper-right region of the base visualisation —
the 75+ age band (lightest tint) within a digestive/respiratory chapter — is
dominated by **ICD 4C `J18.1` (Lobar Pneumonia, unspecified)**. Drill-down and
the supplementary line chart reveal its long-run growth trend, a finding not
apparent from the raw spreadsheets and consistent with the digestive/respiratory
burden reported in Nasar et al. (2023).

---

## Rebuilding from Source

1. Clone the repository.
2. Download the NHS HES primary diagnosis files (1998–99 → 2023–24) into
   `data/raw/` following the two-era folder convention described above.
3. Download the WHO ICD-10 2019 ClaML XML (`icd102019en.xml`) into `data/raw/`.
4. Run [`notebooks/DataWrangler_4C.ipynb`](notebooks/DataWrangler_4C.ipynb) end-to-end.
5. The cleaned master CSV will be written to `data/processed/df_4c_final.csv`
   and can be loaded directly into Tableau Public.

---

## Supplementary Links

- **Video walkthrough** — <https://youtu.be/5xrSCpEAqNA>
- **Tableau Public** — (link in report)
- **Supplementary charts** — Age-group % over 26 years; FCE over 26 years

---

## References

### Academic Papers & Books

- Liu, X., Alharbi, M., Chen, J., Diehl, A., Firat, E. E., Rees, D., Wang, Q., &
  Laramee, R. S. (2022). *Visualization Resources: A Survey.* Information
  Visualization, 22(1), 3–30. <https://doi.org/10.1177/14738716221126992>
- Nasar, A. Y. (2023). *Hospitalisation Profile in England and Wales, 1999 to
  2019: An Ecological Study.* BMJ Open, 13(4), e068393.
  <https://doi.org/10.1136/bmjopen-2022-068393>
- Kimball, R., & Ross, M. (2013). *The Data Warehouse Toolkit: The Definitive
  Guide to Dimensional Modelling* (3rd ed.). John Wiley & Sons.

### Institutional Data Sources

- NHS England. *Hospital Episode Statistics (HES): Admitted Patient Care*
  (primary diagnosis tables, FY 1998–2024).
  <https://digital.nhs.uk/data-and-information/data-tools-and-services/data-services/hospital-episode-statistics>
- World Health Organization. *ICD-10 Version: 2019* — Online Browser
  (Vol. 2 §3.1.3 for the `.X` "category as a whole" convention).
  <https://icd.who.int/browse10/2019/en>
- World Health Organization. *Classification Markup Language (ClaML) Schema —
  ICD-10 Release*.
  <https://www.who.int/standards/classifications/classification-of-diseases>

### Libraries & API Documentation

- The pandas Development Team. *pandas Documentation*.
  <https://pandas.pydata.org/docs/>
- NumPy Developers. *`numpy.where`* — NumPy Reference.
  <https://numpy.org/doc/stable/reference/generated/numpy.where.html>
- NumPy Developers. *`numpy.amax`* — NumPy Reference.
  <https://numpy.org/doc/stable/reference/generated/numpy.amax.html>
- Python Software Foundation. *`xml.etree.ElementTree` — The ElementTree XML API*.
  <https://docs.python.org/3/library/xml.etree.elementtree.html>

### Tutorials & Articles

- Moffitt, C. (2016). *Combining Data From Multiple Excel Files.* Practical
  Business Python. <https://pbpython.com/combining-excel-files.html>
- Traceytyh (2020). *Excelpython.* GitHub.
  <https://github.com/traceytyh/excelpython>

### Video Resources

- Laramee, B. *A Short Introduction to Treemaps.* YouTube.
  <https://youtu.be/Ovz29t5ML9s>
- Laramee, B. *Four Very Good Examples of Coursework 2, Data Visualization,
  Submissions.* YouTube. <https://youtu.be/S1yKN9bvhiA>
- Laramee, B. *Tableau for Absolute Beginners: A Live Introduction and Hands-on
  Demonstration.* YouTube. <https://youtu.be/vah0Y6YDhts>
- Laramee, B. *Applied Visualization using Tableau: A Hands-On Tutorial and
  Demonstration.* YouTube. <https://youtu.be/bIz0BuEvHQ4>
- Tableau Tim. *Tableau Tutorial — Dynamic Charts — Dynamic Sheet Swapping in
  Tableau.* YouTube. <https://www.youtube.com/watch?v=VZDTaxEdWnM>
- Tableau Software (2020). *How to Create a Drill Down Treemap in Tableau
  Software.* YouTube. <https://www.youtube.com/watch?v=UJRatVd_ZSU>
- Tableau Tim. *Tableau Tutorial — Tree Map Drill Down (Set Parameter Action).*
  YouTube. <https://youtu.be/cOATHf0608o>
- Basulto, S. *Pandas & Python for Data Analysis by Example — Full Course for
  Beginners.* freeCodeCamp, YouTube.
  <https://www.youtube.com/watch?v=gtjxAH8uaP0>

---

## Acknowledgements

- The structure of the Data Wrangling pipeline in Jupyter Notebooks is adapted
  from and inspired by the lab coursework material for **COMP4030: Data Science
  with Machine Learning**, with heavy customisation for the NHS dataset.
- Claude AI was used to perform initial research on resources required to prepare the data wrangling pipeline. It was also used to write the final `README.md` file for GitHub by compiling personal Obsidian Notes.

