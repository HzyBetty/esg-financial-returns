# ESG Performance & Financial Returns: An Empirical Analysis

> **RSM 8224 – Analytical Insights Using Accounting and Financial Data**  
> Rotman School of Management, University of Toronto | Winter 2026  
> **Group 9:** Haolong Yang, Jiayu Niu, Siyan Li, Zhenyu Wang, Zhongyi Hu

---

## Overview

This repository contains all code, data, and outputs for our group research project investigating whether U.S. firms that perform well on Environmental, Social, and Governance (ESG) dimensions also perform well financially — and in which industries that alignment is genuine versus coincidental.

Using a panel of **18,443 firm-year observations across 3,086 firms (2013–2023)**, we integrate three institutional-grade databases — LSEG (Refinitiv) ESG scores, Compustat accounting data, and CRSP stock return data — and apply the econometric toolkit common in academic asset pricing and corporate finance research.

**My role in this project:** I led the end-to-end pipeline design, including the multi-source database merging strategy (WRDS/CRSP/Compustat), construction of the panel dataset, regression specifications for Parts II–III, and the double materiality quadrant analysis in Part IV. I also drove the analytical narrative connecting our empirical findings to the financial materiality framework.

---

## Key Findings

| Question | Finding |
|---|---|
| Are ESG scores improving? | U-shaped trend: declined 2013–2017 (composition effect), rose steadily post-2017 |
| What drives ESG? | **Firm size** is the dominant predictor (coef = +7.1 per log-unit of market cap) |
| Does ESG predict ROA? | **No** — unconditional correlation disappears entirely after controlling for firm characteristics |
| Does ESG predict stock returns? | **Conditionally yes** — +1.5 pp annual return per 1 SD increase in ESG, after controlling for size, value, and momentum |
| Best industry for ESG-conscious investors? | **Shops (Wholesale/Retail)** — the only FF12 industry that is a double materiality winner on *both* BHR and ROA |

---

## Repository Structure

```
.
├── Final_Project_Code.ipynb      # Full analysis pipeline (see breakdown below)
├── esg_panel.csv                 # Final merged panel dataset (18,443 firm-years)
├── Final_Project_Report.pdf      # Full written report (22 pages + appendix)
├── Final_Project_Slides.pdf      # Presentation deck (15 slides)
├── Final_Project_Description.pdf # Original assignment brief
└── README.md
```

---

## Data Pipeline & Technical Skills Demonstrated

This project is structured as a complete data science workflow, from raw database queries to publishable regression tables. Below is a breakdown of each stage and the transferable skills involved.

### 1. Data Acquisition via SQL on WRDS (`Step 1–3` in notebook)

**Skills:** SQL, financial databases, API/database connectivity

- Connected to WRDS (Wharton Research Data Services) using the `wrds` Python library
- Queried three separate institutional databases using raw SQL:
  - `tr_esg.esgscores` — Refinitiv ESG scores (hierarchical pillar structure)
  - `comp.funda` — Compustat annual fundamentals (balance sheet, income statement)
  - `crsp.msf` + `crsp.msenames` — CRSP monthly stock returns with exchange/share-type filters
- Applied sample filters in SQL (e.g., `shrcd IN (10,11)`, `exchcd IN (1,2,3)`) to restrict to U.S. domestic common stocks on NYSE/AMEX/NASDAQ

```python
crsp_msf = conn.raw_sql("""
    SELECT a.permno, a.date, a.ret, a.prc, a.shrout,
           b.shrcd, b.exchcd, b.siccd
    FROM crsp.msf AS a
    LEFT JOIN crsp.msenames AS b
        ON a.permno = b.permno
        AND b.namedt <= a.date AND a.date <= b.nameendt
    WHERE a.date BETWEEN '2010-01-01' AND '2025-12-31'
      AND b.shrcd IN (10, 11) AND b.exchcd IN (1, 2, 3)
""", date_cols=['date'])
```

---

