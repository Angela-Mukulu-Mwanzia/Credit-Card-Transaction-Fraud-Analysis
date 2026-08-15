# Power BI build specification

This is the reproducible technical specification for `powerbi/Credit_Card_Transactions.pbix`. Complete it in Power BI Desktop, then save the finished PBIX in the `powerbi` folder.

## 1. Import and staging query

1. Open Power BI Desktop, select **Get Data > Text/CSV**, and choose `Credit Card Transactions Dataset/credit_card_transactions.csv`.
2. Choose **Transform Data**, rename the query `Stg_Transactions_Raw`, and turn off **Enable load** after the dependent queries are created.
3. Promote headers. Set types: `trans_date_trans_time` Date/Time; `amt`, `lat`, `long`, `merch_lat`, `merch_long` Decimal Number; `zip`, `city_pop`, `is_fraud` Whole Number; all labels/text fields Text.
4. Remove `Unnamed: 0`. Before removing `cc_num`, add `CustomerID` using `Text.From([cc_num])`. Before removing `trans_num`, rename it to `TransactionID`; use it to remove duplicates.
5. Filter `amt` to non-null values greater than zero, then rename it `Amount`. Rename `is_fraud` to `IsFraud`, `category` to `Transaction Type`, `city` to `City`, and `state` to `State`.
6. Create `MerchantName`: `Text.Proper(Text.Trim(Text.Replace([merchant], "fraud_", "")))`. Create a clean `Transaction Type` with underscore replaced by a space and Proper Case.
7. Create `DateKey = Date.From([trans_date_trans_time])`, `Time = Time.From([trans_date_trans_time])`, `Hour = Time.Hour([Time])`, `Year = Date.Year([DateKey])`, `Month Number = Date.Month([DateKey])`, `Month Name = Date.MonthName([DateKey])`, and `Day Name = Date.DayOfWeekName([DateKey])`.
8. Standardise `Gender`: uppercase/trim, map M to Male, F to Female, else Unknown. Trim `City` and uppercase/trim `State`.
9. Remove duplicates on `TransactionID`. Remove fields no longer needed for reporting: `cc_num`, `first`, `last`, `street`, `job`, `dob`, `unix_time`, `merchant`, and any unused raw location fields. Keep the relevant fields until all reference queries are created.

## 2. Build the star schema in Power Query

### DimCustomer

1. Right-click `Stg_Transactions_Raw` and select **Reference**. Rename it `DimCustomer`.
2. Retain `CustomerID`, `Gender`, `City`, `State`, `zip`, `lat`, `long`. Rename `zip` to `Zip`, `lat` to `Customer Latitude`, and `long` to `Customer Longitude`.
3. Remove duplicates on `CustomerID`; confirm one row per key. Keep this table loaded.

### DimMerchant

1. Create a second reference named `DimMerchant`.
2. Keep `MerchantName`, `merch_lat`, and `merch_long`; remove duplicates on `MerchantName`.
3. Add an index from 1 named `MerchantID`. Rename coordinates to `Merchant Latitude` and `Merchant Longitude`. Keep this table loaded.

### FactTransactions

1. Create another reference named `FactTransactions`.
2. Merge it with `DimMerchant` on `MerchantName` using a Left Outer join and expand only `MerchantID`.
3. Retain `TransactionID`, `CustomerID`, `MerchantID`, `DateKey`, `Time`, `Hour`, `Transaction Type`, `Amount`, and `IsFraud`. Do not retain names, card account number or street address.
4. Set `DateKey` to Date, `Amount` to Fixed Decimal Number/Decimal Number, and key/count columns to Whole Number/Text as appropriate. Keep this table loaded.

Select **Close & Apply** and resolve errors before continuing. Verify that fact row count is sensible and that no duplicate dimension keys exist.

## 3. Create the date table and relationships

1. In **Modeling > New table**, create `DimDate` with the DAX below. Create the table only after `FactTransactions` has loaded.

```DAX
DimDate =
ADDCOLUMNS (
    CALENDAR ( MIN ( FactTransactions[DateKey] ), MAX ( FactTransactions[DateKey] ) ),
    "Year", YEAR ( [Date] ),
    "Quarter", "Q" & FORMAT ( [Date], "Q" ),
    "Month Number", MONTH ( [Date] ),
    "Month Name", FORMAT ( [Date], "MMMM" ),
    "Year-Month", FORMAT ( [Date], "YYYY-MM" ),
    "Year-Month Sort", YEAR ( [Date] ) * 100 + MONTH ( [Date] ),
    "Day Number", DAY ( [Date] ),
    "Day Name", FORMAT ( [Date], "dddd" ),
    "Day of Week Number", WEEKDAY ( [Date], 2 )
)
```

2. Sort `DimDate[Month Name]` by `Month Number`, `DimDate[Day Name]` by `Day of Week Number`, and `Year-Month` by `Year-Month Sort`.
3. Mark `DimDate` as a date table using `[Date]`.
4. In Model view create these active, one-to-many, single-direction relationships: `DimCustomer[CustomerID]` to `FactTransactions[CustomerID]`; `DimMerchant[MerchantID]` to `FactTransactions[MerchantID]`; `DimDate[Date]` to `FactTransactions[DateKey]`.
5. Hide technical keys and sort columns from Report view. Create a separate Measures table if desired and paste the formulas from `DAX_Measures.txt`.

## 4. Build report pages

### Executive Overview

Use a 16:9 canvas. Put four KPI cards in the top row. Add the monthly sales line chart across the centre, category sales bar chart lower-left, State sales map lower-right, and a tidy slicer column. Format monetary values as currency, titles in navy and one primary accent colour.

### Customer & Merchant Analysis

Place gender sales bar and merchant treemap across the top. Put the customer transaction-count versus average-value scatter plot in the centre. Place a Top 10 customer table at the bottom with a Top N 10 visual filter by Total Sales Amount. Add Year, Month and Gender slicers.

### Fraud & Anomaly Review

Use a clearly labelled warning colour only for risk content. Place fraud/flag KPIs across the top. Add daily sales and control-limit lines, an Hour sales chart, and a detailed transaction table filtered to `Potential Fraud Flag = 1`. Include text/tooltip explaining that a flag is a screening signal, not a confirmed fraud decision.

## 5. Verify before saving

- Refresh completes with no errors.
- Dimension-to-fact relationship arrows are single-direction and dimensions have unique keys.
- `DimDate` is marked as a date table and date measures return values where comparable prior-year periods exist.
- Currency, percentage and count formats are correct.
- No raw card numbers, street addresses or personal names appear in visuals or loaded model.
- All report pages cross-filter as intended and contain useful titles, accessible contrast and slicers.
- Capture the seven screenshots listed in `Screenshots/README.md`, then save the PBIX as `powerbi/Credit_Card_Transactions.pbix`.
