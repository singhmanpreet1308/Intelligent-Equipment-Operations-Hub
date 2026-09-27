
# Phase 8 — AI & Agentic Operations Assistant

## Objective

Build an AI-powered Equipment Operations Assistant capable of:

- answering equipment-related questions
- retrieving asset information
- investigating failures and maintenance activity
- explaining equipment condition
- recommending next actions
- triggering controlled workflows
- maintaining human approval for operational changes

---

## 8.1 AI Assistant Scope

### Target Users

- Operations Manager
- Maintenance Technician

### Core Use Cases

- Equipment health investigation
- Failure investigation
- Downtime analysis
- Maintenance history
- Work order investigation
- Recommended maintenance actions

### Operating Principle

Question → Retrieve → Analyse → Explain → Recommend → Human Review → Execute

---

## 8.2 Architecture

### Target Architecture

User
↓
Copilot Studio
↓
Tools / Power Automate
↓
Dataverse + Fabric
↓
Operational Data
↓
AI Recommendation
↓
Human Approval
↓
Operational Action

### Components

- Microsoft Copilot Studio
- Microsoft Dataverse
- Power Automate
- Microsoft Fabric
- Power BI
- External LLM API fallback
- Python test harness

---

## 8.3 Copilot Studio Agent

### Agent Name

Equipment Operations Agent

### Responsibilities

- understand natural-language operational questions
- retrieve operational data
- explain findings
- recommend actions
- request confirmation before changes
- never invent equipment information

---

## 8.4 Agent Instructions

Include the final Copilot instructions used in the agent.

---

## 8.5 Dataverse Integration

Initial operational tables:

- Site
- Asset
- Issue
- Maintenance Action

Dataverse Search was enabled.

Direct Dataverse Knowledge was evaluated but was not available in the current Copilot Studio environment.

Therefore Dataverse was accessed through Copilot tools/connectors.

---

## 8.6 Asset Retrieval Tool

### Tool

Get Asset Details

### Connector

Microsoft Dataverse

### Action

List rows from selected environment

### Table

Asset

### Important columns

- Asset_ID
- Asset Name
- Asset Type
- Asset Status
- Criticality
- Health Score
- Manufacturer
- Model
- Serial Number
- Open Issue Count

### Filter

Asset records are filtered using Asset_ID.

Example:

new_Asset_ID eq 'AST-001'

---

## 8.7 Final Target Architecture

Operations Manager / Technician
↓
Equipment Operations Agent
↓
Copilot Studio
↓
Dataverse / Fabric / Power Automate
↓
Operational Context
↓
AI Analysis
↓
Recommendation
↓
Human Confirmation
↓
Maintenance / Issue / Workflow Action

---

## 8.8 Phase 8 Acceptance Criteria

Phase 8 is complete when:

- Equipment Operations Agent is created
- Agent instructions are configured
- Dataverse retrieval tools are available
- Asset information can be queried
- AI response generation is validated
- live operational data integration works
- recommendations are grounded in operational data
- human approval is retained
- operational actions are controlled
- Copilot or fallback AI interface is demonstrated
