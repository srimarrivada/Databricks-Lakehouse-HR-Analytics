# Medallion Data Flow — Integration-Level View

## Overview

This document explains how Medallion Architecture is applied across HR integration analytics in Azure Databricks.

The design follows a standard lakehouse flow:

> Source Systems → Bronze → Silver → Gold → Reporting Views → BI Consumption

The Medallion Architecture separates raw data capture, cleaned/validated transformation, analytical modeling, and dashboard-ready consumption into distinct layers.

This is a sanitized portfolio artifact created to demonstrate lakehouse architecture and data engineering design thinking. It is not client documentation and does not include proprietary data, production code, credentials, internal system names, or real employee/customer data.

---

## Architecture Diagram

![Medallion Data Flow — Integration-Level View](../diagrams/medallion-data-flow-integration-view.png)

---

## Medallion Architecture Layers

### Bronze Layer — Raw Capture / Audit Trail

The Bronze layer preserves source data as received.

Key responsibilities:

- Capture raw records from source systems
- Preserve original file/API/database structure where practical
- Add audit metadata such as load timestamp, source name, and file name where applicable
- Support replay and reconstruction of Silver/Gold layers
- Maintain a durable audit trail

Typical source patterns:

- JDBC incremental extraction
- JDBC full overwrite for small reference data
- CSV/file capture
- File-level deduplication
- Watermark-based processing

---

### Silver Layer — Cleaned / Typed / Validated

The Silver layer standardizes and validates data from Bronze.

Key responsibilities:

- Type casting
- Deduplication
- Standardization
- Business classification derivation
- Data quality boundary
- Enrichment from secondary sources
- Filtering invalid records
- Normalizing source-specific structures

Silver data is designed to be clean, consistent, and ready for dimensional modeling.

---

### Gold Layer — Analytical / Dimensional Model

The Gold layer provides analytics-ready dimensional models.

Key responsibilities:

- Resolve surrogate keys
- Build fact tables
- Build local dimensions
- Reuse federated conformed dimensions
- Calculate measures
- Apply business rules
- Create star/constellation schemas
- Run data quality checks
- Feed reporting views and dashboards

The Gold layer combines local Delta Lake dimensions and facts with federated conformed dimensions from the enterprise SQL Server HR DataHub / data warehouse.

---

### Reporting Views — Semantic Consumption Layer

Reporting views sit on top of Gold tables and simplify BI consumption.

Key responsibilities:

- Pre-join facts and dimensions
- Hide surrogate keys and technical complexity
- Expose business-friendly column names
- Apply reusable calculations and KPI logic
- Support dashboard-ready datasets
- Provide security-aware consumption views

---

## Integration-Level Flow

The platform applies Medallion Architecture across three integration analytics domains:

1. New Hire Operational Analytics
2. Recruiting Pipeline Analytics
3. Training Enrollment Analytics

Each integration follows the same architectural pattern but uses different ingestion methods, transformation logic, Gold-layer modeling patterns, DQ checks, and BI consumption designs.

---

## 1. New Hire Operational Analytics

### Source Pattern

New Hire data is sourced from operational event tables through JDBC extraction.

Representative sources:

- New hire event table
- Worker reference table
- Downstream notification status
- Downstream service configuration

Refresh pattern:

- Runs every 2 hours
- Uses incremental extraction where applicable
- Uses full overwrite for small reference/configuration tables

### Bronze Layer

Bronze captures raw operational tables and adds audit metadata.

Examples:

- New hire events
- Worker reference data
- Downstream notification status
- Downstream service configuration

Key characteristics:

- Raw preservation
- Audit trail
- Watermark tracking
- Durable replay capability

### Silver Layer

Silver applies cleaning and standardization.

Example transformations:

- Parse event payloads
- Derive event type
- Standardize worker identifiers
- Cast data types
- Deduplicate records
- Derive error flags
- Normalize notification status

### Gold Layer

Gold creates an operational analytics model.

Representative dimensions:

