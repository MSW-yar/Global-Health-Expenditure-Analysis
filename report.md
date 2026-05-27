# In-Depth Analytical Report
# Global Health Expenditure Analysis

**Author:** Muhammad Shaheryar Wasim  
**Program:** MS Business Analytics, Mercy University New York  
**Tools:** Python 3, Google Colab, Pandas, Plotly, Matplotlib, scikit-learn  
**Dataset:** WHO Global Health Expenditure Database (GHED)  
**Period Covered:** April 2026

---

## 1. Introduction

### 1.1 Project Background

Health expenditure is one of the most consequential economic measures of a nation's commitment to its population's wellbeing. How much a country spends on health, how that spending is financed, and how the burden falls between government, employers, insurers, and households directly determines whether people access care or face financial catastrophe when sick.

The WHO Global Health Expenditure Database (GHED) is the most comprehensive cross-national health finance dataset available — tracking 192 countries from 2000 through 2021 across hundreds of health expenditure indicators. This project analyses the core financing indicators to quantify the global income inequality gradient in health spending, identify countries at risk of financial hardship from out-of-pocket costs, and examine how COVID-19 disrupted two decades of gradual improvement in health financing equity.

The project is also technically significant: the raw dataset spans 4,224 rows and 3,220 columns and required one of the most demanding data cleaning pipelines in this portfolio — 12 distinct steps addressing encoding failures, comma-formatted numbers, missing value cascades, domain violations, and categorical inconsistencies.

### 1.2 Relevance to MS Business Analytics

This project builds competency in:

- **Enterprise-scale data cleaning** — systematic 12-step pipeline for wide, messy datasets
- **Interactive dashboard development** — Plotly for business-facing visualizations
- **Geospatial analytics** — choropleth and bubble maps for cross-country comparison
- **Principal Component Analysis** — dimensionality reduction for multi-indicator datasets
- **Health economics** — applying business analytics skills in a high-impact public sector domain
- **Stakeholder communication** — translating technical findings into policy-relevant insights

---

## 2. Dataset Description

### 2.1 Overview

| Attribute | Value |
|-----------|-------|
| Source | WHO National Health Accounts (NHA) / GHED |
| Raw dimensions | 4,224 rows × 3,220 columns |
| Clean dimensions | 3,982 rows × 3,228 columns |
| Countries | 192 |
| Period | 2000 – 2021 (22 years) |
| Core indicators | 6 |
| File format | Excel (.xlsx, 3 sheets) |
| Encoding | Windows-1252 (not UTF-8) |

### 2.2 Core Indicators

| Indicator | Column | Mean | WHO Threshold |
|-----------|--------|------|---------------|
| CHE % GDP | che_gdp | 6.2% | — |
| CHE per Capita (USD) | che_pc_usd | $293 (mean) / $184 (median) | — |
| OOP % CHE | oops_che | 33.4% | **< 40%** |
| External % CHE | ext_che | 8.6% | — |
| VHI % CHE | vhi_che | 4.5% | — |
| Private % CHE | pvtd_che | ~45% | — |

The large CHE per Capita mean–median gap ($293 vs $184) reflects a right-skewed distribution — a small number of very high-spending countries (USA, Switzerland, Norway) pull the mean far above the median, while most of the world's 192 countries cluster at much lower spending levels.

---

## 3. Methodology — 12-Step Cleaning Pipeline

The dataset presented 10 distinct technical challenges encountered sequentially during cleaning. Each is documented as a lesson learned.

### 3.1 Step 1 — Encoding Error

**Problem:** `pd.read_csv()` crashed with `UnicodeDecodeError: 'utf-8' codec can't decode byte 0x93`.

**Cause:** Excel exports CSVs in Windows-1252 encoding by default. Byte 0x93 is a Windows-1252 curly-quote character that UTF-8 cannot decode.

**Solution:** `pd.read_csv('file.csv', encoding='windows-1252')`

**Lesson:** Always specify encoding when reading Excel-exported CSVs. This encoding issue will silently affect any file containing curly quotes, em-dashes, or other Windows-specific characters.

### 3.2 Step 2 — Wrong File Loaded

