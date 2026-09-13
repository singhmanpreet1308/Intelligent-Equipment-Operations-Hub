# Intelligent Equipment Operations Hub

## Project Definition

The Intelligent Equipment Operations Hub is an end-to-end manufacturing
operations solution designed to improve visibility, control, and response
to equipment downtime and maintenance issues.

The solution represents a fictional manufacturer operating 50 equipment
assets across 3 manufacturing sites.

Operational equipment data is ingested and transformed using Microsoft
Fabric through a Bronze, Silver, and Gold medallion architecture. Power BI
provides operational KPI monitoring, while Dataverse acts as the governed
system of record for equipment issues and maintenance actions.

Power Apps enables engineers and operations teams to report and manage
equipment faults, while Power Automate controls critical maintenance
approvals and operational notifications.

Copilot Studio provides an authenticated conversational interface that
allows authorised users to query operational information and perform
controlled actions.

The complete operational process is:

Equipment Data
→ Fabric Data Platform
→ Power BI Monitoring
→ Fault Reporting
→ Dataverse Issue Management
→ Approval & Maintenance Workflow
→ Resolution
→ Operational Analytics
→ Copilot Interaction

The project uses synthetic data only and is designed as a portfolio
demonstration of Microsoft Fabric, Power Platform, Power BI, automation,
security, governance, and Agentic AI capabilities.

## Personas & Stakeholders

The Intelligent Equipment Operations Hub supports five primary personas.

### Engineer / Technician
Reports equipment faults, records issue details and tracks issue progress
using the Canvas App and Dataverse.

### Maintenance Manager
Reviews reported issues, triages maintenance work and approves critical
maintenance requests through the Model-driven App and Power Automate.

### Operations Manager
Monitors operational KPIs including equipment availability, downtime,
maintenance cost, open critical issues and resolution performance through
Power BI.

### Data / Operations Analyst
Validates operational data, investigates data-quality issues and analyses
equipment and maintenance performance using Microsoft Fabric, SQL and
Power BI.

### Business User / Supervisor
Uses the authenticated Copilot Studio agent to retrieve permitted
operational information, access procedures and perform controlled actions.

## Key Stakeholders

- Operations Management
- Maintenance Management
- Engineering Teams
- Data / BI Team
- Power Platform Administrators
- Microsoft Fabric Administrators
- Security / Governance Team
- Business Leadership


## Business Requirements & User Stories

### User Stories

US-01 — Operations KPI Monitoring
As an Operations Manager, I want to view site and asset KPIs so that I can
identify equipment downtime and performance issues quickly.

US-02 — Equipment Fault Reporting
As an Engineer, I want to report an equipment fault so that the issue can
be formally tracked and acted upon.

US-03 — Critical Maintenance Approval
As a Maintenance Manager, I want to approve or reject critical maintenance
requests so that high-impact maintenance work is controlled.

US-04 — Issue Resolution
As an operational user, I want to update and resolve maintenance issues so
that issue progress and resolution are accurately recorded.

US-05 — Operational AI Assistant
As an authorised user, I want to ask an authenticated agent about
procedures and live operational issues so that I can access information and
perform controlled actions quickly.

## Key Functional Requirements

- Ingest and transform operational equipment data.
- Apply Bronze, Silver and Gold medallion processing.
- Detect duplicates and invalid records.
- Provide equipment and site KPIs in Power BI.
- Store equipment issues in Dataverse.
- Support fault reporting through Power Apps.
- Support Reported → Triage → Approved → Resolved lifecycle.
- Trigger manager approval for critical issues.
- Provide authenticated Copilot access to procedures and live data.
- Require user confirmation before agent-triggered issue creation.
- Enforce role-based access and least privilege.
- Support controlled Dev/Test/Prod deployment.

## Project Scope & Boundaries

### In Scope

The Intelligent Equipment Operations Hub covers the complete operational
workflow from equipment performance monitoring through fault reporting,
maintenance approval, resolution and authenticated AI interaction.

The project includes:

- Synthetic operations data for 3 sites, 50 assets and 90 days
- Data-quality validation and duplicate handling
- Microsoft Fabric Bronze, Silver and Gold architecture
- Dataflow Gen2, Fabric Pipelines, PySpark and SQL
- Power BI operational analytics
- Dataverse issue and maintenance management
- Canvas App fault reporting
- Model-driven App issue lifecycle management
- Power Automate critical-maintenance approval
- Copilot Studio procedure Q&A and controlled operational actions
- Role-based security, DLP and environment strategy
- End-to-end validation and interview demonstration evidence

