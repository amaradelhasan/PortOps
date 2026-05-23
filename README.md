# PortOps Data Warehouse Assessment

---

## Setup Instructions

### Prerequisites
- SQL Server 2019+
- Visual Studio 2019+ with SSDT (SQL Server Data Tools)
- Power BI Desktop
- Python 3.x (used for Bronze layer ingestion)

### Steps to Run

1. Run `sql/Bronze_Layer.sql` in SSMS to create the Bronze schema and tables
2. Run `sql/Silver_Layer.sql` to create the Silver schema and apply transformations
3. Run `sql/Gold_Layer.sql` to create the Gold schema (dimensions and facts)
4. Open `ssis/DB.sln` in Visual Studio
5. Update the connection strings in Connection Managers to match your environment
6. Run SSIS packages in this order:
   - `Bronze_Layer.dtsx` — raw data ingestion
   - `Silver_Layer.dtsx` — cleaning and transformation
   - `Gold.DimCustomer.dtsx`
   - `Gold.dimEquipment.dtsx`
   - `Gold.DimShift.dtsx`
   - `Gold.DimTerminal.dtsx`
   - `Gold.FactContainerMovements.dtsx`
   - `Gold.FactGateTransactions.dtsx`
   - `Gold.FactVesselCalls.dtsx`
7. Open `powerbi/dashboard.pbix` in Power BI Desktop
8. Update the SQL Server data source connection if prompted

### Assumptions
- Source file: `PortOps_SourceData.xlsx` — ingested via Python script in Bronze layer due to SSIS file source limitations
- Fiscal year starts April 1 (FY runs April–March)
- dim_date covers the full date range present in the source data
- Rows with dates outside dim_date range are redirected to sentinel row (Out of Range)
- Unknown dimension members use surrogate key = -1
- SCD Type 2 tracks changes to: tier_category, credit_limit
- SCD Type 1 tracks changes to: customer_name, country

---

## Implementation Overview

### 1. Data Ingestion — Bypassing SSIS File Source Limitations

At the beginning of the task, I attempted to use an Excel Source in SSIS, but it did not work due to a known compatibility issue with SSIS. I then tried converting the file to CSV and using a Flat File Source, but that also failed. As a workaround, I used a Python script to reliably ingest the raw data regardless of file format issues.

### 2. Medallion Architecture

I adopted a Medallion Architecture and divided the pipeline into three layers:

- **Bronze** — Raw data ingestion using a Python script to bypass SSIS file source limitations and ensure reliable loading.
- **Silver** — Data cleaning and transformation, including joins and format standardization, to prepare data for analysis.
- **Gold** — Aggregation and structuring of cleaned data into a business-ready Data Warehouse format to support reporting and decision-making.

### 3. Slowly Changing Dimensions (SCD) — Manual Implementation

I implemented SCD without using the SSIS Wizard:

- **SCD Type 1** — Overwrites existing records with the latest values when changes occur (applied to terminal, equipment, shift).
- **SCD Type 2** — Tracks historical changes by inserting new records and maintaining versioning using effective date ranges and an IsCurrent flag (applied to customer).

This manual approach provided more control over the logic and ensured better flexibility in handling data history and integrity across the warehouse layers.

### 4. Error Handling

I implemented an error handling strategy by creating separate error tables to capture any records that failed during processing. Instead of losing problematic data, failed rows are redirected into these tables along with relevant error details such as the stage at which the error occurred. This approach improves data reliability and makes it easier to monitor, debug, and correct issues without interrupting the main ETL workflow.

### 5. Auditing

I implemented auditing at the Gold layer level to improve data tracking and transparency. Key metrics are calculated and recorded for each load process, including the number of source records, destination records, and rejected records. This enables validation of data consistency across the pipeline, monitoring of data quality, and quick identification of any discrepancies between input and output.

---

## Project Structure

```
submission/
├── README.md
├── sql/
│   ├── Bronze_Layer.sql
│   ├── Silver_Layer.sql
│   └── Gold_Layer.sql
├── ssis/
│   ├── DB.sln
│   ├── Project.params
│   ├── Connection Managers/
│   └── SSIS Packages/
│       ├── Bronze_Layer.dtsx
│       ├── Silver_Layer.dtsx
│       ├── Gold.DimCustomer.dtsx
│       ├── Gold.dimEquipment.dtsx
│       ├── Gold.DimShift.dtsx
│       ├── Gold.DimTerminal.dtsx
│       ├── Gold.FactContainerMovements.dtsx
│       ├── Gold.FactGateTransactions.dtsx
│       └── Gold.FactVesselCalls.dtsx
├── powerbi/
│   └── dashboard.pbix
└── screenshots/
```

---

## DAX Measures

```dax
Avg Crane Cycle Seconds = 
VAR TotalCycleSeconds =
    SUM('FactContainerMovements'[crane_cycle_time_seconds])
VAR TotalMoves =
    [Total Container Moves]
RETURN
DIVIDE(
    TotalCycleSeconds,
    TotalMoves,
    0
)
```

```dax
Avg Delay Minutes = 
DIVIDE (
    AVERAGE ( FactVesselCalls[Delay  in scend] ),
    60
)
```

```dax
Avg Turnaround Minutes = 
AVERAGEX (
    FactGateTransactions,
    DIVIDE ( FactGateTransactions[stay_duration_seconds], 60 )
)
```

```dax
Avg Weight (Tons) = 
AVERAGEX (
    FactContainerMovements,
    COALESCE ( FactContainerMovements[weight_tons], 0 )
)
```

```dax
Berth Occupancy % = 
DIVIDE(
    'Mesaurs'[Total Turnaround Hours],
    'Mesaurs'[Total Available Hours],
    0
)
```

