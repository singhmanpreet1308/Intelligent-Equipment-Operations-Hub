# Phase 4 — Power BI Analytics & Reporting

**Project:** Intelligent Equipment Operations Hub  
**Status:** Completed, as confirmed by the user in the Phase 4 conversation.  
**Purpose:** Compact implementation record, measure reference, and validation runbook.

This document records the completed conversation; it is not a fresh audit of the live Power BI or Fabric environment. SQL below is provided for repeatable validation and was not executed while preparing this document.

## 1. Architecture and scope

```text
Phase 3 Gold data → EquipmentOperations_Warehouse (dbo)
    → Direct Lake semantic model + dim_date
    → DAX measures → three analytical pages + Home/navigation
```

| Table | Grain / key | Purpose |
|---|---|---|
| dim_site | One site / Site_ID | Site filtering |
| dim_asset | One asset / Asset_ID | Asset, type, criticality; Site_ID links to site |
| fact_failure | One failure / Failure_ID | Failures, downtime, type, root cause |
| fact_maintenance | One event / Maintenance_ID | Maintenance activity, components, planned status |
| fact_work_order | One order / WorkOrder_ID | Priority, status, team, SLA, estimated/actual hours |
| fact_cost | One cost record / Cost_ID | Operational cost |
| dim_date | One calendar day / Date | Shared date filtering and operating-time denominator |

The six business tables use Direct Lake on the Warehouse. `dim_date` was created as a semantic-model calculated table. Sensor telemetry remains outside this report's scope.

## 2. Relationships and date window

All recorded relationships are **active, one-to-many, single-direction**, filtering from the dimension to the many side. No fact-to-fact relationships are required.

| One side | Many side |
|---|---|
| dim_site[Site_ID] | dim_asset[Site_ID] |
| dim_asset[Asset_ID] | fact_failure[Asset_ID] |
| dim_asset[Asset_ID] | fact_maintenance[Asset_ID] |
| dim_asset[Asset_ID] | fact_work_order[Asset_ID] |
| dim_asset[Asset_ID] | fact_cost[Asset_ID] |
| dim_date[Date] | fact_failure[Failure_Date] |
| dim_date[Date] | fact_maintenance[Maintenance_Date] |
| dim_date[Date] | fact_cost[Cost_Date] |

**Operational window:** 2026-05-30 through 2026-08-29, inclusive (**92 days**). This replaced the initial 2025–2027 calendar, which overstated potential operating hours. The observed failure-only window was narrower: 2026-06-05 through 2026-08-29.

```dax
dim_date =
ADDCOLUMNS(
    CALENDAR(DATE(2026, 5, 30), DATE(2026, 8, 29)),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month", FORMAT([Date], "MMM"),
    "Year Month", FORMAT([Date], "YYYY-MM"),
    "Quarter", "Q" & FORMAT([Date], "Q"),
    "Day", DAY([Date]),
    "Day Name", FORMAT([Date], "DDD")
)
```

**Filter boundary:** No `dim_date → fact_work_order` relationship was documented. A Date slicer on the work-order page therefore does not, by itself, filter work-order measures. Site and asset filters do. A future work-order date relationship needs an explicit business choice (created, scheduled, or completed date).

## 3. Direct Lake implementation fixes

- **Calculated columns:** New column was unavailable in the implemented Direct Lake on SQL setup. Required fact-table columns were added in the Warehouse instead of DAX.
- **Failure_Date:** Added as `DATE`, populated with `CAST(Failure_DateTime AS DATE)`. The date relationship uses this column, replacing the timestamp relationship.
- **Planned_Status:** Added as `VARCHAR(20)`, populated with `CASE WHEN Planned_Flag = 1 THEN 'Planned' ELSE 'Unplanned' END`. Used as the donut legend; a text measure cannot substitute for a categorical legend column. This mapping also labels null flags as Unplanned, so check null flags separately.
- **Schema sync:** Editing mode → **Edit tables** → reselect/apply the six Warehouse tables exposed added columns. Recheck all relationships afterward: relationship loss during this workflow was reported in the conversation.
- **Refresh dependency:** Phase 3 uses truncate-and-reload. Ensure subsequent loads populate both derived columns, either upstream or in a post-load step, before refreshing reporting. Initial manual population alone does not survive new data loads.