**Problem:** The "data" CSV was actually the codebook/metadata sheet with only 12 columns and no numeric health data.

**Cause:** The GHED Excel file contains 3 sheets. Exporting from the wrong sheet produced a file with column definitions instead of actual country-year data.

**Solution:** Open the Excel file directly and verify which sheet contains the main data (rows = country-years, columns = indicators).

**Lesson:** Multi-sheet Excel files must be inspected sheet by sheet before any analysis. A shape check immediately after loading — if it shows (4224, 12) instead of (4224, 3220), the wrong sheet was exported.

### 3.3 Step 3 — Column Mapping Failures

**Problem:** Canonical column names from the WHO codebook (`iso3`, `region_who`, `income_group`) raised `KeyError` — the actual file uses `code`, `region`, `income`.

**Cause:** Documentation names often differ from actual file column names, especially for international datasets maintained across multiple versions.

**Solution:** Always print `df.columns` immediately after loading and wire canonical column names from actual output, not documentation.

**Lesson:** A diagnostic cell that prints column names and confirms each expected column exists/is missing is non-negotiable before any downstream processing.

### 3.4 Step 4 — Comma-Formatted Numbers

**Problem:** 863 numeric columns contained string values like `"9,423"` — pandas could not convert these to numeric.

**Cause:** Excel inserts thousands separators when exporting large numbers to CSV.

**Solution:** Strip all commas from non-ID object columns before calling `pd.to_numeric()`:
```python
df[col] = df[col].astype(str).str.replace(',', '', regex=False)
df[col] = pd.to_numeric(df[col], errors='coerce')
```

**Lesson:** This is one of the most common silent data quality issues in Excel-sourced data. Numbers that look numeric may fail `pd.to_numeric()` silently if `errors='coerce'` is used — always count coerced NaN values to detect this issue.

### 3.5 Step 5 — ID Column Corruption

**Problem:** Blanket `pd.to_numeric()` conversion wiped string ID columns (`code`, `region`, `income`) to NaN.

**Solution:** Define an explicit `ID_SET` exclusion list before casting:
```python
numeric_candidates = df.columns.difference(ID_COLS)
```
Then restore from `df_raw` if corruption occurred.

### 3.6 Steps 6–7 — Missing Value Cascade

**Problem:** Core indicators showed 5–27% missing values depending on country and year.

**Strategy (4-stage cascade):**
1. Drop all-null rows (empty records from sheet padding)
2. Forward/backward fill within each country's time series — most missingness is "no data for this year" where adjacent years have data
3. Income-year median imputation — if country has no adjacent data, use comparable income group + year median
4. Global median fallback — last resort for isolated missing values

After this cascade, **0 NaN values remained in any core column** across 3,982 rows.

### 3.7 Steps 8–12 — Validation and Engineering

**Domain validation:** One CHE % GDP value of 50.2% was identified and clipped to 40% — no country spends more than 40% of GDP on health; this value represents a data entry error.

**Winsorisation:** All core columns winsorised at 1st–99th percentile to reduce extreme value influence on visualisations.

**Feature engineering (7 new columns):**
- `period` — 5-year bins (2000–04, 2005–09, ...)
- `decade` — decade label
- `covid_flag` — binary flag for 2020–2021
- `oop_risk` — categorical OOP risk tier (Low/Medium/High vs WHO threshold)
- `oop_quintile` — quintile within income group
- `yoy_growth` — year-on-year CHE per Capita growth rate
- `che_pc_log` — log-transformed CHE per Capita for regression use

---

## 4. Analysis Results

### 4.1 Income Inequality Gradient

The single most consistent finding across all 6 indicators is the stark income inequality gradient:

| Income Group | CHE per Capita | OOP % CHE | External % CHE | VHI % CHE |
|-------------|----------------|-----------|----------------|-----------|
| High | ~$470+ | 10–20% | ~1% | ~10% |
| Upper-middle | ~$224 | ~30% | ~2% | ~5% |
| Lower-middle | ~$138 | ~40–50% | ~5% | ~2% |
| Low | ~$40 | 30–55% | ~30% | ~0% |

