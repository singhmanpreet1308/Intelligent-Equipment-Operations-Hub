# Phase 3 — Microsoft Fabric Data Foundation

**Project:** Intelligent Equipment Operations Hub
**Status:** Completed, as confirmed in the Phase 3 implementation conversation.
**Purpose:** Practical implementation record and future refresh runbook.

## 1. Objectives and delivered scope

- Move the Phase 2 equipment datasets into a Fabric medallion architecture.
- Preserve raw inputs, validate data with PySpark, and support explainable quarantine.
- Build Silver operational tables and Gold dimensions/facts for analytics.
- Serve six business tables through Fabric Warehouse.
- Validate KPI-critical fields and make Warehouse full refreshes safe to rerun.

This record summarizes the completed conversation; it is not a fresh audit of the live Fabric environment. The implemented source landing was manual CSV upload; the implemented orchestration covers Gold → Warehouse.

## 2. Architecture and Fabric items

```text
Seven source CSVs (manual upload)
    ↓
EquipmentOperations_Lakehouse / Files/bronze
    ↓ PySpark notebook
bronze_* Delta tables
    ↓ schema, duplicates, relationships, business rules
    ├── rejected work orders → Files/quarantine/work_orders
    ↓ validated data
silver_* Delta tables
    ↓ dimensions and transaction-grain facts
gold_* Delta tables
    ├── gold_fact_sensor_reading stays in Lakehouse
    ↓ PL_Gold_to_Warehouse: truncate → six copies
EquipmentOperations_Warehouse / dbo
    ↓ next phase
Power BI semantic model and decision dashboard
```

| Item                 | Name / purpose                                                                       |
| -------------------- | ------------------------------------------------------------------------------------ |
| Workspace            | Project Fabric workspace; suggested name was`Intelligent-Equipment-Operations-Hub` |
| Lakehouse            | `EquipmentOperations_Lakehouse`                                                    |
| Notebook             | `03_Bronze_Ingestion_Validation`                                                   |
| Pipeline             | `PL_Gold_to_Warehouse`                                                             |
| Warehouse            | `EquipmentOperations_Warehouse`                                                    |
| Raw landing folder   | `Files/bronze`                                                                     |
| Rejected-data folder | `Files/quarantine`                                                                 |
| Control folder       | `Files/control` — created; populated control logging was not evidenced            |

`bronze_*`, `silver_*`, and `gold_*` are table prefixes, not folders or separate schemas.

## 3. Setup and build sequence

1. Create the Lakehouse in the project workspace and create `bronze`, `quarantine`, and `control` under **Files**.
2. Upload `sites.csv`, `assets.csv`, `sensor_readings.csv`, `failures.csv`, `maintenance.csv`, `work_orders.csv`, and `costs.csv` into `Files/bronze`. Organizational OneDrive access was unavailable, so the proposed OneDrive/Dataflow route was replaced by this landing method.
3. Create the notebook as a separate workspace item. Attach `EquipmentOperations_Lakehouse` and **set it as default** so relative `Files/...` paths resolve.
4. Read CSVs with `header=True` and `inferSchema=True`; inspect schemas, counts, null profiles, and exact duplicates.
5. Write Bronze tables using `.format("delta").mode("overwrite").saveAsTable(...)`.
6. Run work-order quarantine diagnostics, foreign-key checks, and business-rule checks before promoting data.
7. Write Silver Delta tables in overwrite mode. The current source passed checks, so Silver reused the Bronze tables without additional cleaning or row removal.
8. Build Gold dimensions and facts, write in overwrite mode, and reconcile keys, relationships, counts, and KPI fields.
9. Create the Warehouse and six Copy Data activities; validate loads and add truncate-before-insert orchestration.

## 4. Exact tables, grain, and completed row counts

Counts below are the completed source snapshot. Bronze, Silver, and Gold reconcile to the same counts; future source versions may legitimately differ.