## 4. DAX measure reference

Each row below is a separate measure definition. Counts use distinct business IDs; duplicate rows can still inflate downtime and cost sums and must be checked separately.

### Core measures

| Measure | DAX expression |
|---|---|
| Total Failures | `DISTINCTCOUNT(fact_failure[Failure_ID])` |
| Total Downtime Hours | `SUM(fact_failure[Downtime_Hours])` |
| Total Maintenance Events | `DISTINCTCOUNT(fact_maintenance[Maintenance_ID])` |
| Total Work Orders | `DISTINCTCOUNT(fact_work_order[WorkOrder_ID])` |
| Total Cost | `SUM(fact_cost[Total_Cost])` |

### Operational measures

| Measure | DAX expression |
|---|---|
| MTTR Hours | `COALESCE(DIVIDE([Total Downtime Hours], [Total Failures]), 0)` |
| Planned Maintenance % | `DIVIDE(CALCULATE([Total Maintenance Events], fact_maintenance[Planned_Flag] = TRUE()), [Total Maintenance Events], 0)` |
| Open Work Orders | `COALESCE(CALCULATE([Total Work Orders], fact_work_order[WorkOrder_Status] = "Open"), 0)` |
| Asset Count | `DISTINCTCOUNT(dim_asset[Asset_ID])` |
| Selected Days | `COUNTROWS(VALUES(dim_date[Date]))` |
| Potential Operating Hours | `[Asset Count] * [Selected Days] * 24` |
| Availability % | `DIVIDE([Potential Operating Hours] - [Total Downtime Hours], [Potential Operating Hours], 0)` |
| MTBF Hours | `DIVIDE([Potential Operating Hours] - [Total Downtime Hours], [Total Failures], 0)` |
| Completed Work Orders | `CALCULATE([Total Work Orders], fact_work_order[WorkOrder_Status] = "Completed")` |
| SLA Breached Work Orders | `CALCULATE([Total Work Orders], fact_work_order[SLA_Breached_Flag] = TRUE())` |
| Average Actual Hours | `AVERAGE(fact_work_order[Actual_Hours])` |
| Average Estimated Hours | `AVERAGE(fact_work_order[Estimated_Hours])` |
| Average Hours Variance | `[Average Actual Hours] - [Average Estimated Hours]` |

**Interpretation and formatting:**

- Availability and MTBF use a **24×7 assumption for every selected asset on every selected calendar day**. They are fleet-level approximations, not measured runtime; shifts, commissioning dates, asset active periods, and overlapping downtime are not explicitly modeled.
- MTTR is average recorded downtime per failure. Zero returned for a no-failure period is a display convention, not evidence of instantaneous repair; MTBF also returns zero for a zero denominator.
- Open means the exact status `Open`, not every non-completed status. The current snapshot contains only `Completed`, so Open Work Orders is correctly zero.
- Average Hours Variance is the difference of two averages; positive means actual hours exceed estimates. With differing missing values, this differs from averaging paired row variances.
- Format percentages as percentages, hours to two decimals, counts as integers, and costs as GBP (£). Use dimension labels and explicit measures in summary tables, especially `[Total Cost]` rather than the raw cost column.

### Asset insights and rankings

The three insight measures use the same recorded pattern:

```dax
Highest Failure Asset =
VAR AssetTable =
    TOPN(1,
        SUMMARIZE(dim_asset, dim_asset[Asset_Name], "Failures", [Total Failures]),
        [Failures], DESC)
RETURN MAXX(AssetTable, dim_asset[Asset_Name])
```

| Measure | Substitute in the pattern |
|---|---|
| Highest Downtime Asset | `"Downtime", [Total Downtime Hours]` and sort by `[Downtime]` |
| Highest Cost Asset | `"Cost", [Total Cost]` and sort by `[Cost]` |

These measures respect current filters. `TOPN` can return ties; `MAXX` selects one asset name from them. They do not list every tied asset or explicitly suppress no-activity results.

