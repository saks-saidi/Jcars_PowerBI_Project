# JCars Logistics: From Raw Data to Business Decisions (Power BI)

> A Power BI business intelligence solution for JCars Logistics, a Kenyan vehicle importer, seller and delivery company. It takes a deliberately messy 276-row flat file through a data-quality audit, currency standardisation, Power Query cleaning, star-schema modelling, DAX, an Executive Dashboard and detailed report pages, and ends in evidence-based recommendations.

**Author:** [Saidi] · **Cohort/Track:** [cohort9 / Data Science ]
**Dev.to article:** [link] · **Tools:** Power BI Desktop, Power Query (M), DAX

---

## Table of contents

1. [Business objective](#1-business-objective)
2. [Dataset and grain](#2-dataset-and-grain)
3. [Data quality audit](#3-data-quality-audit)
4. [Currency standardisation](#4-currency-standardisation)
5. [Cleaning and transformation (Power Query)](#5-cleaning-and-transformation-power-query)
6. [Validation](#6-validation)
7. [Assumptions and business rules](#7-assumptions-and-business-rules)
8. [Data model](#8-data-model)
9. [Key DAX measures](#9-key-dax-measures)
10. [Executive Dashboard and report structure](#10-executive-dashboard-and-report-structure)
11. [Interactivity](#11-interactivity)
12. [Analyst-defined questions](#12-analyst-definedquestions)
13. [Key insights](#14-key-insights)
14. [Challenges and lessons learned](#16-challenges-and-lessons-learned)
15. [Repository structure](#17-repository-structure)

---

## 1. Business objective

JCars Logistics imports, sells and delivers vehicles across Kenya. Management wants to know **what happened, where, how significant it was, and what deserves attention** across revenue, costs, profitability, vehicles, customers, branches and regions, sales reps and lead sources, payments, deliveries, returns and cancellations, customer ratings, and trends over time.

The deliverable is a decision-support tool, not a set of charts: page 1 answers *"How is JCars performing?"*, and later pages let a manager investigate *where* and *why*.

## 2. Dataset and grain

| Item      | Detail                                                                                                           |
| --------- | ---------------------------------------------------------------------------------------------------------------- |
| File      | `Jcars_data.csv` (raw, untouched copy kept in `/data/raw`)                                                       |
| Size      | 276 rows × 32 columns, every column ingested as text                                                             |
| Period    | Order dates between Jan 2025 and Jun 2026 (after cleaning)                                                       |
| **Grain** | **One row = one sales order for one vehicle configuration (make/model/year) with a quantity of 1 or more units** |

**Column groups**

| Group            | Columns                                                                                             |
| ---------------- | --------------------------------------------------------------------------------------------------- |
| Transaction      | Order ID, Order Date, Delivery Date                                                                 |
| Customer         | Customer Name, Customer Type, Customer Age                                                          |
| Geography        | Region, County, City, Branch                                                                        |
| People / channel | Sales Rep, Lead Source                                                                              |
| Vehicle          | Car Make, Car Model, Vehicle Type, Vehicle Year, Fuel Type, Transmission, Color                     |
| Measures         | Units Sold, Unit Selling Price, Unit Cost, Discount, Delivery Fee, Logistics Cost, Revenue Recorded |
| Status           | Payment Method, Payment Status, Delivery Status, Returned                                           |
| Experience       | Customer Rating, Review Count                                                                       |

**Keys:** They were no Foreign keys or Primary keys yet.

## 3. Data quality audit

I found **17 significant issues**. The table shows what I found, the risk, and the decision. Nothing was deleted merely for looking unusual; records were corrected, standardised, flagged or excluded *from specific calculations*.

| #   | Issue                              | Evidence                                                                                                                                | Risk if ignored                                  | Decision                                                                                                                                 |
| --- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Order ID unusable as a key**     | 5 prefix styles (`LC`, `LCL-`, `ORD`, `ord`, `CAR`); 19 blanks/`N/A`/`-`; numbers 1020, 1086, 1174 each appear twice                    | Broken joins, false duplicates                   | The numeric part equals `1000 + row position` for 254 of 257 rows, so I rebuilt `OrderKey`. Original kept as `OrderID_Raw`               |
| 2   | **Mixed date formats**             | Excel serials (81 order dates), `dd/mm/yyyy`, `mm/dd/yyyy`, `26-Mar-25`, `Aug 29, 2025`, `2026-13-04` (year-day-month)                  | Wrong or missing dates, broken time intelligence | Changed Format to Date and Removed errors                                                                                                |
|     |                                    |                                                                                                                                         |                                                  |                                                                                                                                          |
|     |                                    |                                                                                                                                         |                                                  |                                                                                                                                          |
| 3   | **Mixed currencies**               | `$`, `USD`, `EUR`, `ZAR`, `R`, `?`, plain numbers and `KSh`/`KES` in all 5 money columns                                                | Adding dollars to shillings                      | Converted to the respective rates                                                                                                        |
| 4   | **Money stored as text**           | `KSh 8,136,000`, `KSh6,708,000`, `1.95M`, `9.14M`                                                                                       | Text cannot be summed; `M` suffix misread        | Calculated to KSh and changed type to Fixed decimal number                                                                               |
| 5   | **Text pretending to be null**     | `missing`, `error`, `#VALUE!`, `not available`, `TBD`, `-`, `N/A`, `NULL`                                                               | Calculation errors or fake zeros                 | Converted to  `null`                                                                                                                     |
| 6   | **Negative and zero money values** | 1 negative price, 2 negative costs, 2 negative delivery fees, 1 negative logistics cost; 8 negative and 18 zero `Revenue Recorded`      | Corrupted profit                                 | Turned the negative to positives and the zero to 0                                                                                       |
|     |                                    |                                                                                                                                         |                                                  |                                                                                                                                          |
| 7   | **Inconsistent discount encoding** | `7%`, `0.07`, `7`, `7 percent`, `ten percent`, `No Discount`, `#ERROR`, `Disc`; also `120%`, `1.2`, `-10%`                              | Discounts of 7 vs 700%                           | Normalised to a fraction. Valid range 0–25%. Values outside it (10 above 25%, 5 negative) treated as invalid, not guessed                |
| 11  | **Inconsistent categories**        | Make 62 spellings (`Toyta`, `Totoya`, `Toyota Kenya`, `V.W`); Customer Type 33; Branch 64; Lead Source 57; Region 53; Payment Status 32 | Split totals (Toyota shown as 6 makes)           | Rule-based mapping to a canonical list (10 makes, 5 customer types, 8 branches, 8 lead sources, 5 payment statuses, 5 delivery statuses) |
| 12  | **Invalid vehicle years**          | `Twenty Twenty`, `202A`, `1899`, `2032`, `null`, blanks                                                                                 | Wrong vehicle-age analysis                       | `Twenty Twenty` → 2020.                                                                                                                  |
| 13  | **Units as text / invalid**        | `two`, `one`, `3 cars`, `-1` (×3), `0`, blanks                                                                                          | Units and revenue wrong                          | Words and `3 cars` converted. Zero, negative and missing units excluded from calculations (11 rows)                                      |
| 14  | **Ratings and ages out of range**  | Ratings as `4.5 out of 5`, `4.7/5`, `Excellent`, `6`, `-1`; ages `121`, `-5`, `0`, `thirty`                                             | Skewed averages                                  | Parsed to 0–5. Out-of-range → null. `Excellent` → 5. Ages outside 18–100 → null                                                          |
| 15  | **Sales-rep name corruption**      | `Dan1el Kimani`, `Grace Njer1`, `Mercy` (first name only), double spaces                                                                | 80 "reps" instead of 10                          | Digit `1` → `i`, whitespace trimmed, proper case, first-name-only mapped to the one matching rep                                         |

## 4. Currency standardisation

**Affected columns:** Unit Selling Price, Unit Cost, Delivery Fee, Logistics Cost, Revenue Recorded.

**Rule:** Changed values in  KSh . Explicit currency markers are detected and converted.

| Marker in data       | Currency            | Rate to KES |
| -------------------- | ------------------- | ----------- |
| none, `KSh`, `KES`   | KSh                 | 1           |
| `$`, `USD`           | US Dollar           | 129         |
| `EUR`                | Euro                | 147         |
| `?` (mangled symbol) | Euro (inferred)     | 147         |
| `ZAR`, `R`           | South African Rand  | 7.8         |
| `1.9M` suffix        | ×1,000,000 (amount) | n/a         |

**Rates and source.** One fixed set of rates is used for the whole project, so every conversion is comparable.

## 5. Cleaning and transformation (Power Query)

Every column was loaded as **text** first so nothing was changed. 

**Query flow:** `Raw` (all text) → `Staging` (clean money, dates, units, discount; standardise text) → `FactSales` + dimension tables. Only `FactSales` and dimensions load into the model.

**Currency conversion:** merge the currency code against an `ExchangeRates` table (`Currency`, `RateToKES`), then `AmountKES = Amount * RateToKES`. Keeping rates in a table means they are documented, visible and changeable in one place.

**Text standardisation:** trim, collapse repeated spaces, then map to canonical values via `Text.Contains` on a lower-cased, punctuation-stripped key. Example: any make containing `toyt`/`totoya` becomes `Toyota`.

**Geography:** `County` has `Region` (Kiambu → Central, Machakos → Eastern, Mombasa → Coast, Kisumu → Nyanza, Kakamega → Western, Nakuru and Uasin Gishu → Rift Valley, Nairobi → Nairobi). This repairs 53 messy region spellings in one step. County typos (`Kambu`, `Mchakos`, `NKR`, `Msa`, `Eldoret`) were mapped to the correct county.

## 6. Validation

| Check                                 | Result                                                                                                                           |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Row count preserved                   | 276 in → 74 in `FactSales` (Duplicates were dropped)                                                                             |
| Data types                            | Dates `date`, money `decimal`, units `whole number`, flags `true/false`                                                          |
| Category counts after standardisation | 10 makes, 8 branches, 5 customer types, 8 lead sources, 5 payment methods, 5 payment statuses, 5 delivery statuses (+ `Unknown`) |
| No duplication from joins             | Sum of `RevenueKES` in `FactSales` equals sum in the model after relationships                                                   |
| Currency                              | No un-converted `USD`/`EUR`/`ZAR` amounts remain; all money columns end in `KSh`                                                 |

## 7. Assumptions and business rules

| Area             | Rule                                                                                                                         |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Currency         | Unmarked values are KES. USD 130, EUR 150, ZAR 7.5. `?` is treated as EUR. One rate set for the whole project                |
| Revenue          | `RevenueKES` = Units × Price × (1 − Discount) + Delivery Fee. Delivery fee is **income billed to the customer**, not a cost  |
| Cost of goods    | Units × Unit Cost                                                                                                            |
| **Gross Profit** | Revenue − COGS                                                                                                               |
| Gross Margin %   | Gross Profit ÷ Revenue                                                                                                       |
| Customer value   | Sum of `RevenueKES` per cleaned customer name. No customer ID exists, so identical names are assumed to be the same customer |
| Geography        | Branch = selling yard. Region/County/City = customer/delivery location. Whether these match the branch is not verified       |
| Ratings          | 0–5 scale; `Excellent` = 5                                                                                                   |

## 8. Data model

The flat table is reshaped into a **star schema**. The business process is **vehicle sales orders**, so `FactSales` (one row per order) sits in the centre and descriptive attributes move to dimensions.

```
                     DimDate
                        │ 1
                        │ (active: Order Date; inactive: Delivery Date)
DimCar ─1───*── FactSales 
DimCustomer ─1──*──    │    ──*───1─ DimSalesRep
DimLocation ─1─*──    │   
DimPaymentMethod ─1─*──┘    ──*───1─ DimOrderStatus

```

| Table            | Type           | Key        | Contents                                                         |
| ---------------- | -------------- | ---------- | ---------------------------------------------------------------- |
| FactSales        | Fact           | OrderID    | Units, prices, costs, revenue, flags, rating, keys               |
| DimDate          | Dimension      | Date       | Year, Quarter, Month, MonthNo, Year-Month (marked as date table) |
| DimCar           | Dimension      | CarID      | Make, Model, Type, Year, Fuel, Transmission, Colour              |
| DimCustomer      | Dimension      | CustomerID | Cleaned name, Customer Type, Age band                            |
| DimLocation      | Dimension      | LocationID | City, County, Region, Branch                                     |
|                  |                |            |                                                                  |
| DimSalesRep      | Dimension      | SalesRepID | Rep, Lead source                                                 |
|                  |                |            |                                                                  |
| DimPaymentMethod | Dimension      | PaymentID  | Payment method                                                   |
| DimOrderStatus   | Junk dimension | StatusID   | Payment status, Delivery status, Returned                        |

**Relationships:** all **one-to-many**, dimension → fact, **single direction** (dimension filters fact). This avoids ambiguity and unintended filter loops.

**Dates.** One `Date_table` table, marked as the date table. `Order Date` is the *active* relationship. `Delivery Date` is an *inactive* relationship activated with `USERELATIONSHIP` only in delivery measures, so there is never ambiguity about which date drives a visual.

📷 `screenshots/model_view.png`

## 9. Key DAX measures

Every measure has a purpose. Base measures use the `IsAnalysable` flag so all KPIs share one consistent population.

```dax
-- Main
Units Sold      = SUM ( jcars_fact_table[Units sold] )
Revenue         = SUM ( jcars_fact_table[Revenue Recorded] )
COGS         = CALCULATE ( SUM (jcars_fact_table[Units sold] * jcars_fact_table[Unit cost])
Gross Profit    = [Revenue] - [COGS]
Gross Margin %  = DIVIDE ( [Gross Profit], [Revenue] )
Orders          = CALCULATE ( DISTINCTCOUNT ( jcars_fact_table[OrderID] ), FactSales[IsAnalysable] = TRUE () )
Avg Order Value = DIVIDE ( [Revenue], [Orders] )

-- Logistics (only where logistics cost is known)
Logistics Cost  = SUM(jcars_fact_table[Logistic cost])

-- Discount
Avg Discount % =  AVERAGE ( jcars_fact_table[Discount] )

-- All orders and unpaid revenue
All Orders       = COUNTROWS ( jcars_fact_table )
Unpaid Revenue =
    CALCULATE ( [Revenue], DimOrderStatus[Payment Status] IN { "Pending", "Partially Paid", "Unpaid" } )
Unpaid % = DIVIDE ( [Unpaid Revenue], [Revenue] )

-- Ratings
Avg Rating = AVERAGE ( jcars_fact_table[Rating] )


-- Time series analysis
Revenue PY   = CALCULATE ( [Revenue], SAMEPERIODLASTYEAR ( Date_table[Date] ) )
Revenue YTD  = TOTALYTD ( [Revenue], Date_table[Date] )
```

**Time-intelligence caveat:** data covers Jan 2025 to Jun 2026 only, so year-on-year comparison is valid for **Jan–Jun** only. The Executive Dashboard therefore leads with quarterly and monthly trend rather than a full-year YoY figure.

## 10. Executive Dashboard and report structure

**Page 1: Executive Dashboard ("How is JCars performing?")**

| Zone            | Content                                                                               |
| --------------- | ------------------------------------------------------------------------------------- |
| KPI strip       | Revenue · Units Sold · Gross Profit · Gross Margin % · Return Rate % · Unpaid %       |
| Trend           | Monthly revenue (columns) with Gross  (bars). Shows revenue rising while margin slips |
| Comparisons     | Revenue and margin by Branch; Revenue by Vehicle Type; Revenue by Make                |
| Attention panel | Card counts for exception orders, high-discount orders, deliveries delayed            |
| Slicers         | Region, Branch, Customer Type                                                         |

**Detailed report**

| Page                                  | Purpose                                                                              | Main questions answered                                              |
| ------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| 2. Sales and Profitability            | Revenue, cost, GP, margin, discount vs margin                                        | Total revenue, GP, margin, discount relationship                     |
| 3. Vehicles                           | Make, model, type, fuel, year of manufacture                                         | Strongest makes/models, margin by vehicle, vehicle-year patterns     |
| 4. Customers                          | Customer type, top customers, contribution                                           | Highest-value customers, customer-type contribution                  |
| 5. Branches, Regions and Reps         | Geographic and rep performance, lead sources                                         | Branch/county/region performance, rep ranking, valuable lead sources |
| 6. Payments and Delivery              | Payment method/status by value and count; delivery status, lead time, logistics cost | Payment profile, delivery by branch/vehicle, logistics vs value      |
| 7. Returns, Cancellations and Ratings | Return and cancellation patterns, rating relationships                               | Vehicles frequently returned/cancelled, rating vs revenue            |



## 11. Interactivity

| Slicers (Year, Region, Branch, Customer Type) | Every page, synced | Let a manager narrow to their area |
| --------------------------------------------- | ------------------ | ---------------------------------- |

## 12. Analyst-defined questions

| #   | Question                                                   | Why it matters                                                               | Answered on     |
| --- | ---------------------------------------------------------- | ---------------------------------------------------------------------------- | --------------- |
| 1   | Do deeper discounts actually sell more vehicles?           | Discounts cost margin. If they do not lift volume, they are money given away | No              |
| 2   | How much revenue is still uncollected, and from whom?      | Cash flow risk                                                               | 54.98M from NGO |
|     |                                                            |                                                                              |                 |
| 3   | Do newer vehicles get returned more often than older ones? | Points to sourcing or quality problems                                       | Yes             |
| 4   | Can `Revenue Recorded` be trusted?                         | Determines whether existing reports are reliable                             | Yes             |

Questions 1, 2, 3 and 4 are answered inside the Power BI report.

## 13. Key insights

**Headline:** KSh **408.6M** revenue · **128** vehicles · **KSh 7.8** gross profit · **2%** gross margin.

1. **Deep discounts erase profit without buying volume.** Quarterly gross margin slid from 15.0% (Q1 2025) to -3.0% (Q2 2026) while Q4 2026 revenue hit a period high of KSh 49.2M, with a gross margin of 53.0%.
2. **Athi River Branch** had the highest Revenue of 53.9M and a gross margin of 49.0%, with least is Kakamega Branch with a Revenue of 1.9M and a gross margin of -1.0%.
3. **Volume is not value in lead sources.** Whatsapp is the biggest revenue source (KSh 65.8M) but earns 36.0% margin, with 9 units sold and Walk-in had 16.2M revenue with a negative margin of -17%, selling 11 units.
4. **24% of active revenue (≈ KSh 96.33M) is Pending ,  Partially Paid or Unpaid.** The exposure is spread across every customer type (NGO and Corporate each ≈ KSh 64.81M), so it is a process issue, not one bad customer.
5. **Toyota and SUVs dominate, and customer types differ sharply in margin.** Toyota is 8% of revenue and SUVs 15%%, which is strong but a concentration risk. Dealers earn the best margin (14.0%) while NGOs, at 16% of revenue. Cash and Wire deals earn 30.0% against  24% for RTGS and  Cash.
6. **Customer ratings are not linked to revenue.** Correlation of rating with order revenue is −0.08, and the average rating is 3.6 out of 5 for every customer type (3.56–3.62). customer satisfaction is not a function of deal size or customer type.

## 14. Challenges and lessons learned

- **PowerQuery  issues** with performance and ran slower.
- **Discovering that the delivery fee sits inside recorded revenue** changed the profit definition. Testing a formula against a recorded column paid off more than any single cleaning step.
- **Recorded values are not automatically right.** The recorded revenue column was less reliable than a recalculation from its own components.
- **Lesson:** understand the business meaning of each field before modelling. Most of the value came from questions asked *before* touching a visual.

## 15. Repository structure

```
jcars-powerbi/
├── README.md
├── JCars_Logistics.pbix
├── data/
│   ├── raw/Jcars_data.csv
│   └── cleaned/FactSales_clean.csv
├── docs/
│   ├── data_quality_log.md
│   
└── screenshots/
    ├── model_view.png
    ├── 01_executive_dashboard.png
    └── ...
```