In low-income countries, OOP spending + External Aid accounts for approximately **95% of total health financing**. This means:
- Almost no government health expenditure
- Almost no insurance system (VHI near zero)
- Households pay directly at the point of care — or forgo care entirely
- Healthcare access is a function of individual wealth, not entitlement

### 4.2 The OOP Threshold and Financial Hardship

The WHO designates OOP > 40% of total health spending as the financial hardship threshold — at this level, household health costs crowd out essential spending on food, education, and housing. Key findings:

- High income countries: IQR 10–20% OOP — well below threshold
- Low income countries: IQR 30–55% OOP — majority at or above threshold

Countries above the 40% threshold in the latest available year include Afghanistan (79% OOP), Honduras (50% OOP), and Philippines (41% OOP) — all lower-middle income nations where public health systems are severely underfunded.

### 4.3 COVID-19 Impact

The COVID-19 period (2020–2021) produced the most dramatic health expenditure disruption in the 22-year dataset:

**CHE % GDP spike:** All income groups showed sharp increases in 2020 as governments mobilised emergency health spending while GDP contracted — the double effect of increased health expenditure and reduced denominator.

**OOP reversal:** The steady 20-year decline in OOP % CHE reversed sharply in 2020–2021. As public health systems were overwhelmed by COVID, out-of-pocket spending surged — particularly in low and lower-middle income countries where public capacity was most constrained.

**Low income financing collapse:** The stacked area charts show that in the Low income panel, the OOP + External Aid layer accounts for nearly the entire bar, with COVID-19 causing the OOP layer to expand dramatically as donor-funded systems struggled to cope.

### 4.4 Correlation Analysis

| Pair | Pearson r | Spearman r | Interpretation |
|------|-----------|------------|----------------|
| OOP vs Private CHE | 0.92 | 0.91 | OOP is private spending in LICs — no insurance |
| External vs CHE per Capita | −0.60 | — | Richer countries receive less donor aid (expected) |
| OOP vs CHE % GDP | −0.34 | −0.32 | More GDP spending = lower household burden |
| CHE per Capita vs VHI | 0.54 | 0.49 | Richer countries have more insurance |

The OOP–Private CHE correlation of 0.92 reveals a structural insight: in low-income countries, the "private health sector" essentially means "pay out of pocket at a clinic." There are no meaningful insurance markets — private and OOP are nearly synonymous.

### 4.5 PCA Results

Principal Component Analysis on the 6 core indicators:

| Component | Variance Explained | Dominant Loadings | Interpretation |
|-----------|-------------------|-------------------|----------------|
| PC1 | 47.6% | OOP +, External +, Private + | "Poverty of financing" — high on this axis = low-income |
| PC2 | 27.8% | CHE per Capita +, VHI + | "Spending level" — high = high absolute expenditure |
| PC3 | 11.8% | CHE % GDP + | "Fiscal commitment" |
| PC4 | 10.5% | Mixed | Residual variance |

PC1 + PC2 together explain 75.4% of total variance. The biplot coloured by income group shows clear separation — high income countries cluster in the lower-right (high spending, low OOP), while low income countries cluster in the upper-left (low spending, high OOP).

---

## 5. Dashboard Design

### 5.1 Static Matplotlib Dashboard (9 panels)

Designed for offline sharing and PDF export:
- Row 1: CHE per Capita by income, CHE % GDP by WHO region, CHE per Capita trend 2000–2021
- Row 2: OOP distribution, CHE financing composition, CHE per Capita vs OOP scatter
- Row 3: Time-series with COVID shading, Financing composition over time, Country rankings

### 5.2 Interactive Plotly Dashboard (6 panels)

Designed for stakeholder exploration with hover tooltips revealing country-level detail:
- All 192 countries accessible through hover
- Cross-filtering by income group through legend clicks
- COVID-19 annotation on trend chart

### 5.3 Animated Choropleth Map

Year slider from 2000 to 2021 shows the geographic progression of health spending:
- Dark regions (Sub-Saharan Africa, South Asia) remain consistently low throughout
- Western Europe, North America brighten steadily
- COVID-19 spike visible globally in 2020

### 5.4 Bubble Map