```dax
Asset Failure Rank =
VAR CurrentFailures = [Total Failures]
VAR CurrentDowntime = [Total Downtime Hours]
VAR CurrentAsset = SELECTEDVALUE(dim_asset[Asset_Name])
RETURN
    1 + COUNTROWS(
        FILTER(ALLSELECTED(dim_asset[Asset_Name]),
            VAR CompareAsset = dim_asset[Asset_Name]
            VAR CompareFailures = CALCULATE([Total Failures])
            VAR CompareDowntime = CALCULATE([Total Downtime Hours])
            RETURN
                CompareFailures > CurrentFailures
                || (CompareFailures = CurrentFailures && CompareDowntime > CurrentDowntime)
                || (CompareFailures = CurrentFailures && CompareDowntime = CurrentDowntime
                    && CompareAsset < CurrentAsset)
        )
    )

Asset Downtime Rank =
RANKX(ALLSELECTED(dim_asset[Asset_Name]), [Total Downtime Hours], , DESC, DENSE)
```

Use rank `<= 10` as the visual filter. Failure rank breaks ties by downtime, then asset name; this assumes unique asset names. Downtime rank is dense and may display more than ten assets because of ties. Use Asset_ID in a future revision if names are not unique.

## 5. Completed report pages

| Page | Content and interactions |
|---|---|
| **Executive Operations Overview** | Failure, downtime, availability, MTTR, MTBF and cost KPIs; failures by asset type, downtime trend, maintenance by component, cost by site, asset summary and highest-impact asset insights. Site and Date slicers. |
| **Asset Reliability & Maintenance Analysis** | Reliability KPI row; ranked failures/downtime by asset; failure type and root-cause analysis; maintenance by component; Planned vs Unplanned donut using Planned_Status. Site, Asset Type, Asset Name and Date slicers. |
| **Work Orders & Maintenance Operations** | Total, completed, open and SLA-breached orders; average actual hours and hours variance; priority, SLA breaches by team, estimated vs actual hours (clustered columns), and workload by team. Site, Asset Type, Priority and Date slicers; date limitation noted above. |
| **Home / navigation** | Project landing page; Overview → Executive Operations Overview; Reliability & Maintenance → Asset Reliability & Maintenance Analysis; Work Orders → Work Orders & Maintenance Operations. Buttons use Action → Page navigation. |

Navigation, KPI responses, and chart cross-filtering were included in final testing. The optional Cost & Operational Impact page was not required. AI Support is a later-phase placeholder, not a completed AI integration.

## 6. Validation and reconciliation

### Recorded snapshot

Reference values from the conversation, not live query results: **150 assets; 40 failures; 306.86 downtime hours; 310 maintenance events; 310 work orders; approximately £404,618.18 cost; 87.10% planned maintenance; 310 completed / 0 open orders; 82 SLA breaches**. Average actual hours was approximately **2.93**, and average hours variance **−0.01**.

At full window and all assets: `150 × 92 × 24 = 331,200` potential hours; MTTR ≈ **7.67 hours**, availability ≈ **99.91%**, and MTBF ≈ **8,272.33 hours**. Reconcile unrounded values with identical filter context.

### Warehouse SQL examples

Run against `EquipmentOperations_Warehouse`. For the first query, clear report slicers and compare the same source snapshot.

```sql
SELECT
    (SELECT COUNT(DISTINCT Failure_ID) FROM dbo.fact_failure) AS Total_Failures,
    (SELECT SUM(Downtime_Hours) FROM dbo.fact_failure) AS Total_Downtime_Hours,
    (SELECT COUNT(DISTINCT Maintenance_ID) FROM dbo.fact_maintenance) AS Total_Maintenance_Events,
    (SELECT COUNT(DISTINCT WorkOrder_ID) FROM dbo.fact_work_order) AS Total_Work_Orders,
    (SELECT SUM(Total_Cost) FROM dbo.fact_cost) AS Total_Cost;

-- Date-filtered failure KPIs by site; compare with identical report selections.
SELECT s.Site_Name,
       COUNT(DISTINCT f.Failure_ID) AS Total_Failures,
       SUM(f.Downtime_Hours) AS Total_Downtime_Hours,
       SUM(f.Downtime_Hours) / NULLIF(COUNT(DISTINCT f.Failure_ID) * 1.0, 0) AS MTTR_Hours
FROM dbo.fact_failure f
JOIN dbo.dim_asset a ON a.Asset_ID = f.Asset_ID
JOIN dbo.dim_site s ON s.Site_ID = a.Site_ID
WHERE f.Failure_Date >= '2026-05-30' AND f.Failure_Date < '2026-08-30'
GROUP BY s.Site_Name;

-- Work-order snapshot: no date predicate, matching the documented model.
SELECT WorkOrder_Status, COUNT(DISTINCT WorkOrder_ID) AS Work_Orders,
       COUNT(DISTINCT CASE WHEN SLA_Breached_Flag = 1 THEN WorkOrder_ID END) AS SLA_Breaches,
       AVG(CAST(Actual_Hours AS decimal(18,4))) AS Average_Actual_Hours,
       AVG(CAST(Estimated_Hours AS decimal(18,4))) AS Average_Estimated_Hours,
       AVG(CAST(Actual_Hours AS decimal(18,4)))
         - AVG(CAST(Estimated_Hours AS decimal(18,4))) AS Average_Hours_Variance
FROM dbo.fact_work_order
GROUP BY WorkOrder_Status;

SELECT 100.0 * COUNT(DISTINCT CASE WHEN Planned_Flag = 1 THEN Maintenance_ID END)
       / NULLIF(COUNT(DISTINCT Maintenance_ID), 0) AS Planned_Maintenance_Pct
FROM dbo.fact_maintenance;
```

