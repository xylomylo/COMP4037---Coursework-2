# Annual & Summary NHS Clinical Burden by ICD Levels & Age Groups

**COMP4037 Research Methods — Coursework 2: Data Visualisation**
Anan Faiyaz Radin

An interactive nested treemap of NHS Hospital Episode Statistics (HES) primary
diagnoses spanning 26 financial years (FY 1998–99 → FY 2023–24), drillable
across the full ICD-10 hierarchy (Chapter → Summary Group → 3-character →
4-character) and age bands. Built from heterogeneous NHS spreadsheets and
enriched with the WHO ICD-10 2019 classification.

---

## Repository Layout

```
.
├── data/
│   ├── raw/           # NHS HES source files (gitignored)
│   └── processed/     # Cleaned master CSV output
├── notebooks/
│   └── DataWrangler_4C.ipynb   # End-to-end preparation pipeline
└── output/            # Generated artefacts
```

`data/raw` is excluded from version control due to size; rebuild by downloading
the NHS HES source files listed under References.

---

## Dataset at a Glance

**54 data files across 26 financial years**, split into two eras:

- **Group A (1998–2011, 14 FYs)** — `.xls` files in per-year folders (3 sheets
  each), with 4 coarse age bands (0–14, 15–59, 60–74, 75+).
- **Group B (2012–2024, 12 FYs)** — multi-sheet `.xlsx` workbooks at the root,
  with 20 fine age bands (0, 1–4, 5–9, … 85–89, 90+).

ICD-10 abstraction levels: **Summary** (~216 chapter ranges), **3-character**
(~1,614 codes), and **4-character** (~8,000 codes; ~1,565 stable across all 26
years, 49 added including the U-chapter for COVID).

Key fields retained: ICD code / description, Finished Consultant Episodes
(FCE), Finished Admission Episodes (FAE), emergency / waiting-list / planned
admissions, mean and median wait time and length of stay, mean age, age-band
percentages, day cases, and FCE bed days.

---

## Data Preparation Pipeline

Final output: **`data/processed/df_4c_final.csv`** — **237,978 rows × 25
columns** (26 years × 9,153 WHO 4-character codes).

The 8-phase pipeline, implemented in
[`notebooks/DataWrangler_4C.ipynb`](notebooks/DataWrangler_4C.ipynb):

1. **Load** — Read 26 heterogeneous sheets into per-year DataFrames using
   per-year header-offset and sheet-name maps.
2. **Clean** — Extract ICD codes via regex (`.0–.9` and `.X`), strip footnote
   marks, and harmonise 18 column-name variants into a canonical schema.
3. **Typecast + MAG** — Convert privacy-suppression placeholders (`* - . .. :`)
   to NaN, enforce nullable dtypes, and compute Max Age Group via `idxmax`
   over the 4 canonical age buckets.
4. **Reshape** — Roll Group B's fine age bands up to the 4 canonical buckets,
   insert fiscal-year labels, and concatenate all 26 DataFrames.
5. **Trim** — Drop rows where every metric column is NaN (privacy-suppressed).
6. **Enrich** — Parse the WHO `icd102019en.xml` ClaML file (22 chapters, 274
   blocks, 2,090 3-char, 9,153 4-char codes) to attach chapter, group, and
   parent labels; patch U07.0 manually.
7. **Dense grid + Final** — Left-join observations onto a full
   `Year × 9,153 codes` Cartesian grid; zero-fill count metrics while leaving
   means / medians NULL; revert count metrics to NA for pre-introduction
   years of post-1994 codes (U04 SARS 2003; U06 / U07.x COVID + Vaping 2019);
   derive age-band percentages; write the CSV.
8. **Validate** — 12 automated integrity checks (row count, schema order,
   hierarchy nulls, 3C⊂WHO, Age % sums, raw spot-check `I21.9 2022 = 6512`,
   `.X` coexistence, MAG ties).

### Key Design Decisions

- **Zero-imputation split** — count metrics filled with `0`; mean / median
  metrics left `NULL` (mean of zero admissions is undefined).
- **Pre-introduction guard** — distinguishes *"no admissions"* from *"code
  not yet defined"* for codes introduced mid-series.
- **Fiscal-year convention** — `Year = N` denotes the NHS fiscal year starting
  April N (e.g., `2022` ≡ Apr-2022 → Mar-2023).
- **`.X` semantics** — retained as valid 4-character codes per the WHO
  "category-as-whole" convention; NHS-only obsolete `.X` codes (A90 / A91 /
  B59 / I84) are filtered during Cartesian expansion.

---

## Visualisation

- **Type** — Nested treemap with interactive drill-down and filters.
- **Tooling** — Jupyter Notebook (Python) for preparation; Tableau Public for
  the interactive treemap.