- Federated worker dimension
- Event type
- Downstream service
- Notification status
- Federated date/job/company/location/organization context

Representative facts:

- New hire event fact
- Downstream notification fact

Key measures:

- Processing time
- Notification count
- Days pending
- Delivery time
- Overdue indicator
- Success/failure status

### Reconciliation Pattern

New hire analytics uses a reconciliation pattern because newly hired workers may not yet exist in the enterprise worker dimension.

Pattern:

1. Gold fact initially allows unresolved `Worker_SK`
2. Worker is still displayed using fallback identity attributes
3. Daily reconciliation job updates unresolved keys after enterprise worker dimension refresh
4. Records unresolved beyond threshold become DQ alerts

This distinguishes valid timing delays from actual source data issues.

### Consumption

New Hire analytics is consumed through a Databricks SQL dashboard for operational monitoring.

Reasoning:

- Small operational audience
- Need for near-real-time visibility
- Users already work in Databricks
- Live SQL queries avoid separate BI refresh cycles

---

## 2. Recruiting Pipeline Analytics

### Source Pattern

Recruiting data is sourced from a combination of operational database tables and file feeds.

Representative sources:

- Candidate pipeline data
- Requisition data
- Candidate education enrichment files

Refresh pattern:

- Runs every 2 hours
- Uses incremental extraction for operational data
- Captures file feeds before cleanup/expiration
- Uses file-level deduplication for repeated file drops

### Bronze Layer

Bronze captures both operational records and file-based records.

Examples:

- Candidate pipeline records
- Requisition data
- Candidate enrichment files

Key characteristics:

- Raw preservation
- File-level audit metadata
- File capture before cleanup
- Replayable source history

### Silver Layer

Silver standardizes candidate and requisition data.

Example transformations:

- Derive candidate status category
- Derive stage name and stage order
- Identify terminal and positive outcome stages
- Enrich candidates using education file data
- Type cast dates and flags
- Deduplicate candidate status records

### Gold Layer

Gold creates a recruiting pipeline analytics model.

Representative dimensions:

- Federated worker dimension for recruiter identity
- Candidate stage dimension
- Recruiting category dimension
- Requisition dimension

Representative facts:

- Candidate pipeline fact
- Requisition snapshot fact

Key modeling patterns:

- Step dimension for sequential candidate stages
- Window functions for pipeline movement
- Periodic snapshot for requisition health
- Candidate enrichment for education analytics

Example measures:

- Days in stage
- Days in pipeline
- Stage sequence number
- Conversion rate
- Drop-off rate
- Open requisition count
- Candidate count

### Reconciliation Pattern

Recruiting candidates are external applicants and are not expected to resolve to the enterprise worker dimension.

Therefore:

- Candidate identity is modeled separately
- Recruiter identity resolves through federated worker dimension
- Unresolved recruiter keys are treated as DQ issues
- Candidate-worker reconciliation is intentionally not required

### Consumption

Recruiting analytics is consumed through Power BI.

Reasoning:

- Broader business audience
- Need for recruiting leadership dashboards
- Power BI supports familiar enterprise reporting experience
- Row-level security supports recruiter-level access control
- Import refresh is sufficient for daily business review

---

## 3. Training Enrollment Analytics

### Source Pattern

Training analytics is sourced from learning and event-management platforms using a combination of file feeds and operational tables.

Representative sources:

- Training session files
- Learner enrollment files
- Instructor data
- Session mapping data

Refresh pattern:

- Runs every 3 hours
- Captures files before encryption or downstream delivery
- Uses JDBC extraction for operational lookup tables
- Uses incremental processing where applicable

### Bronze Layer

Bronze captures training session, learner, instructor, and session mapping data.

Key characteristics:

- Raw file preservation
- Capture before file transformation or encryption
- Audit metadata
- Source reconstruction capability

### Silver Layer

Silver cleans and standardizes training data.

Example transformations:

- Cast session capacity fields
- Normalize enrollment statuses
- Derive active/cancelled indicators
- Trim identifiers
- Remove consistently empty attributes
- Standardize timestamp fields