### 2. Multi-Source Database Merging (`Step 4–5` in notebook)

**Skills:** Multi-key joins, identifier mapping, temporal link tables, data integrity

This is the most technically demanding step. Merging ESG, accounting, and return data requires navigating three different identifier systems:

- **CRSP ↔ Compustat:** Used the CCM (CRSP–Compustat Merged) link table (`crsp.ccmxpf_linktable`), filtering on `linktype IN ('LC','LU','LS')` and `linkprim IN ('C','P')`, with date-range validation to handle firm identity changes over time
- **ESG ↔ Compustat:** Extracted 8-digit CUSIP from ISIN using string slicing (`isin[2:10]`) and matched to Compustat's `cusip` field; used `wrds_ref_esg` as the bridge table
- **Buy-and-Hold Returns:** Computed 12-month BHR starting 4 months after fiscal year-end (`t+4` convention) to respect information availability and avoid look-ahead bias

```python
# Temporal CCM link validation
ccm_link = conn.raw_sql("""
    SELECT gvkey, lpermno AS permno, linktype, linkprim,
           linkdt, linkenddt
    FROM crsp.ccmxpf_linktable
    WHERE linktype IN ('LC','LU','LS')
      AND linkprim IN ('C','P')
""")
```

---

### 3. Feature Engineering & Panel Construction (`Step 6` in notebook)

**Skills:** Pandas, time-series feature engineering, winsorization, panel data structuring

- Constructed all regression variables: ROA, operating ROA, BHR, book-to-market, leverage, R&D intensity, sales growth, earnings news proxy
- Applied **winsorization at 1st/99th percentiles** for all continuous variables to limit outlier influence (industry-standard in empirical finance)
- Generated **lagged variables** (`esg_lag`, `lagged_roa`) with proper panel alignment by `gvkey` and `fyear`
- Assigned **Fama-French 12-industry classification** using SIC codes
- Final panel: 18,443 firm-years, 3,086 unique firms, fiscal years 2013–2023

---

### 4. Exploratory Data Analysis & Visualization (`Part I` in notebook)

**Skills:** matplotlib, seaborn, statistical visualization, financial data storytelling

- ESG score distributions by year and industry (bar + line plots)
- Pillar-level trend analysis (E/S/G separately over time)
- Industry-level ESG heatmaps and radar charts
- Identified and explained a **composition effect** behind the 2013–2017 ESG decline (dataset expansion, not performance deterioration)

---

### 5. Determinants of ESG — OLS Regression (`Part II` in notebook)

**Skills:** statsmodels, OLS regression, fixed effects, omitted variable analysis, coefficient interpretation

- Estimated OLS regressions with progressively richer specifications using `statsmodels.formula.api`
- Demonstrated **omitted variable bias**: profitability flips sign (positive → negative) and R&D coefficient jumps from near-zero to −11.0 once industry/year fixed effects are added
- Used heteroskedasticity-robust (HC1) standard errors throughout
- Complemented regressions with **quartile analysis** and **scatter plots** to visualize non-linear relationships

| Variable | M1 (No FE) | M2 (+Year & Industry FE) |
|---|---|---|
| Firm size | +6.68*** | +7.11*** |
| R&D intensity | +0.16 | −11.00*** |
| Leverage | +5.91*** | +2.87*** |
| Book-to-market | +4.53*** | +4.40*** |

---

### 6. ESG → ROA Regressions (`Part III-A` in notebook)

**Skills:** Panel OLS, firm fixed effects, earnings persistence modeling, causal inference thinking

- Four-model sequence from unconditional to firm-FE specification
- Used `linearmodels.panel.PanelOLS` for within-firm estimation
- Key result: strong unconditional ESG–ROA correlation (R² = 0.054 → 0.509) is fully absorbed by firm controls, confirming ESG as a **proxy for firm quality** rather than an independent earnings driver
- Robustness check using **operating ROA** (less sensitive to accruals)

---

### 7. ESG → Stock Returns Regressions (`Part III-B` in notebook)

