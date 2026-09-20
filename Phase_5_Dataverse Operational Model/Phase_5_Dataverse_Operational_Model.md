# Phase 5 — Dataverse Operational Model

## 1. Phase Objective

The objective of Phase 5 is to build the **operational data model in Microsoft Dataverse** for the Intelligent Equipment Operations Hub.

Dataverse is used as the operational system of record for equipment-management activities that will later support:

- Power Apps operational interfaces
- Power Automate workflows and approvals
- Issue and maintenance tracking
- Asset-to-site relationship management
- Integration with Microsoft Fabric and Power BI
- Future AI-assisted operational workflows

The analytical model remains in Microsoft Fabric, while Dataverse provides the transactional/operational layer.

---

## 2. Solution Setup

A dedicated Power Platform solution was created:

```text
Intelligent Equipment Operations Hub
```

The solution was set as the **Preferred Solution** so that new Power Platform components are created inside the project solution.

### Publisher

The environment currently uses the default `new_` publisher prefix.

Examples of resulting schema names include:

```text
new_asset
new_issue
new_maintenanceaction
new_eoh_site
```

Although a custom prefix such as `eoh_` would normally be preferable, the existing schema was retained to avoid recreating components after development had already started.

---

## 3. Core Dataverse Tables

Four custom operational tables were created:

```text
Site
  └── Asset
        └── Issue
              └── Maintenance Action
```

### 3.1 Site

Purpose: stores manufacturing/operational site master data.

| Column | Type | Purpose |
|---|---|---|
| Site Name | Primary name | Human-readable site name |
| Site ID | Text | Business identifier such as `SITE001` |
| Location | Text | Site location |
| Region | Text | Operational/geographic region |
| Country | Text | Country |
| Site Type | Choice | Type of operational site |
| Operational Status | Choice | Current site status |

#### Site Type Choices

```text
Manufacturing
Warehouse
Distribution
Service
Other
```

#### Operational Status Choices

```text
Active
Inactive
Maintenance
```

---

## 4. Asset Table

Purpose: stores equipment/asset master data and links every asset to a Site.

| Column | Type | Purpose |
|---|---|---|
| Asset Name | Primary name | Human-readable asset name |
| Asset ID | Text | Business identifier |
| Asset Type | Choice | Equipment category |
| Manufacturer | Text | Manufacturer |
| Model | Text | Equipment model |
| Serial Number | Text | Serial number |
| Criticality | Choice | Operational criticality |
| Asset Status | Choice | Current operating status |
| Installation Date | Date | Installation date |
| Health Score | Whole number | Future operational health score |
| Site_ID | Lookup → Site | Parent Site |
| Open Issue Count | Rollup | Number of unresolved issues |

#### Asset Type Choices

Aligned to the actual source `assets.csv` values:

```text
Motor
Pump
Compressor
Conveyor
Fan
```

#### Asset Status Choices

```text
Active
Maintenance
Offline
```

#### Criticality Choices

```text
Low
Medium
High
```

`Critical` may be retained as a future value if required.

---

## 5. Issue Table

Purpose: stores operational equipment issues/failures linked to individual assets.

| Column | Type | Purpose |
|---|---|---|
| Issue Title | Primary name | Human-readable issue description |
| Issue ID | Text | Business issue identifier |
| Asset | Lookup → Asset | Related equipment asset |
| Severity | Choice | Issue severity |
| Issue Status | Choice | Current issue status |
| Description | Multiple lines text | Issue description |
| Reported Date | Date and time | Issue creation/detection time |
| Resolved Date | Date and time | Resolution time |
| Detection Method | Choice | How the issue was detected |
| Root Cause | Multiple lines text | Root cause |
| Production Impact | Choice | Operational impact |
| Resolution Hours | Formula/Calculated | Time to resolution |

#### Severity Choices

```text
Low
Medium
High
Critical
```

#### Issue Status Choices

```text
Open
In Progress
Resolved
Closed
```

Default:

```text
Open
```

#### Detection Method Choices

```text
Sensor
Operator
Inspection
Maintenance
Automated Alert
Other
```

#### Production Impact Choices

```text
None
Low
Medium
High
Critical
```

---

## 6. Maintenance Action Table

Purpose: stores operational maintenance actions linked to Issues.

