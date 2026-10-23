# Gold Layer Modeling

## Overview

This document explains the Gold-layer dimensional modeling design for HR integration analytics on Azure Databricks.

The Gold layer transforms cleaned and validated Silver data into analytics-ready dimensional models. It uses a constellation pattern where each integration has local facts and dimensions, while common worker and organization context is reused through Lakehouse Federation from the enterprise SQL Server HR DataHub.

The design supports:

- Operational analytics
- Recruiting pipeline analytics
- Training enrollment analytics
- Reusable reporting views
- Data quality checks
- BI consumption through Databricks SQL and Power BI

This is a sanitized portfolio artifact created to demonstrate lakehouse architecture, dimensional modeling, and analytics design thinking. It is not client documentation and does not include proprietary data, production code, credentials, internal system names, or real employee/customer data.

---

## Architecture Diagram

![Databricks Gold Layer Models](../diagrams/databricks-gold-layer-star-schemas.png)

---

## Gold Layer Design Pattern

The Gold layer is designed using star schema and constellation modeling principles.

Each integration is modeled as a focused constellation with:

- One or more business-process fact tables
- Local integration-specific dimensions
- Shared federated worker dimensions
- Reusable reporting views
- Shared DQ logging

The common modeling principle is:

> Facts stay local to the platform where data originates, while conformed worker and organization context is reused through federation.

---

## Gold Layer Object Summary

Representative Gold-layer objects include:

### Facts

- `fact_new_hire_event`
- `fact_downstream_notification`
- `fact_candidate_pipeline`
- `fact_requisition_snapshot`
- `fact_learner_enrollment`
- `fact_instructor_assignment`

### Local Dimensions

- `dim_event_type`
- `dim_downstream_service`
- `dim_notification_status`
- `dim_candidate_stage`
- `dim_recruiting_category`
- `dim_requisition`
- `dim_session`
- `dim_enrollment_status`

### Shared Federated Views

- `dim_worker_federated`
- `dim_worker_org_federated`

### Shared Utility

- `dq_check_log`

---

## Shared / Federated Dimension Bus

The Gold layer reuses enterprise conformed dimensions through Lakehouse Federation.

### Federated Worker Identity

`dim_worker_federated` provides reusable worker identity attributes such as:

- Worker surrogate key
- Worker identifier
- Worker name
- Work email
- Hire date
- Employee status
- Worker active flag

Used by:

- New Hire analytics
- Recruiting analytics
- Training analytics

### Federated Worker Organization Context

`dim_worker_org_federated` provides worker identity plus current organizational context such as:

- Line of Service
- Management level
- Company
- Location
- Department
- Job family

Used primarily by:

- Training coverage analytics
- Organization-based learning analysis

### Direct Federated Dimension Lookups

Some conformed dimensions are accessed directly during ETL-time surrogate key resolution.

Examples:

- Date
- Job
- Company
- Location
- Line of Service
- Department
- Action Reason

The resolved surrogate keys are stored in Gold fact tables so dashboards do not repeatedly query foreign dimensions at runtime.

---

## 1. New Hire Gold Model

### Purpose

The New Hire Gold model supports operational analytics for new hire event processing and downstream notification delivery.

It answers questions such as:

- How many new hire events were processed today?
- Which downstream services received notifications?
- Which notifications are overdue?
- What is the average processing time?
- Which new hires are unresolved due to worker dimension timing?

### Grain

#### `fact_new_hire_event`

One row per new hire event.

#### `fact_downstream_notification`

One row per worker / downstream service / event notification.

### Dimensions

Representative dimensions:

- `dim_worker_federated`
- `dim_event_type`
- `dim_downstream_service`
- `dim_notification_status`
- Direct federated dimensions for job, company, location, Line of Service, department, action reason, and date

### Measures

Representative measures:

- Event count
- Processing time
- Notifications triggered
- Days pending
- Delivery time
- Queue wait time
- Overdue indicator
- Worker resolved indicator

