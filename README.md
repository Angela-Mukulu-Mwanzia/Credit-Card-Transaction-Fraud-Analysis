# Credit Card Transactions - Power BI Business Intelligence Solution

> **Repository name:** `DSA3050-PowerBI-ANGELAMWANZIA-663887`  

## 1. Introduction

This project develops an end-to-end Power BI solution for analysing credit card transaction activity. It moves from a raw, high-volume transaction CSV through Power Query cleaning, a star-schema data model, DAX calculations, and three interactive report pages. The report helps decision-makers understand spending performance, customer and merchant behaviour, and transactions requiring further fraud review.

**Important scope note.** The supplied CSV contains **1,296,675 rows** and covers **1 January 2019 to 21 June 2020**. The Kaggle listing may describe a broader/original release with a different row count; the values in this report describe the exact file included with this submission.

# PowerBI Dashboard Link
- https://drive.google.com/drive/folders/1ywKrBtt-aImh2ip42iWVhdWl9xr7eyaS?usp=drive_link

## 2. Section A - Dataset selection and understanding

### Source and representation

The data is the [Credit Card Transactions Dataset on Kaggle](https://www.kaggle.com/datasets/priyamchoksi/credit-card-transactions-dataset). It records individual card transactions, including transaction timestamp, amount, merchant, category, customer attributes, customer location, merchant coordinates, and an `is_fraud` label. Direct identifiers are excluded from the analytical model.

### Why this dataset was selected

It is a credible, real-world-style, transaction-level dataset with sufficient scale and complexity for meaningful BI work. The supplied file has 1.30 million records, 14 transaction categories, 51 states, 983 cardholders and 693 merchants. It combines numeric, categorical, geographic and date/time fields, supports financial KPIs, and includes a fraud outcome for diagnostic investigation.

### Main variables

| Raw field | Type | Analytical use |
|---|---|---|
| `trans_date_trans_time` | Date/time | Trends, time-of-day and date filtering |
| `amt` | Decimal number | Sales, average transaction value and anomaly rules |
| `merchant` | Text | Merchant performance and investigation |
| `category` | Text | Spending-category analysis |
| `gender` | Text | Customer segmentation |
| `city`, `state`, `zip` | Text/number | Geographic and regional analysis |
| `cc_num` | Whole number | Used only to create a surrogate customer key; removed thereafter |
| `is_fraud` | Whole number (0/1) | Fraud-rate and flagged-transaction analysis |
| `lat`, `long`, `merch_lat`, `merch_long` | Decimal number | Customer/merchant location analysis |

### Verified initial profile

The raw file totals **$91.22M** in transaction value, with an average transaction value of **$70.35**. It contains **7,506 labelled fraud records (0.579%)**. These profiling figures should be refreshed after transformations if data-cleaning rules remove records.

### Business problem

The business needs to understand where and how customers spend, identify valuable customer and merchant segments, and isolate unusual transactions for fraud review. The solution must distinguish normal variation in purchasing from transactions that warrant investigation, without treating an analytical flag as proof of fraud.

### Analytical questions

1. How do transaction count and transaction value change by month and year?
2. Which states, spending categories and merchants contribute the most transaction value?
3. How do spend, transaction frequency and average ticket differ by customer gender and location?
4. Which customers are highest value, and how concentrated is spend among them?
5. When do transactions occur, and are there unusual off-hours patterns?
6. Which transactions are statistically unusual for the customer and should be prioritised for investigation?

## 3. Section B - Power Query transformation

The query design uses `Stg_Transactions_Raw` as a load-disabled staging query. `FactTransactions`, `DimCustomer` and `DimMerchant` are references of this query. This prevents repeatedly reading the large CSV and keeps transformations reproducible.

| # | Problem | Transformation | Reason | Result |
|---:|---|---|---|---|
| 1 | Headers and inferred types cannot be trusted in a large CSV. | Promote headers; explicitly set date/time, decimal, whole-number and text types. | Prevents aggregation and relationship errors caused by accidental text types. | Reliable types for every analytical field. |
| 2 | `Unnamed: 0` is an export index, not a business field. | Remove `Unnamed: 0`. | It adds no analytical value and increases model size. | Leaner staging query. |
| 3 | Names, street, job, date of birth, card number, Unix time and raw transaction hash are not required in the dashboard. | Remove `first`, `last`, `street`, `job`, `dob`, `unix_time`, `trans_num`, `cc_num` after creating `CustomerID`, plus unused postal/merchant-coordinate fields where mapping is not used. | Reduces exposure of personal data and improves refresh/model performance. | Privacy-conscious analytical model. |
| 4 | The same person needs a stable relationship key, but card number must not be loaded. | Create `CustomerID` in staging with a deterministic text key from `cc_num`; retain it only as a surrogate key. | Enables a customer dimension without displaying sensitive account data. | `CustomerID` supports a one-to-many relationship. |
| 5 | Merchant names include a `fraud_` prefix introduced by the synthetic source. | Add `MerchantName = Text.Proper(Text.Trim(Text.Replace([merchant], "fraud_", "")))`; use this for reporting. | Gives business-readable merchant labels and avoids showing source artefacts. | Clean merchant names in visuals. |
| 6 | Categories use underscores, e.g. `grocery_pos`. | Replace `_` with space and apply proper case in `Transaction Type`. | Produces readable, consistently grouped categories. | Labels such as `Grocery Pos`. |
| 7 | A single timestamp cannot directly support date hierarchy and hour analysis. | Create `DateKey = Date.From([trans_date_trans_time])`, `Time`, `Hour`, `Year`, `Month Number`, `Month Name`, and `Day Name`. | Supports time intelligence, date filtering and off-hours diagnostics. | Flexible calendar and time-of-day analysis. |
| 8 | Empty/invalid or non-positive amounts would distort financial KPIs. | Filter `amt` to non-null values greater than 0; rename it `Amount`. | A payment value must be positive for this spend analysis. | Valid monetary base for measures. |
| 9 | Duplicate imports could double-count a transaction. | Remove duplicates using the original transaction hash (`trans_num`) before it is removed. | The transaction hash is the safest available source-row identifier. | One fact row per source transaction. |
| 10 | Gender may need to be presented consistently. | Trim and uppercase `gender`; map `M` to `Male`, `F` to `Female`, else `Unknown`. | Prevents fragmented demographic groups and preserves unexpected values. | A consistent `Gender` attribute. |
| 11 | Address-level data is too granular and privacy-sensitive for this report. | Retain and clean `City`, `State`, and `Zip`; remove street address. | Enables regional analysis while minimising personal data. | Geographic filters at sensible reporting grain. |
| 12 | The flat source is inefficient for repeated descriptive values. | Create reference queries for customer and merchant dimensions, keep distinct rows, add index surrogate IDs where needed. | Produces a star schema and reduces repeated text in the fact table. | Reusable dimensions joined to the central fact. |

### Power Query implementation notes

The full click-by-click build is in [PowerBI_Build_Specification.md](PowerBI_Build_Specification.md). Key quality checks before **Close & Apply** are:

- Confirm `Amount` has no null or non-positive values.
- Confirm `DateKey` is a Date (not Date/Time).
- Confirm one row per `TransactionID`/`trans_num` before that source field is removed.
- Confirm `DimCustomer[CustomerID]` and `DimMerchant[MerchantID]` contain no duplicates.
- Preserve `is_fraud` as a whole number, not True/False text.

## 4. Section C - Data model

The final model is a star schema with `FactTransactions` at the centre. Its grain is **one valid source transaction**. It stores transaction keys, foreign keys, amount, the fraud label, date key and hour. Descriptive attributes are stored once in dimensions rather than repeatedly in the fact table.

| Table | Purpose and key fields |
|---|---|
| `FactTransactions` | Transaction facts: `TransactionID`, `CustomerID`, `MerchantID`, `DateKey`, `Amount`, `IsFraud`, `Hour`, and `Transaction Type`. |
| `DimCustomer` | Unique customer characteristics: `CustomerID`, `Gender`, `City`, `State`, `Zip`, customer latitude and longitude. |
| `DimMerchant` | Unique merchants: `MerchantID`, `Merchant Name`, merchant latitude and longitude. |
| `DimDate` | Continuous calendar from the minimum to maximum `DateKey`: Date, Year, Quarter, Month Number, Month Name, Year-Month, weekday and day number. |

### Relationships and design decisions

| From | To | Cardinality | Cross-filter direction | Active |
|---|---|---|---|---|
| `DimCustomer[CustomerID]` | `FactTransactions[CustomerID]` | One-to-many | Single (dimension to fact) | Yes |
| `DimMerchant[MerchantID]` | `FactTransactions[MerchantID]` | One-to-many | Single (dimension to fact) | Yes |
| `DimDate[Date]` | `FactTransactions[DateKey]` | One-to-many | Single (dimension to fact) | Yes |

Single-direction relationships avoid ambiguous paths and make filter behaviour predictable. `DimDate` is marked as the model date table using `DimDate[Date]`, enabling time-intelligence calculations. The main modelling challenge is that the source does not provide safe business dimension keys. This is resolved by creating deterministic `CustomerID` from the card number in the staging query and a `MerchantID` from a distinct merchant list, then removing the card number from all loaded tables.

## 5. Section D - DAX and business calculations

All measure definitions are in [DAX_Measures.txt](DAX_Measures.txt). The table below documents six priority measures.

| Measure | What it calculates and why it is useful | DAX/context behaviour | Dashboard use |
|---|---|---|---|
| `Total Sales Amount` | Sum of valid transaction value; the primary performance KPI. | `SUM` respects every current date, category, state, merchant and customer filter. | Executive KPI, trend, map and category chart. |
| `Total Customers` | Number of distinct active customer keys. Shows customer reach rather than transaction frequency. | `DISTINCTCOUNT` changes with the current fact-table filter context. | Executive KPI and customer segmentation. |
| `High-Value Transactions` | Counts transactions above the selected business threshold ($100). Highlights larger exposures. | `CALCULATE` changes the filter context by adding `Amount > 100`. Existing slicers remain active. | Executive and diagnostic cards. |
| `Fraud Rate %` | Share of transactions carrying the supplied fraud label; supports monitored risk levels. | `DIVIDE` safely handles zero transaction contexts; numerator and denominator react to slicers. | Diagnostic KPI and category/state comparison. |
| `YoY Sales Growth %` | Current-period sales change against the equivalent period last year. | `SAMEPERIODLASTYEAR` requires the marked, continuous date table; it preserves non-date filters such as state. | Executive trend tooltip/KPI. |
| `Potential Fraud Transactions` | Counts transactions exceeding a customer-specific statistical threshold. It prioritises review rather than confirming fraud. | `SUMX` iterates transactions; `CALCULATE`, `ALLEXCEPT`, `AVERAGEX` and `STDEVX.P` construct a customer-level benchmark. Date/category filters remain in the evaluated visual context. | Diagnostic KPI and flagged-transaction table. |

## 6. Section E - Dashboard design and storytelling

The report progresses from overall performance to driver analysis to investigation. A consistent palette is recommended: navy `#17365D` (headings), teal `#00A6A6` (positive/primary), amber `#F4B942` (attention) and red `#C62828` (risk). Use Segoe UI, white canvas, subtle grey containers, aligned grid spacing and no unnecessary chart borders.

### Page 1 - Executive Overview: “What happened?”

- Four KPI cards: Total Transactions, Total Sales Amount, Average Transaction Value, Total Customers.
- Monthly line chart: Total Sales Amount by `DimDate[Year-Month]` (sort by a numeric Year-Month key).
- Clustered bar chart: Total Sales Amount by `Transaction Type`.
- Filled map or bubble map: Total Sales Amount by `DimCustomer[State]`.
- Slicers: Year, Month Name, Transaction Type and State.
- Add a dynamic title and a report-page tooltip with Sales, Transactions and Fraud Rate.

### Page 2 - Customer & Merchant Analysis: “Who and what drove it?”

- Bar chart: Total Sales Amount by Gender.
- Treemap: Total Sales Amount by Merchant Name (apply a Top N 15 filter for legibility).
- Scatter chart: customer `CustomerID` in Details, Total Transactions on X, Average Transaction Value on Y, and Total Sales Amount as size.
- Table: Top 10 customers by sales, with CustomerID, State, Total Sales Amount, Total Transactions and Customer Rank.
- Slicers: Year, Month Name and Gender. Use drill-through from a customer selection to the diagnostic page.

### Page 3 - Fraud & Anomaly Review: “What needs attention?”

- KPI cards: Fraud Rate %, Potential Fraud Transactions and Potential Fraud Amount.
- Line chart: Total Sales Amount with `Upper Sales Control Limit` and `Lower Sales Control Limit` by Date; use the Analytics pane anomaly detection where available.
- Column chart: Total Sales Amount by Hour, coloured by `Potential Fraud Transactions` or accompanied by a risk tooltip.
- Transaction table: Date/Time, CustomerID, Merchant Name, Transaction Type, Amount, IsFraud, Customer Average Amount and Potential Fraud Flag. Apply a visual filter where `Potential Fraud Flag = 1`.
- Slicers: Date, Transaction Type and Merchant Name. Use the transaction table as a drill-through destination where practical.

### Interactivity and accessibility

All charts should cross-filter by default; check **Edit interactions** so maps and category charts filter instead of merely highlighting where appropriate. Sync Year and Month slicers between the first two pages. Add concise alt text, high-contrast data labels, currency formatting, and an information button/bookmark explaining that anomaly flags require analyst review.

## 7. Business insights and actions to validate in Power BI

Initial profiling suggests that large states such as TX, NY and PA account for the highest total spend, while `gas_transport`, `grocery_pos` and `home` have the largest transaction counts. These are starting hypotheses, not final conclusions: the dashboard should validate them after cleaning and use rate-based measures before claiming a segment is riskier.

Recommended actions:

1. Focus retention and offer analysis on high-value customer segments, but compare average ticket and frequency together to avoid rewarding one-off spend.
2. Review category and hour combinations with unusually high customer-relative amounts, especially when the potential-fraud flag and supplied label agree.
3. Compare fraud rate (not just fraud count) across states, merchants and categories; volume-heavy segments can otherwise appear riskier merely because they have more transactions.
4. Investigate material departures above the upper control limit before operational escalation.

## 8. Section F - repository evidence and submission checklist

```text
DSA3050-PowerBI-YourName-YourRegNo/
|- README.md
|- DAX_Measures.txt
|- PowerBI_Build_Specification.md
|- Credit Card Transactions Dataset/
|  |- credit_card_transactions.csv
|- Dataset/
|  `- README.md
|- Screenshots/
|  `- README.md
`- powerbi/
   `- README.md
```

The raw CSV is retained in `Credit Card Transactions Dataset/` and documented in `Dataset/README.md`; avoid duplicating a very large file solely for repository layout. An existing PBIX named `Credit Card Transaction 2.pbix` has been retained unchanged at the repository root. After validating it against this specification, rename/move it to `powerbi/Credit_Card_Transactions.pbix` before final submission.

### Required screenshots

| File | What it must show |
|---|---|
| `Screenshots/01_Raw_Data.png` | CSV preview with headers and row/column profile. |
| `Screenshots/02_PowerQuery_Transformations.png` | Power Query Editor, applied steps and transformed fields. |
| `Screenshots/03_Data_Model.png` | Model View with all four tables, key fields and one-to-many single-direction relationships. |
| `Screenshots/04_DAX_Measures.png` | Measures pane and a representative measure formula. |
| `Screenshots/05_Dashboard_Overview.png` | Completed Executive Overview with slicers visible. |
| `Screenshots/06_Dashboard_Detailed.png` | Completed Customer & Merchant Analysis. |
| `Screenshots/07_Dashboard_Diagnostic.png` | Completed Fraud & Anomaly Review with flagged table. |