| Column | Type | Purpose |
|---|---|---|
| Action Title | Primary name | Maintenance action description |
| Action ID | Text | Business identifier |
| Issue | Lookup → Issue | Related issue |
| Action Type | Choice | Maintenance activity type |
| Action Status | Choice | Current activity status |
| Assigned To | Lookup → User | Assigned user/team |
| Scheduled Date | Date and time | Planned execution date |
| Started Date | Date and time | Actual start |
| Completed Date | Date and time | Completion date |
| Notes | Multiple lines text | Maintenance notes |
| Estimated Cost | Currency | Planned cost |
| Actual Cost | Currency | Actual cost |

#### Action Type Choices

```text
Inspection
Repair
Replacement
Preventive Maintenance
Corrective Maintenance
Calibration
Other
```

#### Action Status Choices

```text
Planned
Assigned
In Progress
Completed
Cancelled
```

Default:

```text
Planned
```

---

## 7. Dataverse Relationships

The following 1:N relationships were implemented through lookup columns:

```text
Site 1:N Asset
Asset 1:N Issue
Issue 1:N Maintenance Action
```

Operational relationship chain:

```text
Site
  ↓
Asset
  ↓
Issue
  ↓
Maintenance Action
```

The standard Dataverse `User` table is referenced through the `Assigned To` lookup in the Maintenance Action table.

---

## 8. Alternate Keys

Business keys were created to protect against duplicate operational records.

| Table | Alternate Key |
|---|---|
| Site | Site ID Key |
| Asset | Asset ID Key |
| Issue | Issue ID Key |
| Maintenance Action | Action ID Key |

Example:

```text
Site ID = SITE001
Asset ID = AST001
Issue ID = ISS001
Action ID = ACT001
```

These business keys remain separate from the internal Dataverse GUID primary keys.

---

## 9. Calculated and Rollup Columns

### 9.1 Resolution Hours

Table:

```text
Issue
```

Purpose: calculate how long an issue took to resolve.

Conceptual logic:

```text
Resolution Hours =
Resolved Date - Reported Date
```

Using Power Fx:

```powerfx
If(
    IsBlank('Resolved Date'),
    Blank(),
    DateDiff(
        'Reported Date',
        'Resolved Date',
        TimeUnit.Hours
    )
)
```

### 9.2 Open Issue Count

Table:

```text
Asset
```

Type:

```text
Whole Number — Rollup
```

Related table:

```text
Issue
```

Filter:

```text
Issue Status = Open
OR
Issue Status = In Progress
```

Aggregation:

```text
COUNT
```

Purpose: provides the number of unresolved issues for each asset.

---

## 10. Operational Data Import Strategy

Data is loaded in parent-to-child order:

```text
1. Site
2. Asset
3. Issue
4. Maintenance Action
```

This order is required because Dataverse lookup relationships depend on parent records already existing.

---

## 11. Site Data Import

Site records were imported first.

Example validated Site record:

```text
Site Name: Belfast Operations Plant
Site ID: SITE001
Location: Belfast
Region: Northern Ireland
Site Type: Manufacturing
Operational Status: Active
```

Example Dataverse Site GUID:

```text
26d274a5-78b4-f111-aaad-6045bd0d0e15
```

The internal GUID is different from the business identifier `SITE001`.

---

## 12. Asset Source Data

The actual asset source contains:

```text
Asset_ID
Site_ID
Asset_Name
Asset_Type
Manufacturer
Model
Serial_Number
Install_Date
Criticality
Rated_Capacity
Capacity_Unit
Asset_Status
Expected_Life_Years
Warranty_End_Date
```

The initial Dataverse Asset model intentionally remains lean.

### Mapped Fields

```text
Asset_ID       → Asset ID
Asset_Name     → Asset Name
Asset_Type     → Asset Type
Manufacturer   → Manufacturer
Model          → Model
Serial_Number  → Serial Number
Install_Date   → Installation Date
Criticality    → Criticality
Asset_Status   → Asset Status
Site_ID/GUID   → Site lookup
```

### Currently Unmapped Source Fields

```text
Rated_Capacity
Capacity_Unit
Expected_Life_Years
Warranty_End_Date
```

These can be added later if operational requirements justify them.

---

## 13. Asset Import Troubleshooting

### 13.1 Duplicate Asset Name Column

Initially, the Asset table contained:

```text
Asset Name → custom column
Name       → Dataverse primary name column
```

This caused import confusion.

The duplicate custom Asset Name column was removed and the Dataverse primary name column was retained as:

```text
Asset Name
```

### 13.2 Choice Value Validation Errors

Dataverse Choice fields require source labels to match configured Choice values.

Example ingestion errors:

```text
The value Fan is not a valid value for the Asset Type choice field.
The value Active is not a valid value for the Asset Status choice field.
```

The Dataverse Choices were aligned to the actual source values.

Actual Asset Type values:

```text
Motor
Pump
Compressor
Conveyor
Fan
```

Actual Asset Status values:

```text
Active
Maintenance
Offline
```

Actual Criticality values:

```text
Low
Medium
High
```

### 13.3 Lookup GUID Error

A major lookup ingestion error occurred:

```text
SITE001 is not a valid primary id Guid value.
```

Cause:

`SITE001` is a business identifier, while the Dataverse Asset `Site` lookup expects a Dataverse row reference/GUID unless an alternate-key lookup is explicitly configured by the import mechanism.

Therefore:

```text
SITE001 ≠ Dataverse GUID
```

Example:

```text
Business ID:
SITE001

Dataverse Site GUID:
26d274a5-78b4-f111-aaad-6045bd0d0e15
```

### Resolution Approach

A Dataverse-specific asset import file was prepared with an additional column:

```text
Site_GUID
```

Example transformation:

```python
site_map = {
    "SITE001": "26d274a5-78b4-f111-aaad-6045bd0d0e15",
    "SITE002": "<SITE002_GUID>",
    "SITE003": "<SITE003_GUID>"
}

assets["Site_GUID"] = assets["Site_ID"].map(site_map)
```

Validation:

```python
print(
    assets[["Site_ID", "Site_GUID"]]
    .drop_duplicates()
)

print("Missing Site GUIDs:", assets["Site_GUID"].isna().sum())
```

Required result:

```text
Missing Site GUIDs: 0
```

The intended Dataverse import mapping is:

```text
Site_GUID → Site lookup
Site_ID   → Unmapped
```

The original `Site_ID` column must not be mapped directly to the lookup if Dataverse interprets it as a GUID.

---

## 14. Maintenance Action Import Preparation

Maintenance and Work Order data were combined into a single Dataverse import structure.

Final dataframe columns:

```text
Action_ID
Action_Title
Issue_ID
Action_Type
Action_Status
Scheduled_Date
Started_Date
Completed_Date
Notes
Estimated_Cost
Actual_Cost
```

Example export:

```python
maintenance_action_import.to_csv(
    "maintenance_action_import.csv",
    index=False
)
```

The Dataverse alternate key selected for import is:

```text
Action ID Key
```

---

## 15. Relationship Validation Plan

Once all operational data is successfully loaded, validate one full record chain:

```text
SITE001
  ↓
Asset
  ↓
Issue
  ↓
Maintenance Action
```

Validation criteria:

```text
Asset.Site = correct Site
Issue.Asset = correct Asset
Maintenance Action.Issue = correct Issue
```

Business logic validation:

```text
Resolution Hours > 0
for resolved Issues

Open Issue Count =
number of related Open/In Progress Issues
```

---

## 16. Phase 5 Current Status

### Completed

- Dataverse solution created
- Preferred solution configured
- Site table created
- Asset table created
- Issue table created
- Maintenance Action table created
- Core lookup relationships created
- Choice columns configured
- Alternate business keys created
- Resolution Hours logic created
- Open Issue Count rollup created
- Site seed data imported
- Asset import mapping corrected
- Asset Choice values aligned with source data
- Duplicate Asset Name field removed
- Maintenance Action import structure prepared

### In Progress

- Final successful Asset import
- Site lookup GUID resolution during Asset ingestion
- End-to-end relationship validation

### Pending

- Final Issue data validation
- Final Maintenance Action relationship validation
- Security roles and access model
- Final Phase 5 acceptance test

---

## 17. Phase 5 Acceptance Criteria

Phase 5 will be considered complete when:

- [ ] Site records are successfully loaded
- [ ] Asset records are successfully loaded
- [ ] Every Asset resolves to a valid Site
- [ ] Issue records resolve to valid Assets
- [ ] Maintenance Actions resolve to valid Issues
- [ ] Alternate keys are active
- [ ] Choice fields accept all source values
- [ ] Resolution Hours returns valid values
- [ ] Open Issue Count returns expected values
- [ ] No duplicate business IDs exist
- [ ] No orphan lookup records exist
- [ ] Security/ownership configuration is validated
- [ ] Full Site → Asset → Issue → Maintenance Action chain is tested

---

## 18. Phase 5 Architecture Outcome

At completion, the Dataverse operational layer will provide:

```text
Microsoft Fabric
    ↓
Analytical / Historical Data

Dataverse
    ↓
Operational Master & Transactional Data
    ↓
Power Apps
    ↓
Power Automate
    ↓
Operational Workflows / Approvals / Actions
```

This provides the transactional foundation required for the next Power Platform phases of the Intelligent Equipment Operations Hub.