### Gold Layer

Gold creates a training enrollment analytics model.

Representative dimensions:

- Federated worker dimension
- Federated worker organization dimension
- Session dimension
- Enrollment status dimension

Representative facts:

- Learner enrollment fact
- Instructor assignment fact

Key modeling patterns:

- Dual-key session lookup
- Worker dimension role-playing
- Learner and instructor represented as different worker roles
- Coverage analysis using federated worker organization context

Example measures:

- Enrollment count
- Registered count
- Withdrawal count
- Session utilization
- Instructor workload
- Training coverage

### DQ Alert Pattern

Training participants and instructors are expected to be existing workers.

Therefore:

- Unresolved worker keys are treated as DQ issues
- No timing-based reconciliation is expected
- DQ alerts trigger source investigation

This differs from New Hire, where unresolved worker keys may be valid temporarily.

### Consumption

Training analytics is consumed through Power BI.

Reasoning:

- Audience includes HR, learning, and leadership users
- Daily scheduled refresh is sufficient
- Power BI provides familiar enterprise dashboarding
- Data can be refreshed after upstream learning data cycles complete

---

## Shared Data Quality Logging

All integrations write DQ results to a shared Gold utility table. 

Representative DQ categories:

- Unresolved worker keys
- Unresolved recruiter keys
- Unresolved session keys
- Unmapped stage values
- Volume anomalies
- SLA / overdue checks
- Completeness checks
- Business rule threshold checks

DQ result pattern:

- Integration name
- Check name
- Check category
- Severity
- Status: PASS / WARN / FAIL
- Failed record count
- Run timestamp
- Details for investigation

This enables cross-integration DQ monitoring and trend analysis. In this design, DQ results are logged for monitoring, alerting, and investigation. Databricks integration DQ is alert-oriented and does not automatically block every pipeline run.

---

## Medallion Layer Comparison

| Area | New Hire | Recruiting | Training |
|---|---|---|---|
| Refresh | Every 2 hours | Every 2 hours | Every 3 hours |
| Bronze pattern | JDBC tables | JDBC + file capture | File capture + JDBC |
| Silver focus | Event parsing and notification cleanup | Candidate stage classification and education enrichment | Session/enrollment cleanup |
| Gold model | Operational event analytics | Recruiting pipeline analytics | Training enrollment analytics |
| Shared dimension | Worker identity | Recruiter identity | Worker + worker org context |
| Key pattern | Worker_SK reconciliation | Step dimension + window functions | Dual-key session lookup |
| DQ strategy | Reconcile temporary unresolved workers | Alert on unresolved recruiter/stage/req | Alert on unresolved worker/session |
| BI tool | Databricks SQL | Power BI | Power BI |
| Audience | Operations team | Recruiters / HR leadership | Learning / HR leadership |

---

## Key Design Decisions

### 1. Bronze retention supports reconstruction

Bronze stores raw source data so Silver and Gold can be rebuilt if transformation logic changes.

### 2. Silver acts as the data quality boundary

Silver standardizes and validates source data before dimensional modeling.

### 3. Gold is modeled for analytics consumption

Gold uses facts, dimensions, measures, and reporting views to support dashboard-ready analytics.

### 4. Federation avoids dimension replication

Conformed worker and organization dimensions remain in the enterprise data warehouse and are accessed through Lakehouse Federation.

### 5. Reconciliation is used only when timing delay is expected

New Hire uses reconciliation because new workers may arrive before the enterprise worker dimension refresh.

### 6. DQ alerts are used when unresolved keys indicate source problems

Training workers and instructors should already exist, so unresolved keys are treated as data quality issues.

### 7. BI tool choice depends on audience and freshness

Databricks SQL supports operational, near-real-time monitoring. Power BI supports broader enterprise analytics and self-service consumption.

---

## Confidentiality Note

This document is a sanitized portfolio artifact created for data architecture demonstration purposes. It does not include proprietary client documentation, confidential data, production code, credentials, internal system names, or real employee/customer data.