### Prototype / Limited Scope

The following capabilities may be demonstrated at prototype level:

- Fabric Warehouse maintenance-cost mart
- Paginated reporting
- Row-level security testing
- Scheduled reminder automation
- Custom HTTP connector
- Advanced flow telemetry
- End-to-end UAT packaging

Managed Dev/Test/Prod deployment is documented as an intended approach
and may be completed later as part of the governance phase.

### Out of Scope

The project does not include:

- Production or employer data
- Physical IoT or PLC/SCADA integration
- Real-time streaming telemetry
- Predictive maintenance machine learning
- ERP or SAP implementation
- Full CMMS replacement
- Production-scale infrastructure or SLA commitments

All project data is synthetic and the solution is designed as independent
portfolio evidence of hands-on Microsoft Fabric and Power Platform skills.

## KPI Definitions & Business Rules

### KPI Dictionary

| KPI | Definition | Unit |
|---|---|---|
| Downtime Hours | Total validated equipment downtime | Hours |
| Scheduled Hours | Total planned equipment operating time | Hours |
| Availability % | (Scheduled Hours - Downtime Hours) / Scheduled Hours | % |
| Maintenance Cost | Total maintenance expenditure | Currency |
| Open Critical Issues | Number of unresolved issues with Critical severity | Count |
| Resolution Hours | Closed DateTime - Opened DateTime | Hours |
| Average Resolution Hours | Average resolution time of resolved issues | Hours |

### Business Rules

BR-01: Every operational record must reference a valid Asset ID.

BR-02: Asset ID + Operation Date must uniquely identify an asset-day
operational record.

BR-03: Downtime Hours must satisfy:
0 <= Downtime Hours <= Scheduled Hours.

BR-04: Critical issues must trigger manager approval.

BR-05: The standard issue lifecycle is:
Reported → Triage → Approved → Resolved.

BR-06: A resolved issue must contain a valid Closed DateTime.

BR-07: Copilot must request explicit confirmation before creating an issue.

BR-08: Agent-created critical issues must still follow the standard manager
approval process.

BR-09: Operational information must respect role- and site-based access.

## Acceptance Criteria

### Phase 1 Acceptance

Phase 1 is considered complete when:

- The business problem is clearly defined.
- Personas and stakeholders are identified.
- Five core user stories are documented.
- Functional and non-functional requirements are defined.
- Project scope and exclusions are documented.
- KPI definitions and business rules are agreed.
- Target architecture is documented.
- Environment and naming conventions are established.
- Initial backlog and decision log are created.

### Solution Acceptance

The final solution must demonstrate:

#### Data
- 3 sites and 50 assets.
- 90 days of synthetic operational data.
- Detection and handling of missing IDs, duplicates and invalid downtime.
- Bronze, Silver and Gold processing.
- Repeatable pipeline execution without duplicate records.

#### Analytics
- Downtime Hours.
- Availability %.
- Maintenance Cost.
- Open Critical Issues.
- Average Resolution Hours.
- Overview and Asset Detail reporting.

#### Applications
- Engineers can report equipment faults.
- Dataverse stores the Issue record.
- Model-driven App supports:
  Reported → Triage → Approved → Resolved.

#### Automation
- Critical issues trigger manager approval.
- Approval and rejection paths are handled and recorded.

#### Security
- Role- and site-based access is enforced.
- Unauthorised site access is denied.

#### Copilot
- Procedure Q&A works.
- Live critical issue query works.
- Controlled issue creation works.
- Explicit confirmation is required before issue creation.
- Critical issues still require manager approval.

#### End-to-End
The solution must demonstrate:

Data Ingestion
→ KPI Monitoring
→ Issue Submission
→ Approval
→ Agent Query
→ Resolution
→ Analytics Refresh

## Target Architecture

The Intelligent Equipment Operations Hub uses Microsoft Fabric as the
analytical data platform and Microsoft Power Platform as the operational
application and automation layer.

### Data Flow

Synthetic Operational Data
→ Dataflow Gen2
→ Fabric Pipeline
→ OneLake Bronze
→ PySpark Validation
→ Silver
→ Gold Delta Tables
→ Power BI Semantic Model
→ Operational Reporting

### Operational Flow

Engineer
→ Canvas App
→ Dataverse Issue
→ Model-driven App
→ Power Automate Approval
→ Resolution

### Agent Flow