| Source CSV              | Bronze table               | Silver table               | Gold table                   | Rows per layer | Warehouse table / rows         |
| ----------------------- | -------------------------- | -------------------------- | ---------------------------- | -------------: | ------------------------------ |
| `sites.csv`           | `bronze_sites`           | `silver_sites`           | `gold_dim_site`            |              3 | `dbo.dim_site` / 3           |
| `assets.csv`          | `bronze_assets`          | `silver_assets`          | `gold_dim_asset`           |            150 | `dbo.dim_asset` / 150        |
| `sensor_readings.csv` | `bronze_sensor_readings` | `silver_sensor_readings` | `gold_fact_sensor_reading` |        324,000 | Not copied                     |
| `failures.csv`        | `bronze_failures`        | `silver_failures`        | `gold_fact_failure`        |             40 | `dbo.fact_failure` / 40      |
| `maintenance.csv`     | `bronze_maintenance`     | `silver_maintenance`     | `gold_fact_maintenance`    |            310 | `dbo.fact_maintenance` / 310 |
| `work_orders.csv`     | `bronze_work_orders`     | `silver_work_orders`     | `gold_fact_work_order`     |            310 | `dbo.fact_work_order` / 310  |
| `costs.csv`           | `bronze_costs`           | `silver_costs`           | `gold_fact_cost`           |            310 | `dbo.fact_cost` / 310        |

### Layer design

- **Bronze:** Preserve raw CSVs and materialize structured Delta copies with minimal transformation. Initial types were inferred and inspected.
- **Silver:** Validated operational records. This snapshot used direct Bronze → Silver copies because the checks passed; this is not an automatic rejection filter for future dirty inputs.
- **Gold:** Two dimensions and five facts, retaining business IDs and transaction-level detail. Facts were copied from their corresponding Silver tables without aggregation.

| Gold table                   | Grain / business key                                         |
| ---------------------------- | ------------------------------------------------------------ |
| `gold_dim_site`            | One site /`Site_ID`                                        |
| `gold_dim_asset`           | One asset /`Asset_ID`; relates to site through `Site_ID` |
| `gold_fact_sensor_reading` | One reading /`Reading_ID`                                  |
| `gold_fact_failure`        | One failure /`Failure_ID`                                  |
| `gold_fact_maintenance`    | One maintenance record /`Maintenance_ID`                   |
| `gold_fact_work_order`     | One work order /`WorkOrder_ID`                             |
| `gold_fact_cost`           | One cost record /`Cost_ID`                                 |

All five facts relate to assets through `Asset_ID`. Other transactional IDs remain available, but validation of every cross-fact relationship was not documented.

Dimension projections, deduplicated by their business keys:

- `gold_dim_site`: `Site_ID`, `Site_Name`, `Location`, `Region`, `Site_Type`, `Operational_Status`, `Commission_Date`.
- `gold_dim_asset`: `Asset_ID`, `Site_ID`, `Asset_Name`, `Asset_Type`, `Manufacturer`, `Model`, `Criticality`, `Asset_Status`.

## 5. Validation and quarantine

| Check                                                          | Completed result                                        |
| -------------------------------------------------------------- | ------------------------------------------------------- |
| File reads, schemas, counts and null profiles                  | Inspected across seven datasets                         |
| Exact duplicate rows in Bronze                                 | 0 in every dataset                                      |
| Assets → Sites; five transactional datasets → Assets         | All six checks: 0 orphan records, using left-anti joins |
| `Criticality` in `Low`, `Medium`, `High`, `Critical` | 0 invalid values                                        |
| `Load_Pct` and `Health_Score` between 0 and 100            | 0 out-of-range values                                   |
| Work orders scheduled before creation                          | 0 violations                                            |
| Parsed work-order creation / scheduled timestamps              | 0 nulls in each                                         |
| Gold dimension keys and relationships                          | Checked for uniqueness and orphan records               |
| Gold row reconciliation                                        | Counts match the table inventory                        |

Range checks do not replace null checks. Bronze exact-row duplicate checks also differ from business-key uniqueness checks.

### Work-order quarantine logic

- Invalid condition: `Scheduled_DateTime < Created_DateTime`.
- The implemented valid expression accepted `Scheduled_DateTime >= Created_DateTime` or a null scheduled value. Separate timestamp diagnostics confirmed **no nulls** in either timestamp for this snapshot.
- Add `DQ_Rule = "WO_SCHEDULED_BEFORE_CREATED"`, `DQ_Reason = "Scheduled_DateTime is earlier than Created_DateTime"`, and `Quarantine_Timestamp = current_timestamp()` to rejected rows.
- Write rejected rows as Delta, using overwrite mode, to `Files/quarantine/work_orders`; read the path back to verify the count.
- Final Phase 3 result: **310 input = 310 valid + 0 quarantined**. An empty quarantine display was expected.

The earlier Phase 2 expectation of 3 rejected / 307 valid did not match the uploaded Phase 3 data. The current Bronze checks are the reference; the reason for the source-version difference was not proven. No injected-invalid-record test was documented.

For future dirty inputs, explicitly route only accepted rows into Silver and define handling for null/unparseable timestamps. The completed Silver copy worked because this snapshot was clean. Overwrite quarantine retains the current result, not a historical reject log.