```dax
CY Vessel Calls = 
CALCULATE (
    [Total Vessel Calls],
    YEAR ( Dim_Date[full_date] ) = YEAR ( TODAY() )
)
```

```dax
Gate In Count = 
COUNTROWS (
    FILTER (
        FactGateTransactions,
        FactGateTransactions[direction] = "IN"
    )
)
```

```dax
Gate Out Count (By Out Date) = 
CALCULATE (
    COUNTROWS ( FactGateTransactions ),
    USERELATIONSHIP (
        dim_date[date_key],
        FactGateTransactions[gate_out_DateKey]
    ),
    FactGateTransactions[direction] = "OUT"
)
```

```dax
Gate Transaction Volume = 
COUNTROWS('FactGateTransactions')
```

```dax
Moves Variance = 
SUM ( FactVesselCalls[total_moves_actual] ) -
SUM ( FactVesselCalls[total_moves_planned] )
```

```dax
PY Vessel Calls = 
CALCULATE (
    [Total Vessel Calls],
    SAMEPERIODLASTYEAR ( Dim_Date[full_date] )
)
```

```dax
Rolling Avg 7 Days = 
VAR DatesTable =
    DATESINPERIOD (
        dim_date[full_date],
        MAX ( dim_date[full_date] ),
        -7,
        DAY
    )
RETURN
AVERAGEX (
    DatesTable,
    COALESCE ( [Total Container Moves], 0 )
)
```

```dax
Rolling Total 7 Days = 
CALCULATE (
    [Total Container Moves],
    DATESINPERIOD (
        dim_date[full_date],
        MAX ( dim_date[full_date] ),
        -7,
        DAY
    )
)
```

```dax
Total Actual Moves = 
SUM ( FactVesselCalls[total_moves_actual] )
```

```dax
Total Available Hours = 
DISTINCTCOUNT('FactVesselCalls'[atadate_key]) * 24 * [Total Terminals]
```

```dax
Total Container Moves = DISTINCTCOUNT ( FactContainerMovements[movement_id] )
```

```dax
Total Planned Moves = 
SUM ( FactVesselCalls[total_moves_planned] )
```

```dax
Total Terminals = 
DISTINCTCOUNT('gold dimterminal'[terminalSk])
```

```dax
Total Turnaround Hours = 
DIVIDE(
    [Total Turnaround Seconds],
    3600,
    0
)
```

```dax
Total Turnaround Seconds = 
SUM('FactVesselCalls'[Turnaround in scend])
```

```dax
Total Vessel Calls = 
COUNTROWS ( FactVesselCalls )
```

```dax
Vessel YoY % = 
DIVIDE (
    [CY Vessel Calls] - [PY Vessel Calls],
    [PY Vessel Calls]
)
```

---

## Written Questions

**Q1: SCD Type 1 vs Type 2 — practical difference and justification**

SCD Type 1 overwrites data without keeping history, while Type 2 preserves history by adding new rows with versioning (EndDate / IsCurrent). I used Type 1 for reference-like dimensions (terminal, equipment, shift) where changes are just corrections. I used Type 2 for customer data because changes like tier and credit limit are important for historical KPI analysis and must be tracked over time.

**Q2: Surrogate key vs natural key in fact tables — two reasons from dim_customer**

Surrogate keys are used instead of natural keys in fact tables because they uniquely identify each version of a record in SCD Type 2, ensuring correct historical analysis. Also, they improve performance since integer-based joins are much faster than using natural keys like customer_id, especially in large fact tables.

**Q3: Handling dates outside dim_date range without pipeline failure**

If a fact arrives with a date outside the bounded range, the Lookup fails. I handled this by adding sentinel values in dim_date (Unknown and Out of Range) so records are redirected instead of causing a pipeline failure.

**Q4: Why the SSIS SCD Wizard is unsuitable for production Type 2 loads**

The SSIS SCD Wizard is not suitable for production because it generates hard-to-maintain logic and processes rows one by one. I used a manual approach with Lookup + Conditional Split and set-based inserts/updates, which is faster, more scalable, and production-ready.

**Q6: Role of the staging layer and risks of loading directly from Excel**

In a Medallion Architecture, the Bronze layer loads raw data once before transformation, making the pipeline faster, more reliable, and easier to audit and recover. Loading directly from Excel into facts is risky because errors require re-reading the source file, which may be unavailable or inconsistent. Staging also enables early row-count reconciliation before moving to Silver and Gold layers.

**Q7: Why only one active relationship is allowed between two tables**

DAX allows only one active relationship to avoid ambiguity in filter context. If both gate_in_date_key and gate_out_date_key were active, it could lead to incorrect or duplicated results. In my model, gate_in_date_key is active, while gate_out_date_key is kept inactive and only used in measures via USERELATIONSHIP when needed.

**Q8: USERELATIONSHIP vs CROSSFILTER — when to use each**

USERELATIONSHIP activates an inactive relationship within a specific measure's context. CROSSFILTER changes the filter direction between tables (single or both directions). USERELATIONSHIP is used when two relationships exist between the same tables and the measure needs to use the inactive one, as in Gate Out Count. CROSSFILTER is used when bidirectional filtering is needed between a dimension and a fact.

**Q9: KPI not responding to date slicer — causes in investigation order**

A KPI not responding to a date slicer is usually caused by: the measure using a fact table date column instead of the dim_date relationship; a hidden visual-level filter overriding the slicer; the slicer being on another page without Sync Slicers enabled; using an inactive relationship such as gate_out_date_key without USERELATIONSHIP; or CALCULATE with ALL or REMOVEFILTERS removing the date filter context.