**Skills:** Asset pricing regression, momentum controls, cross-sectional vs. within-firm identification

- Documented a **sign reversal** in the ESG coefficient after controlling for size: unconditional coefficient is negative (large-firm bias), conditional coefficient is positive (+0.0008, p=0.002)
- Controlled for earnings news (∆ROA as proxy for earnings surprise) and prior 12-month momentum
- Firm FE specification showed ESG operates as a **cross-sectional sorting variable**, not a dynamic timing signal — a practically important distinction for portfolio construction

---

### 8. Industry-Level Double Materiality Analysis (`Part IV` in notebook)

**Skills:** Grouped aggregation, quadrant classification, bubble charts, investment implication framing

- Computed industry-level means across ESG pillars, BHR, and ROA using the FF12 classification
- Built a **double materiality quadrant** (ESG × returns, ESG × ROA) using cross-industry medians as thresholds
- Decomposed composite ESG into E/S/G pillars to distinguish structural from coincidental alignment
- Conclusion: **Shops** is the only double materiality winner on both dimensions; HiTec's alignment is momentum-driven rather than structurally grounded

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Python 3.13 | Core language |
| `wrds` | WRDS database connection & SQL queries |
| `pandas` / `numpy` | Data wrangling, panel construction |
| `statsmodels` | OLS regression, HC1 robust SEs |
| `linearmodels` | Panel OLS with firm/year fixed effects |
| `matplotlib` / `seaborn` | Visualization |
| `scipy.stats` | Descriptive statistics |
| WRDS (PostgreSQL) | Cloud database: CRSP, Compustat, Refinitiv ESG |

---

## How to Run

> **Note:** Live WRDS database queries require institutional access credentials. The final merged panel is provided as `esg_panel.csv` for reproducibility without a WRDS subscription.

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/esg-financial-returns.git
cd esg-financial-returns

# 2. Install dependencies
pip install wrds pandas numpy matplotlib seaborn statsmodels linearmodels scipy

# 3. (Optional) Configure WRDS credentials for live queries
# The notebook will prompt for your WRDS username/password

# 4. Open the notebook
jupyter notebook Final_Project_Code.ipynb
```

To skip database queries and use the pre-built panel directly, jump to the **"Load Panel Data"** cell in the notebook which reads from `esg_panel.csv`.

---

## Relevance to Data Analytics in Fintech & Banking

This project mirrors workflows common in **quantitative research, risk analytics, and data science** roles in financial services:

- **Multi-source data integration** from institutional financial databases (CRSP, Compustat, Refinitiv) — the same sources used at buy-side firms, investment banks, and risk teams
- **Panel econometrics** with fixed effects — standard methodology for credit risk modeling, factor research, and performance attribution
- **Bias identification and correction** (omitted variable bias, look-ahead bias, survivorship bias) — critical for any production-grade analytical model
- **Financial materiality framing** — directly applicable to ESG integration, sustainable finance mandates (CSRD, SFDR), and credit risk under climate transition scenarios
- **Cross-sectional return prediction** — foundational to quantitative equity strategies and factor investing

---

## References

Selected academic references used in the analysis:

- Berg, F., Koelbel, J.F., and Rigobon, R. (2022). Aggregate Confusion: The Divergence of ESG Ratings. *Review of Finance*, 26(6), 1315–1344.
- Bolton, P. and Kacperczyk, M. (2021). Do Investors Care about Carbon Risk? *Journal of Financial Economics*, 142(2), 517–549.
- Khan, M., Serafeim, G., and Yoon, A. (2016). Corporate Sustainability: First Evidence on Materiality. *The Accounting Review*, 91(6), 1697–1724.
- Sloan, R.G. (1996). Do Stock Prices Fully Reflect Information in Accruals and Cash Flows about Future Earnings? *The Accounting Review*, 71(3), 289–315.

---

*Group project completed for RSM 8224, Rotman School of Management, University of Toronto, Winter 2026.*