## 6. KPI readiness

Aggregations were exercised using the actual Gold columns:

| Area        | Fields / supported checks                                                       |
| ----------- | ------------------------------------------------------------------------------- |
| Failures    | Count; sum and average`Downtime_Hours`                                        |
| Maintenance | Count; sum and average`Duration_Hours`                                        |
| Work orders | Count; average`Actual_Hours`; count where `SLA_Breached_Flag = 1`           |
| Costs       | Sum and average`Total_Cost`; sum `Downtime_Cost`                            |
| Sensors     | Average`Health_Score`, average `Load_Pct`; count where `Anomaly_Flag = 1` |

Final completeness checks returned **0 nulls** for `Downtime_Hours`, `Duration_Hours`, `Actual_Hours`, `Total_Cost`, and `Health_Score` in their respective fact tables. Numeric KPI totals were not transcribed in the conversation, so they are not reproduced here.

Phase 4 still needs semantic relationships and agreed measure definitions. Average downtime is not automatically MTTR, and average cost per cost record is not automatically cost per work order.

## 7. Warehouse and Gold-to-Warehouse pipeline

Create `EquipmentOperations_Warehouse` through **New item → Warehouse** in the same workspace. Create `PL_Gold_to_Warehouse` through **New item → Data pipeline**.

For each Copy Data activity:

- Source: Lakehouse `EquipmentOperations_Lakehouse`, using the Gold table below.
- Destination: Warehouse `EquipmentOperations_Warehouse`, schema `dbo`.
- First load: **Auto create table**, manually enter the target name, verify column mapping, and use **Write behavior = Insert**.
- Subsequent refreshes: retain Insert behind the successful truncate dependency.

| Copy activity                  | Source → destination                                 |
| ------------------------------ | ----------------------------------------------------- |
| `Copy_Gold_Dim_Site`         | `gold_dim_site` → `dbo.dim_site`                 |
| `Copy_Gold_Dim_Asset`        | `gold_dim_asset` → `dbo.dim_asset`               |
| `Copy_Gold_Fact_Failure`     | `gold_fact_failure` → `dbo.fact_failure`         |
| `Copy_Gold_Fact_Maintenance` | `gold_fact_maintenance` → `dbo.fact_maintenance` |
| `Copy_Gold_Fact_Work_Order`  | `gold_fact_work_order` → `dbo.fact_work_order`   |
| `Copy_Gold_Fact_Cost`        | `gold_fact_cost` → `dbo.fact_cost`               |

The first activity name was confirmed; the remaining names follow the documented pipeline convention. All six copies were reported complete. The 324,000-row sensor fact remains in the Lakehouse, keeping the Warehouse focused on the business mart.

### Duplicate-load issue and permanent fix

Repeated Insert loads accumulated rows: `dim_site` reached **9 instead of 3**, and `dim_asset` reached **300 instead of 150**. Running the whole pipeline reran previously successful copies.

Recovery was to truncate all six Warehouse tables and run the complete load once. The user confirmed correct results. The permanent fix added a Script/SQL activity named `Truncate_Warehouse_Tables`, connected to `EquipmentOperations_Warehouse`:

```sql
TRUNCATE TABLE dbo.dim_site;
TRUNCATE TABLE dbo.dim_asset;
TRUNCATE TABLE dbo.fact_failure;
TRUNCATE TABLE dbo.fact_maintenance;
TRUNCATE TABLE dbo.fact_work_order;
TRUNCATE TABLE dbo.fact_cost;
```

```text
Truncate_Warehouse_Tables
    └── Succeeded → each of the six Copy Data activities (Insert)
```

All six copies must depend on truncate **Succeeded**; they may then run in parallel. Two complete runs against unchanged Gold data should both return **3, 150, 40, 310, 310, 310**. The user confirmed completion of this hardening step.

This is rerun-safe full refresh on successful completion. It does not provide an atomic refresh across all tables: a failed copy can leave a partial mart. Run one refresh at a time and recover by rerunning the complete pipeline, including truncate, rather than individual Insert activities.

## 8. Future reference / runbook

### Refresh and verify

1. Confirm the intended source snapshot and default Lakehouse attachment.
2. For changed source data, rerun ingestion → Bronze validation/quarantine → Silver → Gold → Gold validation. The Warehouse pipeline alone does not refresh upstream tables.
3. Confirm the six Warehouse targets already exist before running the hardened pipeline. A fresh rebuild requires initial table creation before its truncate step can succeed.
4. Run the entire `PL_Gold_to_Warehouse`; verify truncate and all copies succeeded.
5. Run this Warehouse count query and compare with current Gold counts:

```sql
SELECT 'dim_site' AS Table_Name, COUNT(*) AS Total_Rows FROM dbo.dim_site
UNION ALL SELECT 'dim_asset', COUNT(*) FROM dbo.dim_asset
UNION ALL SELECT 'fact_failure', COUNT(*) FROM dbo.fact_failure
UNION ALL SELECT 'fact_maintenance', COUNT(*) FROM dbo.fact_maintenance
UNION ALL SELECT 'fact_work_order', COUNT(*) FROM dbo.fact_work_order
UNION ALL SELECT 'fact_cost', COUNT(*) FROM dbo.fact_cost;
```

6. Check dimension business keys; each query should return no rows:

```sql
SELECT Site_ID, COUNT(*) AS Total_Rows
FROM dbo.dim_site GROUP BY Site_ID HAVING COUNT(*) > 1;

SELECT Asset_ID, COUNT(*) AS Total_Rows
FROM dbo.dim_asset GROUP BY Asset_ID HAVING COUNT(*) > 1;
```

7. Check relationships; each count should be zero:

```sql
SELECT 'asset_site' AS Check_Name, COUNT(*) AS Invalid_Rows
FROM dbo.dim_asset a LEFT JOIN dbo.dim_site s ON a.Site_ID = s.Site_ID
WHERE s.Site_ID IS NULL
UNION ALL
SELECT 'failure_asset', COUNT(*)
FROM dbo.fact_failure f LEFT JOIN dbo.dim_asset a ON f.Asset_ID = a.Asset_ID
WHERE a.Asset_ID IS NULL
UNION ALL
SELECT 'maintenance_asset', COUNT(*)
FROM dbo.fact_maintenance f LEFT JOIN dbo.dim_asset a ON f.Asset_ID = a.Asset_ID
WHERE a.Asset_ID IS NULL
UNION ALL
SELECT 'work_order_asset', COUNT(*)
FROM dbo.fact_work_order f LEFT JOIN dbo.dim_asset a ON f.Asset_ID = a.Asset_ID
WHERE a.Asset_ID IS NULL
UNION ALL
SELECT 'cost_asset', COUNT(*)
FROM dbo.fact_cost f LEFT JOIN dbo.dim_asset a ON f.Asset_ID = a.Asset_ID
WHERE a.Asset_ID IS NULL;
```

8. After pipeline changes, run twice against unchanged Gold and repeat validation. Save run results and counts as evidence; only then refresh downstream reporting.

### Known fixes

| Symptom                                             | Resolution used                                                                                                         |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| OneLake 400 error reading`Files/bronze/sites.csv` | Attach the correct Lakehouse, set it as default, restart session if needed; test`notebookutils.fs.ls("Files/bronze")` |
| Unresolved work-order date column                   | Use`Scheduled_DateTime` and `Created_DateTime`                                                                      |
| Unresolved sensor load column                       | Use`Load_Pct`                                                                                                         |
| Unresolved asset status column                      | Use`Asset_Status`                                                                                                     |
| Warehouse target missing on initial load            | Use Auto create table with schema`dbo`                                                                                |
| Count-query alias error                             | Use`Total_Rows`, or bracket `[RowCount]`                                                                            |
| Warehouse counts multiplied                         | Verify truncate-success dependencies and rerun the full pipeline                                                        |

Inspect `spark.table("table_name").columns` and `.printSchema()` before adapting transformations to a new source version.

## 9. Final acceptance criteria

The phase was closed as complete in the conversation against this gate:

- [X] Seven CSVs landed and Bronze Delta tables created.
- [X] Schema/null profiling and exact duplicate checks completed.
- [X] Six Bronze foreign-key checks and stated business rules passed.
- [X] Work-order quarantine logic evaluated: 310 valid, 0 quarantined.
- [X] Seven Silver and seven Gold tables created; row counts reconciled.
- [X] Gold key/relationship validation completed and KPI-critical null counts were zero.
- [X] Warehouse created and six business tables loaded.
- [X] Duplicate-load issue recovered and truncate-before-insert configured.
- [X] Rerun-safe validation step reported complete by the user.

**Next:** Phase 4 — Power BI / Semantic Model & Decision Dashboard. Future engineering extensions include connected source ingestion, explicit schemas, persisted run IDs/counts/watermarks, historical quarantine, and automated validation gates; these are not claimed as completed Phase 3 features.