### Key Design Patterns

#### Worker Key Reconciliation

New hire records may arrive before the enterprise worker dimension is refreshed.

Pattern:

1. Fact row is loaded with temporary unresolved worker key.
2. Fallback worker identity attributes are stored for display.
3. Daily reconciliation updates the worker key after dimension refresh.
4. Records unresolved beyond threshold become DQ alerts.

#### Federated Dimension Lookups

New Hire performs multiple federated lookups during Gold fact loading to resolve enterprise conformed dimension keys.

This allows the fact table to retain enterprise context without copying conformed dimensions into Delta Lake.

#### Operational Dashboard Design

The model is consumed through Databricks SQL because the audience is small, operational, and needs near-real-time monitoring.

---

## 2. Recruiting Gold Model

### Purpose

The Recruiting Gold model supports recruiting pipeline analytics, requisition monitoring, conversion funnel analysis, and recruiter performance.

It answers questions such as:

- How many candidates are in each stage?
- Where are candidates dropping off?
- How long do candidates spend in each stage?
- Which requisitions are open, posted, filled, or aging?
- How are recruiters performing across pipeline velocity and outcomes?

### Grain

#### `fact_candidate_pipeline`

One row per candidate per pipeline stage/status change.

#### `fact_requisition_snapshot`

One row per requisition per extraction snapshot.

### Dimensions

Representative dimensions:

- `dim_worker_federated` for recruiter identity
- `dim_candidate_stage`
- `dim_recruiting_category`
- `dim_requisition`
- Date dimension

### Measures

Representative measures:

- Candidate count
- Days in stage
- Days in pipeline
- Stage sequence number
- Conversion rate
- Drop-off count
- Open requisition count
- Posted requisition count
- Filled requisition count

### Key Design Patterns

#### Step Dimension

`dim_candidate_stage` is modeled as a step dimension.

It defines:

- Stage order
- Stage group
- Terminal stage indicator
- Positive outcome indicator
- Movement / pipeline stage classification

This enables funnel analysis and progression tracking.

#### Window Functions

The pipeline fact uses window-function-based measures such as:

- Previous stage timestamp
- Days in stage
- First stage timestamp
- Days in pipeline
- Stage sequence number
- Stage progression indicator

These allow pipeline movement analysis without requiring a separate accumulating snapshot fact.

#### Candidate vs Worker Modeling

Recruiting candidates are external applicants and are not expected to exist in the enterprise worker dimension.

Therefore:

- Candidate identity is stored in recruiting-specific structures.
- Recruiter identity resolves to the federated worker dimension.
- Candidate-to-worker reconciliation is intentionally not required.

#### Power BI Consumption

Recruiting analytics is consumed through Power BI because the audience is broader and business-facing.

The model supports:

- Pipeline overview
- Conversion funnel
- Education analysis
- Requisition health
- Recruiter performance
- Row-level security by recruiter or leadership role

---

## 3. Training Gold Model

### Purpose

The Training Gold model supports learning enrollment, session utilization, instructor workload, and training coverage analytics.

It answers questions such as:

- Who is enrolled in training?
- What is withdrawal rate by session or organization?
- Which sessions are underutilized or over capacity?
- What is instructor workload?
- What is training coverage by Line of Service, level, location, or department?

### Grain

#### `fact_learner_enrollment`

One row per worker / session / enrollment event.

#### `fact_instructor_assignment`

One row per instructor / session / assignment status change.

### Dimensions

Representative dimensions:

- `dim_worker_federated`
- `dim_worker_org_federated`
- `dim_session`
- `dim_enrollment_status`
- Date dimension

### Measures

Representative measures:

- Enrollment count
- Registered count
- Withdrawal count
- Withdrawal rate
- Session capacity
- Session utilization
- Instructor assignment count
- Active assignment indicator
- Training coverage count

### Key Design Patterns

#### Worker Role-Playing

