# Power BI: Multistate Tax Filing Dashboard (step-by-step)

**Goal:** turn raw filing data into a 1-page dashboard of filing status, overdue risk and tax exposure by state. This is the "raw data → insights" work that Deloitte Tax Technology does.

**Time:** about 3–4 hours. **Tool:** [Power BI Desktop](https://www.microsoft.com/power-bi/desktop) (free, Windows).

**Data (fictional):** `data/Clients.csv` (16), `data/Jurisdictions.csv` (8), `data/TaxFilings.csv` (189 rows).

`TaxFilings.csv` is **deliberately messy**, so you get real data-wrangling practice:
- 10 jurisdiction codes in lowercase or with trailing spaces
- 8 status values in UPPERCASE
- 6 duplicate rows

> **Option B:** if you built TaxTrack, use **Get Data → SQL Server → `TaxTrack` → `vw_FilingReport`** instead of the CSVs. That's the same data, already joined.

---

## 1. Load the data
**Home → Get data → Text/CSV**, then load all three files and click **Transform Data** (not Load).

## 2. Clean it in Power Query (data wrangling)
In the `TaxFilings` query:

1. **JurisdictionCode:**
   - right-click → **Transform → Trim**
   - then **Transform → UPPERCASE**
2. **Status:** **Transform → Capitalize Each Word**.
   - `INREVIEW` becomes `Inreview`. Fix it with **Replace Values**: `Inreview` → `InReview`.
3. **Remove duplicates:** select all columns → **Home → Remove Rows → Remove Duplicates**. Check that the row count drops from 189 to 183.
4. **Data types:**
   - `DueDate` and `FiledDate` → Date (blanks become null)
   - `TaxDue` → Fixed decimal
   - `TaxYear` → Whole number
5. **Add Column → Custom Column** `IsFiled`:
   ```m
   = if [FiledDate] = null then 0 else 1
   ```
6. **Close & Apply.**

You can see each step under **Applied Steps**. That list is your audit trail, so mention it in interviews.

## 3. Data model (Model view)
Make a star schema, with `TaxFilings` as the fact table:
- `Clients[ClientId]` 1 → * `TaxFilings[ClientId]`
- `Jurisdictions[JurisdictionCode]` 1 → * `TaxFilings[JurisdictionCode]`

Add a **date table**: **Modeling → New table**
```DAX
Calendar = CALENDAR(DATE(2024,1,1), DATE(2026,12,31))
```
Then relate `Calendar[Date]` 1 → * `TaxFilings[DueDate]`.

## 4. DAX measures (Modeling → New measure)
```DAX
Total Filings   = COUNTROWS(TaxFilings)
Filed           = CALCULATE([Total Filings], TaxFilings[Status] = "Filed")
Filed %         = DIVIDE([Filed], [Total Filings])
Overdue         = CALCULATE([Total Filings], ISBLANK(TaxFilings[FiledDate]), TaxFilings[DueDate] < TODAY())
Total Tax Due   = SUM(TaxFilings[TaxDue])
Days Late       = AVERAGEX(FILTER(TaxFilings, NOT ISBLANK(TaxFilings[FiledDate])),
                           DATEDIFF(TaxFilings[DueDate], TaxFilings[FiledDate], DAY))
On-Time %       = DIVIDE(
                    CALCULATE([Total Filings], TaxFilings[Status] = "Filed",
                              FILTER(TaxFilings, TaxFilings[FiledDate] <= TaxFilings[DueDate])),
                    [Filed])
Tax At Risk     = CALCULATE([Total Tax Due], ISBLANK(TaxFilings[FiledDate]))
```

## 5. Build the page
| Visual | Fields |
|---|---|
| 4 **Cards** (top row) | Total Filings · Filed % · Overdue · Tax At Risk |
| **Clustered bar** | Axis: `Jurisdictions[JurisdictionName]` · Values: Total Filings, Overdue |
| **Donut** | Legend: `Status` · Values: Total Filings |
| **Line chart** | Axis: `TaxYear` · Values: On-Time %, Days Late |
| **Matrix** | Rows: `ClientName` · Columns: `JurisdictionCode` · Values: Total Tax Due. Add conditional formatting (background colour scale). |
| **Slicers** | TaxYear, EntityType, Industry |

**Formatting tips:**
- Keep one colour per meaning (red = overdue).
- Give every visual a clear title.
- Format currency with $ and no decimals.

## 6. Publish and share
- Save the file as `TaxDashboard.pbix`.
- Take a screenshot for your GitHub README.
- Optional: publish to Power BI Service (needs a work or school account; your college ID may work).

---

## What to say in an interview (30 seconds)
"I built a Power BI dashboard on multistate tax filing data. In Power Query I cleaned inconsistent jurisdiction codes and status values and removed duplicate records. I modelled it as a star schema with a date table and wrote DAX measures for on-time %, overdue filings and tax at risk, so a manager can see which states and clients need attention first."