- **Mappings** — Hierarchy Year → Chapter → Group → 3-char → 4-char → Age
  Group (optional); hue encodes ICD chapter, tint encodes dominant age group;
  rectangle area encodes Σ FCE; tooltip surfaces names, dominant age group,
  mean age-band percentages, and Σ FCE.
- **Focus cohort** — All ages across FY 1998–2024 at all four ICD abstraction
  levels.

### Unique Observation

The largest light-brown box in the upper-right region of the base
visualisation — the 75+ age band (lightest tint) within a
digestive / respiratory chapter — is dominated by **ICD 4C `J18.1` (Lobar
Pneumonia, unspecified)**. Drill-down and the supplementary line chart reveal
its long-run growth trend, a finding not apparent from the raw spreadsheets
and consistent with the digestive-system burden reported in Nasar et al.
(2023).

---

## Rebuilding from Source

1. Clone the repository.
2. Download the NHS HES primary diagnosis files (1998–99 → 2023–24) into
   `data/raw/` following the two-era folder convention.
3. Download the WHO ICD-10 2019 ClaML XML (`icd102019en.xml`) into `data/raw/`.
4. Run [`notebooks/DataWrangler_4C.ipynb`](notebooks/DataWrangler_4C.ipynb)
   end-to-end.
5. Load `data/processed/df_4c_final.csv` directly into Tableau Public.

---

## Supplementary Links

- **Video walkthrough** — <https://youtu.be/5xrSCpEAqNA>
- **Tableau Public** — (link in report)
- **Supplementary charts** — Age-group % over 26 years; FCE over 26 years

---

## References

**Academic.**
Liu, X., et al. (2022). *Visualization Resources: A Survey.* Information
Visualization, 22(1), 3–30. <https://doi.org/10.1177/14738716221126992> ·
Nasar, A. Y. (2023). *Hospitalisation Profile in England and Wales, 1999 to
2019: An Ecological Study.* BMJ Open, 13(4), e068393.
<https://doi.org/10.1136/bmjopen-2022-068393> · Kimball, R., & Ross, M.
(2013). *The Data Warehouse Toolkit* (3rd ed.). Wiley.

**Data sources.**
NHS England — *Hospital Episode Statistics: Admitted Patient Care* (primary
diagnosis tables, FY 1998–2024).
<https://digital.nhs.uk/data-and-information/data-tools-and-services/data-services/hospital-episode-statistics> ·
WHO — *ICD-10 Version: 2019.* <https://icd.who.int/browse10/2019/en> · WHO —
*Classification Markup Language (ClaML) Schema.*
<https://www.who.int/standards/classifications/classification-of-diseases>

**Libraries.**
[pandas](https://pandas.pydata.org/docs/) ·
[`numpy.where`](https://numpy.org/doc/stable/reference/generated/numpy.where.html) ·
[`numpy.amax`](https://numpy.org/doc/stable/reference/generated/numpy.amax.html) ·
[`xml.etree.ElementTree`](https://docs.python.org/3/library/xml.etree.elementtree.html).

**Tutorials & videos.**
Moffitt, C. — *Combining Data From Multiple Excel Files*
(<https://pbpython.com/combining-excel-files.html>) · Traceytyh — *Excelpython*
(<https://github.com/traceytyh/excelpython>) · Laramee, B. — *A Short
Introduction to Treemaps* (<https://youtu.be/Ovz29t5ML9s>), *Four Good
Examples of CW2 Submissions* (<https://youtu.be/S1yKN9bvhiA>), *Tableau for
Absolute Beginners* (<https://youtu.be/vah0Y6YDhts>), *Applied Visualization
using Tableau* (<https://youtu.be/bIz0BuEvHQ4>) · Tableau Tim — *Dynamic Sheet
Swapping* (<https://www.youtube.com/watch?v=VZDTaxEdWnM>), *Tree Map Drill
Down* (<https://youtu.be/cOATHf0608o>) · Tableau Software — *Drill Down
Treemap* (<https://www.youtube.com/watch?v=UJRatVd_ZSU>) · Basulto, S. —
*Pandas & Python for Data Analysis*
(<https://www.youtube.com/watch?v=gtjxAH8uaP0>).

---

## Acknowledgements

- The structure of the Data Wrangling pipeline in Jupyter Notebooks is adapted
  from and inspired by the lab coursework material for **COMP4030: Data
  Science with Machine Learning**, with heavy customisation for the NHS
  dataset.
- Claude AI was used to perform initial research on resources required to
  prepare the data wrangling pipeline. It was also used to write the final
  `README.md` file for GitHub by compiling personal Obsidian notes.