The worker dimension is role-played in Training analytics.

Examples:

- Learner worker
- Instructor worker

Both roles resolve to the same enterprise worker dimension but represent different business meanings in different facts.

#### Dual-Key Session Lookup

The session dimension supports different lookup keys depending on the source fact.

Examples:

- Learner enrollment may resolve session using one session identifier.
- Instructor assignment may resolve session using another session identifier.

Both resolve to the same `dim_session` to provide consistent session analytics.

#### Worker Organization Federation

Training coverage uses worker organization context from the enterprise HR DataHub.

This enables analysis such as:

- Coverage by Line of Service
- Coverage by level
- Coverage by location
- Coverage by company
- Coverage by department

#### DQ Alert Pattern

Learners and instructors are expected to be existing workers.

Therefore:

- Unresolved worker keys indicate source data quality issues.
- No automatic reconciliation is applied.
- DQ alerts are generated for investigation.

---

## Shared Data Quality Utility

The Gold layer includes a shared DQ logging utility.

### Purpose

The DQ utility provides a common mechanism for capturing data quality results across integrations.

Representative checks include:

- Unresolved worker keys
- Unresolved recruiter keys
- Unresolved session keys
- Unmapped stage values
- Volume anomalies
- SLA / overdue notifications
- Completeness checks
- Business rule threshold checks

### Result Pattern

Each DQ result captures:

- Integration name
- Check name
- Check category
- Severity
- Status
- Failed record count
- Run timestamp
- Details for investigation

### Value

The shared DQ utility enables:

- Cross-integration monitoring
- Trend analysis
- Production support visibility
- Consistent error classification
- DQ scorecard development

---

## Local vs Federated Dimensions

The Gold model separates dimensions into two categories.

### Local Dimensions

Local dimensions are integration-specific and live in Delta Lake.

Examples:

- Event type
- Notification status
- Downstream service
- Candidate stage
- Recruiting category
- Requisition
- Session
- Enrollment status

These dimensions are close to the facts they support and are maintained by the Databricks pipelines.

### Federated Dimensions

Federated dimensions are enterprise conformed dimensions managed in SQL Server and accessed through Lakehouse Federation.

Examples:

- Worker
- Date
- Job
- Company
- Location
- Line of Service
- Department
- Action Reason

These remain source-of-truth objects in the enterprise warehouse and are reused by Databricks.

---

## Reporting Views

Gold reporting views sit on top of facts and dimensions.

Purpose:

- Pre-join facts and dimensions
- Hide surrogate keys
- Expose business-friendly fields
- Apply reusable calculations
- Prepare dashboard-ready datasets
- Support BI tools with simplified query surfaces

Representative reporting view groups:

### New Hire Views

- New hire event analysis
- Downstream notification analysis
- Daily summary
- SLA summary
- Overdue notification detail

### Recruiting Views

- Candidate pipeline analysis
- Requisition analysis
- Pipeline daily summary
- Conversion rates
- Requisition summary
- Candidate education analysis
- Recruiter performance

### Training Views

- Learner enrollment analysis
- Enrollment daily summary
- Session utilization
- Instructor workload
- Training coverage

---

## Design Principles

1. Each integration is modeled as a focused Gold-layer constellation.
2. Facts remain local to where the data originates.
3. Local dimensions are created only when the subject area requires integration-specific context.
4. Enterprise conformed dimensions are reused through Lakehouse Federation.
5. Worker identity and organization context are not duplicated unnecessarily.
6. Gold facts store resolved surrogate keys where practical.
7. Reporting views simplify BI consumption.
8. DQ checks are part of the Gold model, not an afterthought.
9. BI tool choice depends on audience, latency, and governance needs.
10. Dimensional modeling is applied where it creates analytical value.

---

## Confidentiality Note

This document is a sanitized portfolio artifact created for data architecture demonstration purposes. It does not include proprietary client documentation, confidential data, production code, credentials, internal system names, or real employee/customer data.