For date-filtered maintenance/cost reconciliation, use the same date bounds on `Maintenance_Date` / `Cost_Date`. Aggregate facts separately; joining raw facts together multiplies rows and distorts totals. SQL division above returns null for a zero denominator; the documented DAX display measures may show zero instead.

### Data-quality checks

```sql
-- Dimension key duplicates: expect no rows; repeat for dim_site[Site_ID].
SELECT Asset_ID, COUNT(*) AS Total_Rows
FROM dbo.dim_asset GROUP BY Asset_ID HAVING COUNT(*) > 1;

-- Orphan/null asset keys: expect zero; repeat for the other three facts.
SELECT COUNT(*) AS Unmatched_Assets
FROM dbo.fact_failure f
LEFT JOIN dbo.dim_asset a ON a.Asset_ID = f.Asset_ID
WHERE a.Asset_ID IS NULL;

-- Failure date derivation and coverage: expect zero.
SELECT COUNT(*) AS Invalid_Failure_Dates
FROM dbo.fact_failure
WHERE Failure_Date IS NULL
   OR Failure_DateTime IS NULL
   OR Failure_Date <> CAST(Failure_DateTime AS DATE)
   OR Failure_Date < '2026-05-30' OR Failure_Date >= '2026-08-30';

SELECT COUNT(*) AS Invalid_Planned_Status
FROM dbo.fact_maintenance
WHERE Planned_Flag IS NULL OR Planned_Status IS NULL
   OR Planned_Status <> CASE WHEN Planned_Flag = 1 THEN 'Planned' ELSE 'Unplanned' END;
```

Also check asset-to-site orphans, null dimension labels, fact business-key duplicates, and maintenance/cost date coverage (including unexpected time components). `(Blank)` slicer values were reported; diagnostics were proposed, but explicit resolution was not recorded. Do not assume hiding Blank resolves its cause.

## 7. Final acceptance checklist and handover

Checked items reflect the user's phase-completion confirmation; unchecked items preserve specific evidence gaps or future refresh checks.

- [x] Direct Lake semantic model created on EquipmentOperations_Warehouse with six business tables plus dim_date.
- [x] Site → asset → fact relationships and three documented date relationships configured.
- [x] Failure_Date and Planned_Status added in the Warehouse and exposed in the model.
- [x] Core, operational, insight and ranking measures created; corrected WorkOrder_Status naming used.
- [x] Operational window aligned to 2026-05-30–2026-08-29; 24×7 denominator documented.
- [x] Three analytical pages and Home/navigation completed.
- [x] Navigation, slicer/cross-filter testing and headline reconciliation reported complete.
- [ ] Retain explicit diagnostic evidence resolving the reported Blank slicer members.
- [ ] Before claiming date-filtered work-order reporting, implement and test its intended date relationship.
- [ ] On the next full refresh, verify derived-column population, relationships, date coverage and KPI reconciliation; save SQL results as evidence.

**Refresh sequence:** Finish the Phase 3 load → populate/verify derived columns → sync the model if schema changed → check relationships → update the operational window for new data → rerun reconciliation and interaction checks.

