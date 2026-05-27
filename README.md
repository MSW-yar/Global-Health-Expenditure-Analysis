# Global Health Expenditure Analysis

**Author:** Muhammad Shaheryar Wasim  
**GitHub:** MSW-yar  
**Tools:** Python 3, Google Colab, Pandas, Plotly, Matplotlib, scikit-learn  
**Domain:** Data Science — Global Health Analytics & Interactive Dashboard Creation  
**Level:** Medium — Data Analysis

---

## Overview

This project analyses the WHO Global Health Expenditure Database (GHED) — comparative health spending data for **192 countries over 2000–2021** (22 years). The project examines the income inequality gradient in health financing, out-of-pocket burden relative to the WHO 40% financial hardship threshold, COVID-19's impact on health expenditure, and regional disparities.

The dataset presents significant technical challenges: 4,224 × 3,220 columns, Windows-1252 encoding, comma-formatted numbers, and sparse cross-sectional structure — addressed through a **12-step exhaustive cleaning pipeline**.

---

## Dataset

| Attribute | Detail |
|-----------|--------|
| Source | [WHO Global Health Expenditure Database (GHED)](https://apps.who.int/nha/database) |
| Records | 4,224 rows → 3,982 after cleaning |
| Columns | 3,220 raw → 3,228 after feature engineering |
| Countries | 192 |
| Period | 2000 – 2021 |
| File Format | .xlsx (3 sheets) / .csv export |

### Core Indicators

| Indicator | Column | WHO Threshold |
|-----------|--------|---------------|
| CHE % GDP | `che_gdp` | — |
| CHE per Capita (USD) | `che_pc_usd` | — |
| OOP % CHE | `oops_che` | **< 40%** (financial hardship) |
| External % CHE | `ext_che` | — |
| VHI % CHE | `vhi_che` | — |
| Private % CHE | `pvtd_che` | — |

---

## Project Structure

```
global-health-expenditure-analysis/
│
├── global_health_expenditure.ipynb     # Full Jupyter notebook (4 weeks)
├── README.md                           # This file
├── requirements.txt                    # Python dependencies
├── report.md                           # In-depth analytical report
│
└── data/
    └── GHED_data_main.csv              # Main data sheet (export from GHED .xlsx)
```

---

## How to Run

### Google Colab
1. Download GHED data from [WHO NHA Database](https://apps.who.int/nha/database)
2. Export the main data sheet as CSV
3. Upload to Google Drive at `My Drive/Colab Notebooks/Datasets/GHED_data_main.csv`
4. Upload notebook to Colab, uncomment Drive mount cell, run all cells

### Local Jupyter
```bash
pip install -r requirements.txt
# Place GHED_data_main.csv in same directory
jupyter notebook global_health_expenditure.ipynb
```

**Critical:** Always load with `encoding='windows-1252'` — Excel exports use this encoding, not UTF-8. UTF-8 will crash with `UnicodeDecodeError: byte 0x93`.

---

## Methodology

### Week 1 — Domain Understanding
- Researched WHO health expenditure definitions: CHE, OOP, GGHED, PVTD, VHI, External
- Identified WHO 40% OOP threshold as key financial hardship indicator
- Downloaded GHED dataset and diagnosed 3-sheet Excel structure

### Week 2 — 12-Step Cleaning Pipeline
1. Raw column inventory (3,220 columns catalogued)
2. Column name standardisation (regex snake_case normalisation)
3. Canonical column mapping (verified actual column names vs. codebook)
4. Duplicate detection (0 duplicates confirmed)
5. Type enforcement (comma-stripped 863 columns, cast 565 to float64)
6. Missing value audit (87.2% overall completeness)
7. 4-stage imputation cascade (ffill/bfill → income-year median → global median)
8. Domain validation (clipped CHE GDP > 40% violations)
9. Outlier winsorisation (1st–99th percentile all core columns)
10. Categorical consistency check (WHO region codes, income groups)
11. Feature engineering (period bins, decade, COVID flag, OOP risk tiers)
12. Final validation (3,982 rows × 3,228 columns, 0 remaining NaN in core)

### Week 3 — Analysis & Visualization
- Distribution analysis across all 6 core indicators
- Box plots confirming income inequality gradient
- Pearson + Spearman correlation matrices
- Time-series trends 2000–2021 with COVID shading
- PCA dimensionality reduction (PC1+PC2 = 75.4% variance)

### Week 4 — Dashboard Creation
- **Static Matplotlib dashboard** — 9-panel summary
- **Interactive Plotly dashboard** — 6-panel with hover tooltips (192 countries)
- **Animated choropleth map** — CHE per Capita with year slider 2000–2021
- **Bubble map** — spending size + income group colour (192 countries)

---

## Key Findings

### Income Inequality Gradient

| Income Group | CHE per Capita | OOP % CHE | VHI % CHE |
|-------------|----------------|-----------|-----------|
| High | ~$470+ | 10–20% (IQR) | High |
| Upper-middle | ~$224 | ~30% | Moderate |
| Lower-middle | ~$138 | ~40–50% | Low |
| Low | ~$40 | 30–55% (IQR) | Near-zero |

In low-income countries, **OOP + External Aid ≈ 95% of all health financing** — there is virtually no insurance system, making households entirely vulnerable to catastrophic health costs.

### Strongest Correlations
- **OOP vs Private CHE: r=0.92** — in low-income countries, out-of-pocket spending *is* private spending
- **External Aid vs CHE per Capita: r=−0.60** — richer countries receive less aid (expected)
- **OOP vs CHE % GDP: r=−0.34** — countries spending more of GDP on health have lower household burden

### COVID-19 Impact
- CHE % GDP spiked dramatically in 2020 across all income groups
- OOP % CHE reversed its 20-year declining trend in 2020–2021 — public systems overwhelmed, households absorbed costs
- Low income countries saw largest private financing surge

### PCA Structure
- PC1 (47.6%): "Poverty of financing" — OOP, External, Private load positively
- PC2 (27.8%): "Spending level" — CHE per Capita, VHI load positively
- Income groups clearly separate in PC1–PC2 space

---

## Key Lessons Learned

1. **Windows-1252 encoding** — Excel CSV exports need `encoding='windows-1252'`, not UTF-8; byte 0x93 is a Windows curly-quote
2. **Verify column names from df.columns** — never assume codebook names match actual file; confirmed "code", "region", "income" (not WHO documentation names)
3. **Comma-formatted numbers** — Excel adds thousands separators to large numbers; must strip before `pd.to_numeric()`
4. **ID columns need explicit protection** — blanket type-casting can corrupt string identifiers; exclude them explicitly
5. **Use groupby().last() for sparse cross-sectional data** — `year == 2021` returned only 22 countries; per-country last-available-year gives 182–192
6. **Multi-sheet Excel files** — must inspect sheet-by-sheet before exporting; the "data" file was actually the codebook sheet
7. **Composition checks need correct denominators** — OOP + External + VHI ≠ 100% of CHE; full composition requires adding GGHED
8. **`pd.qcut` in groupby needs safety wrapper** — fails when a group has fewer than n unique values; wrap in try/except