User
→ Copilot Studio
→ SharePoint Knowledge / Dataverse
→ Power Automate Actions
→ Controlled Dataverse Update

### Security

Microsoft Entra ID provides authentication, while Dataverse security roles,
site-owner teams, Power BI RLS, DLP policies and least-privilege access
govern data and application access.

### ALM

The target release strategy follows:

Development
→ Test
→ Production

using Power Platform solutions, environment variables, connection
references and deployment pipelines, alongside separate Fabric/Power BI
workspace promotion.

## Environment & Naming Strategy

### Environments

The target environment strategy is:

DEV → TEST → PROD

Power Platform development will occur in DEV using an unmanaged solution.
TEST and PROD will use managed deployment where supported.

### Power Platform

Environment Names:
- IEOH-DEV
- IEOH-TEST
- IEOH-PROD

Solution:
- IntelligentEquipmentOperationsHub

Publisher Prefix:
- ieoh

Dataverse Tables:
- ieoh_Site
- ieoh_Asset
- ieoh_Issue
- ieoh_MaintenanceAction

Applications:
- IEOH - Fault Reporting
- IEOH - Operations Management

Business Process Flow:
- IEOH - Issue Lifecycle

Power Automate:
- IEOH - Critical Issue Approval
- IEOH - Create Issue
- IEOH - Overdue Issue Reminder
- IEOH - Equipment Status API

Security Roles:
- IEOH Engineer
- IEOH Maintenance Manager
- IEOH Analyst

Copilot:
- IEOH Operations Assistant

### Microsoft Fabric

Workspaces:
- IEOH-Fabric-DEV
- IEOH-Fabric-TEST
- IEOH-Fabric-PROD

Lakehouse:
- lh_ieoh_operations

Warehouse:
- wh_ieoh_maintenance

Pipelines:
- pl_ieoh_ingest_operations
- pl_ieoh_ingest_maintenance
- pl_ieoh_bronze_to_silver
- pl_ieoh_silver_to_gold

Notebooks:
- nb_ieoh_bronze_to_silver
- nb_ieoh_silver_to_gold
- nb_ieoh_data_quality

Gold Tables:
- dim_site
- dim_asset
- dim_date
- fact_asset_daily_operations
- fact_maintenance

### Power BI

Semantic Model:
- IEOH Operations Model

Report:
- IEOH Operations Dashboard

Report Pages:
- Overview
- Asset Detail

## Initial Delivery Backlog

The project backlog is organised according to the ten project phases.

Priority definitions:

- High — required for the end-to-end solution
- Medium — important enterprise/interview capability
- Low — prototype or enhancement

The core delivery sequence is:

Requirements & Architecture
→ Synthetic Data
→ Fabric Data Foundation
→ Power BI
→ Dataverse
→ Power Apps
→ Power Automate
→ Governance
→ Copilot Studio
→ End-to-End Validation

The project will prioritise a working vertical business flow before adding
prototype capabilities.

## Architecture Decision Log

### ADR-01 — Synthetic Data
Use synthetic data only. No employer or production data will be used.

### ADR-02 — Microsoft Fabric Analytical Platform
Use Fabric, OneLake and Bronze/Silver/Gold processing for analytical data.

### ADR-03 — Dataverse Operational System
Use Dataverse as the governed operational system of record for Sites,
Assets, Issues and Maintenance Actions.

### ADR-04 — Separate Analytical and Operational Workloads
Fabric and Power BI will serve analytical workloads.
Dataverse and Power Apps will serve operational workloads.

### ADR-05 — Dual Application Strategy
Use a Canvas App for rapid fault reporting and a Model-driven App for
structured issue management.

### ADR-06 — Human Approval for Critical Issues
Critical maintenance requests must be approved by a manager.

### ADR-07 — Controlled Agent Actions
Copilot must obtain confirmation before creating an issue and may not
bypass existing approval controls.

### ADR-08 — Medallion Architecture
Use Bronze for raw data, Silver for validated data and Gold for
business-ready analytical models.

### ADR-09 — Data Quality Quarantine
Invalid records will be detected and quarantined rather than silently lost.

### ADR-10 — Idempotent Processing
Re-running the same batch must not create duplicate analytical records.

### ADR-11 — Limited Warehouse Scope
Fabric Warehouse will be used only for a small maintenance-cost mart.

### ADR-12 — Environment Strategy
Use DEV → TEST → PROD as the target lifecycle, with unmanaged development
and managed downstream Power Platform deployment where supported.

