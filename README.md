# Collection & Financial Risk Intelligence — Power BI Dashboard

> An enterprise-grade Power BI dashboard that gives a real estate developer selling units via installment plans a single, reconciled view of sales, cash collections, and banking exposure across an 8-project residential portfolio.

**Client:** Memaar Almorshedy — 8-project residential portfolio worth EGP 18.64B
**Type:** Data Analysis Diploma — Graduation Project
**Tool:** Microsoft Power BI (Power Query, DAX, HTML Content visual)

---

## 📌 Table of Contents

- [Business Problem](#-business-problem)
- [Business Value](#-business-value)
- [Tech Stack](#-tech-stack)
- [Data Sources](#-data-sources)
- [Data Model](#-data-model)
- [ETL / Data Preparation](#-etl--data-preparation)
- [DAX Measures](#-dax-measures)
- [Report Architecture (35 Pages)](#-report-architecture-35-pages)
- [Key Insights](#-key-insights)
- [Data Validation](#-data-validation)
- [Repository Contents](#-repository-contents)
- [Screenshots](#-screenshots)

---

## 🧩 Business Problem

Memaar Almorshedy manages 8 residential projects worth over **EGP 18.6 billion**, sold entirely through installment plans. Despite this scale, the company had no unified, reliable view of collection performance — each project lived in its own spreadsheet, disconnected from the rest, making *"how much have we actually collected?"* and *"where is the risk?"* a slow, manual, error-prone exercise.

**Core challenges addressed:**

| # | Challenge | Impact |
|---|-----------|--------|
| 1 | Fragmented, unreconciled data | 8 separate files, no single source of truth |
| 2 | Hard-to-track installment lifecycle | No clear paid / due / not-due classification |
| 3 | Unmonitored financial risk | EGP 791M+ outstanding across cash & 10 banks |
| 4 | Weak portfolio-to-detail navigation | No fast path from KPIs to a single project |

---

## 💡 Business Value

- **Faster decisions** — one report replaces manually reconciling 8 spreadsheets
- **Early risk detection** — underperforming projects surface immediately
- **Stronger bank relationships** — clear per-bank exposure supports negotiation
- **Fair project comparison** — every project measured on identical KPI definitions

---

## 🛠 Tech Stack

- **Power BI Desktop** — data modeling, DAX, report design
- **Power Query (M)** — ETL, unpivoting, custom columns
- **DAX** — all business logic and measures
- **HTML Content visual** — custom-styled tables (not native Table/Matrix), for full control over the dark-navy-and-gold brand palette and color-coded status pills

---

## 📂 Data Sources

Two Excel workbooks:

1. **`Graduation_Project_Data_Source.xlsm`** — 8 worksheets (one per project), 21 identical columns each: customer identification, unit price, payment schedule, 7 installment cells, total due/paid, bank, dates. Each sheet ends with 3 control rows (`Summary Total`, `Paid Installments`, `Remaining Installments`).
2. **`Morshedy_8_Projects_Advertised_Total_Units.xlsx`** — fixed, adopted total unit count per project (the denominator for every `Sold %` calculation).

---

## 🗂 Data Model

A simple star schema:

```
                     Projects
                        │
                    Fact_Sales  ◄────── (grain: customer × project)
                    ╱        ╲
            Installments   DateSlicerTable
     (grain: customer ×      (independent
        installment)          calendar table)
```

| Table | Grain | Role |
|---|---|---|
| **Fact_Sales** | 1 row per customer/unit, per project | Central fact table — all 8 sheets appended |
| **Projects** | 1 row per project | Dimension — fixed Adopted Total Units + display order |
| **Installments** | 1 row per installment (1–7), per customer | Unpivoted from Fact_Sales; drives Overall/Cash/Bank measures |
| **DateSlicerTable** | 1 row per calendar date (2024–2027) | Independent calendar for the OLD/NEW date-range split |

**Relationships:** `Projects[Project Name]` 1 → ∞ `Fact_Sales[Project]`, `DateSlicerTable` 1 → ∞ `Fact_Sales`.

---

## 🔧 ETL / Data Preparation

1. **Get Data** on all 8 project worksheets → Power Query
2. Remove the 3 control rows from every sheet (not customer records)
3. Add a custom column per sheet holding the project name (fixed text)
4. Clean stray column-name spaces (Rihana sheet)
5. Remove a duplicate column (`POC` vs. "engineering completion %" — kept `POC`)
6. Standardize data types across every column, every sheet
7. **Append Queries as New** → unified table `Fact_Sales`
8. Import & clean the Adopted Units file → dimension table `Projects` (+ Index column for display order)
9. Build `Installments` via Reference → Unpivot Columns (P1…P7) → classify into 3 business states:

```m
State (Custom Column)
= if Text.Contains([Raw Value], "تم السداد") then "PAID"
  else if Text.Contains([Raw Value], "مستحقة") then "NOT_DUE"
  else "DUE_NOT_PAID"
```

10. Standardize bank names (`Replace Values` for duplicate spellings) + add `Bank Name EN` translation column for the Bank-wise page

---

## 📐 DAX Measures

All measures were written **once** and reused across every page that needs them — filtered per page, not duplicated per project ("write once, filter per page").

**Sample — Global Summary:**

```dax
# Sold = COUNTROWS(Fact_Sales)
Sold % = DIVIDE([# Sold], [# Units])
Invoiced Amount = [Amount Collected] + [Outstanding Balance]

Bank Outstanding =
CALCULATE(
    [Outstanding Balance],
    Fact_Sales[البنك] <> "كاش",
    NOT ISBLANK(Fact_Sales[البنك])
) + 0
```

**Sample — Installment lifecycle (Overall / Cash / Bank pages):**

```dax
Issued Invoices =
CALCULATE(
    COUNTROWS(Installments),
    Installments[State] <> "NOT_DUE"
)

Collection Rate = DIVIDE([Collected Invoices], [Issued Invoices])
```

Full measure list, including Collection-By-Date OLD/NEW logic and Bank-wise measures, is documented in [`docs/Technical_Details.pdf`](./docs/Technical_Details.pdf).

---

## 🏗 Report Architecture (35 Pages)

```
Home / Navigation (1 page)
│
├── Global Summary
├── Collection By Date Summary
│
└── 8 Projects × 4 pages each (32 pages)
    ├── Overall     — all customers, every payment method
    ├── Cash        — cash customers only
    ├── Bank        — bank-financed customers only
    └── Bank-wise   — exposure broken down per financing bank
```

Every analytical page shares a persistent **Home** button, **Project Completion %** indicator, and **Units** card. All tables are rendered via a custom **HTML Content** visual (not native Power BI tables) to achieve the brand-accurate dark-navy-and-gold palette with color-coded status pills.

---

## 📈 Key Insights

- Portfolio collection rate: **94.3%** (EGP 13.09B collected of EGP 13.88B invoiced)
- Sales absorption is highly uneven: **0.08%** (Zahra North Coast) to **51.07%** (Rihana)
- Bank exposure (**EGP 522M**) is nearly double cash exposure (**EGP 269M**)
- Cash sales collect marginally *better* than bank-financed sales (95.0% vs. 93.8%)
- Highest risk concentration: **Skyline Katamya Compound** (EGP 198M outstanding)

Full project-by-project insights and recommendations are covered in the companion presentation (32 detail pages, one set of insights per project × page combination).

---

## ✅ Data Validation

Every project reconciles exactly against two required identities:

```
Invoiced Amount        = Amount Collected + Outstanding Balance
Cash Outstanding + Bank Outstanding = Outstanding Balance
```

Confirmed for all 8 projects and the portfolio total — see the Validation Proof section in the companion presentation / technical details PDF.

---

## 👤 Credits

Built as a graduation project for the **Data Analysis Diploma**, based on the business case and dataset supplied for Memaar Almorshedy's 8-project residential portfolio.

---

## 👤 Author

**Sabry Elzeftawy**  
*Computer Science Student | Data Analyst | Aspiring Data Engineer*

- 🤝 **LinkedIn:** [Sabry Elzeftawy]([https://linkedin.com](https://www.linkedin.com/in/sabry-elzeftawy-16bb23389?utm_source=share&utm_campaign=share_via&utm_content=profile&utm_medium=android_app))

---

⭐ *If you find this project helpful, please give it a star!*