Most impactful single visualisation for non-technical stakeholders:
- Bubble size = CHE per Capita
- Bubble colour = income group
- Large blue bubbles cluster over Europe and North America
- Tiny red dots cover Sub-Saharan Africa

"This single chart communicates the inequality story more immediately than any table or text."

---

## 6. Business and Policy Implications

### 6.1 For International Development Organisations

- Countries with OOP > 40% and External Aid > 20% represent the highest-priority targets for universal health coverage (UHC) investment
- COVID-19 reversed OOP progress in exactly the countries least able to absorb the shock — emergency preparedness funding for LICs is quantifiably underfunded
- The declining-OOP trend (2000–2019) shows that progress is possible; the 2020 reversal shows it is fragile

### 6.2 For Health Insurance Companies

- Upper-middle income countries show rapid VHI growth — these are the primary expansion markets for voluntary health insurance products
- The OOP-to-VHI transition pathway is quantifiable from this data — countries at 30–40% OOP with growing middle class are prime candidates for private insurance market entry

### 6.3 For Sovereign Debt and Credit Analysis

- External aid dependency (>20% of CHE) indicates a country's health system is not self-sustaining — a credit risk factor for health-sector bonds
- The CHE % GDP indicator, relative to income group peers, indicates whether a government is under- or over-investing in health relative to economic capacity

---

## 7. Lessons Learned

The 10 most significant technical lessons from this project are documented in the notebook. The most broadly applicable:

**Always diagnose before cleaning.** Shape, column names, data types, and null counts should be the first four outputs of any cleaning notebook. Every downstream step depends on these fundamentals being correct.

**Column names in documentation ≠ column names in files.** This caused hours of debugging. The WHO codebook names and the actual GHED file column names are different — wiring canonical variables from `df.columns` output, not documentation, is non-negotiable.

**Excel's formatting is the enemy of data analysis.** Windows-1252 encoding, comma-formatted numbers, thousand-separator integers, multi-sheet files — all are consequences of Excel being optimised for human reading rather than machine parsing. Every Excel-sourced dataset should be treated as requiring significant cleaning before any analysis.

**Use groupby().last() for sparse cross-sectional analysis.** Point-in-time filters (`year == 2021`) on sparse datasets with non-uniform coverage silently reduce sample sizes by 85–90%. Per-country last-available-year gives a representative cross-section without requiring every country to have the same latest year.

---

## 8. Limitations and Future Work

### 8.1 Limitations

- GHED 2021 data is sparse — most countries have coverage only to 2020
- Dollar-denominated CHE per Capita values are not PPP-adjusted — $40 in Sub-Saharan Africa buys more healthcare than $40 in Western Europe
- Country-level aggregates hide within-country inequality (urban vs rural, wealth quintile access differences)
- Missing data imputation (income-year median) may understate true country-level variation

### 8.2 Future Work

- Add PPP-adjusted CHE per Capita for more meaningful cross-country comparison
- Integrate disease burden data (DALYs, mortality) to compute health expenditure efficiency ratios
- Build a regression model predicting OOP % CHE from income, government spending, and insurance penetration
- Create an OOP financial hardship risk index for insurance market prioritisation
- Extend to most recent GHED release (2023) for COVID-recovery period analysis

---

## 9. Conclusion

This project successfully delivers a comprehensive health expenditure analysis across 192 countries and 22 years, supported by an exhaustive 12-step cleaning pipeline that resolved 10 distinct technical challenges. The core findings — widening absolute CHE per Capita gaps between income groups, OOP financial hardship concentrated in low-income countries, and COVID-19's reversal of 20 years of OOP decline — provide actionable intelligence for development organisations, health insurers, and sovereign analysts.

The project's technical contribution is the documented cleaning pipeline itself — a reusable template for handling large, wide, multi-sheet, Excel-sourced datasets with mixed encodings, comma-formatted numbers, sparse cross-sectional structure, and domain-specific validation requirements. These challenges are not specific to GHED; they are characteristic of any enterprise dataset sourced from international organisations, government databases, or legacy reporting systems.

---

*Report prepared as part of MS Business Analytics portfolio — Mercy University New York*
