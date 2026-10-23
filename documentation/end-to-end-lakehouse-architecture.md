# End-to-End Lakehouse Architecture

## Overview

This document explains the end-to-end Databricks Lakehouse architecture for HR integration analytics.

The design demonstrates how operational HR data and file-based feeds can be captured, standardized, modeled, governed, and consumed through Azure Databricks using Medallion Architecture, Delta Lake, Lakehouse Federation, Gold-layer dimensional models, semantic reporting views, data quality monitoring, reconciliation patterns, and BI consumption.

The architecture follows this flow:

> Operational Sources → Bronze → Silver → Gold → Lakehouse Federation → Reporting Views → BI Consumption → Analytics Value

This is a sanitized portfolio artifact created to demonstrate lakehouse architecture, data engineering, dimensional modeling, governance, and analytics consumption design. It is not client documentation and does not include proprietary data, production code, credentials, internal system names, or real employee/customer data.

---

## Architecture Diagram

[View End-to-End Lakehouse Architecture Diagram](../diagrams/databricks-lakehouse-end-to-end-architecture-detailed.png)

---

## Architecture Scope

This architecture is Databricks-centered.

It focuses on the Lakehouse analytics layer and shows how Databricks:

- Ingests operational and file-based HR integration data
- Preserves raw data in Bronze
- Cleans and validates data in Silver
- Builds Gold-layer dimensional models
- Reuses enterprise conformed dimensions through Lakehouse Federation
- Exposes Gold reporting views for BI consumption
- Supports Power BI and Databricks SQL dashboards
- Applies data quality, reconciliation, governance, and operational controls

The enterprise SQL Server HR DataHub appears in this architecture only as an external source of conformed dimensions accessed through Lakehouse Federation.

---

## Layer 1: Source / Intermediate Layer

The Source / Intermediate Layer contains operational stores and file-based feeds that provide data to the Databricks Lakehouse.

Representative source patterns include:

- NRT SQL databases
- Operational event tables
- File feeds from managed file transfer processes
- CSV capture from cloud storage
- JDBC incremental extraction
- JDBC full overwrite for small reference data
- Watermark-based processing
- File-level deduplication

The platform supports three integration analytics domains:

1. New Hire Operational Analytics
2. Recruiting Pipeline Analytics
3. Training Enrollment Analytics

---

## New Hire Sources

New Hire data is sourced from operational event and reference tables.

Representative source tables include:

- New hire event records
- Worker reference records
- Downstream notification status records
- Downstream service configuration records

Common characteristics:

- Event-driven source pattern
- Operational status monitoring
- JDBC extraction
- Incremental processing for events
- Full overwrite for small reference/configuration tables
- Low capture urgency because source operational tables persist

New Hire data supports near-real-time operational monitoring and downstream notification SLA analysis.

---

## Recruiting Sources

Recruiting data is sourced from a combination of operational records and file-based enrichment feeds.

Representative source data includes:

- Candidate pipeline records
- Requisition records
- Candidate education enrichment files
- Recruiter information

Common characteristics:

- JDBC extraction from operational tables
- CSV/file capture for enrichment data
- File-level deduplication
- Capture before file cleanup or retention expiration
- Medium/high capture urgency due to transient file availability

Recruiting data supports candidate pipeline, conversion, requisition health, education, and recruiter performance analytics.

---

## Training Sources

Training data is sourced from learning and event-management processes.

Representative source data includes:

- Training session files
- Learner enrollment files
- Instructor assignment records
- Session mapping records

Common characteristics:

- File capture from managed file transfer or cloud storage
- JDBC extraction from operational lookup tables
- Incremental processing for enrollment and instructor changes
- Full overwrite for small session mapping or reference structures
- Critical capture urgency where files are transformed, encrypted, or delivered downstream after processing

Training data supports enrollment, withdrawal, session utilization, instructor workload, and training coverage analytics.

---

## Layer 2: Bronze Layer — Raw Capture

The Bronze layer preserves source data as received.

### Purpose

The Bronze layer provides:

- Raw source preservation
- Audit trail
- Replay capability
- File capture history
- Source reconstruction support
- Foundation for Silver and Gold rebuilds

### Common Bronze Patterns

- Raw append for event or transactional sources
- Full overwrite for small reference tables
- File-level capture for CSV feeds
- Audit metadata columns such as load timestamp, source identifier, and file name
- Watermark tracking for incremental loads
- Durable retention for recovery and reprocessing

### Integration Examples

#### New Hire Bronze

Captures raw operational event, worker, notification, and configuration records.

#### Recruiting Bronze

Captures raw candidate pipeline records, requisition files, and candidate enrichment files.

#### Training Bronze

Captures raw session files, learner enrollment files, instructor records, and session mapping records.

---

## Layer 3: Silver Layer — Cleaned / Validated / Enriched

The Silver layer standardizes, validates, and enriches Bronze data.

### Purpose

The Silver layer acts as a data quality and standardization boundary.

It prepares data for Gold-layer dimensional modeling.

### Common Silver Patterns

- Type casting
- Timestamp standardization
- Deduplication
- Required-field validation
- Business classification derivation
- Status normalization
- Boolean flag derivation
- File and source metadata standardization
- Enrichment from secondary sources
- Removal of fields not suitable for analytics consumption

### Integration Examples

#### New Hire Silver

Representative transformations:

- Event payload parsing
- Event type derivation
- Worker identifier standardization
- Notification cleanup
- Error flag derivation
- Type casting and deduplication

#### Recruiting Silver

Representative transformations:

- Candidate status classification
- Stage name derivation
- Stage order derivation
- Terminal / positive outcome flag derivation
- Requisition cleanup
- Candidate education enrichment from file-based sources

#### Training Silver

Representative transformations:

- Session data cleanup
- Enrollment status normalization
- Instructor data standardization
- Timestamp and identifier cleanup
- Empty or unavailable source fields removed from analytics flow
- Active/cancelled flags derived from session status

---

## Layer 4: Gold Layer — Dimensional Models

The Gold layer provides analytics-ready dimensional models.

### Purpose

Gold transforms Silver data into facts, dimensions, measures, and reporting-ready structures.

The design uses star schema and constellation modeling patterns.

### Common Gold Patterns

- Local integration-specific dimensions
- Local fact tables
- Federated conformed dimensions
- Surrogate key resolution
- Fact grain definition
- Business measures
- Reconciliation logic where appropriate
- DQ checks
- Reporting view preparation

---

## New Hire Gold Model

### Purpose

Supports new hire event processing, downstream notification SLA tracking, and operational health monitoring.

### Representative Facts

- New hire event fact
- Downstream notification fact

### Representative Dimensions

- Worker federated dimension
- Event type
- Downstream service
- Notification status
- Federated date / job / company / location / organization context

### Key Measures

- Event count
- Processing time
- Notification count
- Days pending
- Delivery time
- Overdue indicator
- Worker resolved indicator

### Key Design Pattern

New Hire uses a reconciliation pattern because a new worker may arrive in the event stream before the enterprise worker dimension refreshes.

Temporary unresolved worker keys are allowed, then resolved after the enterprise dimension refresh.

---

## Recruiting Gold Model

### Purpose

Supports candidate pipeline analytics, conversion analysis, requisition monitoring, and recruiter performance.

### Representative Facts

- Candidate pipeline fact
- Requisition snapshot fact

### Representative Dimensions

- Recruiter federated worker dimension
- Candidate stage
- Recruiting category
- Requisition

### Key Measures

- Candidate count
- Days in stage
- Days in pipeline
- Stage sequence
- Conversion rate
- Drop-off rate
- Open requisition count
- Filled requisition count

### Key Design Patterns

Recruiting uses:

- Step dimension for candidate stages
- Window functions for pipeline progression
- Requisition snapshot for point-in-time requisition health
- Candidate identity modeled locally because candidates are external applicants
- Recruiter identity resolved through federated worker dimension

---

## Training Gold Model

### Purpose

Supports training enrollment, withdrawal, session utilization, instructor workload, and training coverage analytics.

### Representative Facts

- Learner enrollment fact
- Instructor assignment fact

### Representative Dimensions

- Worker federated dimension
- Worker organization federated dimension
- Session dimension
- Enrollment status dimension

### Key Measures

- Enrollment count
- Registered count
- Withdrawal count
- Withdrawal rate
- Session utilization
- Instructor assignment count
- Training coverage

### Key Design Patterns

Training uses:

- Dual-key session lookup
- Worker dimension role-playing
- Learner and instructor worker roles
- Federated worker organization context
- DQ alerting instead of reconciliation when existing workers fail to resolve

---

## Layer 5: Lakehouse Federation

Lakehouse Federation allows Databricks to reuse conformed dimensions from the enterprise SQL Server HR DataHub without persistently replicating them into Delta Lake.

### Purpose

Federation provides:

- Zero-copy access to enterprise conformed dimensions
- Consistent worker and organization context
- Reduced data duplication
- Reduced synchronization overhead
- Centralized source-of-truth ownership

### Federated Dimensions

Representative conformed dimensions include:

- Worker
- Date
- Job
- Company
- Location
- Department
- Line of Service
- Action Reason

### Federated Views

Representative Databricks Gold-layer federated views include:

- Worker identity federated view
- Worker organization context federated view

### Design Principle

Conformed dimensions remain in the enterprise DataHub. Integration-specific facts and dimensions remain in Databricks.

This avoids unnecessary cross-platform replication while preserving enterprise context.

---

## Layer 6: Semantic / Reporting Views

The reporting layer exposes business-friendly SQL views on top of Gold facts and dimensions.

### Purpose

Reporting views simplify BI consumption.

They provide:

- Pre-joined fact and dimension datasets
- Business-friendly column names
- Reusable calculations
- KPI-ready structures
- Security-aware BI consumption
- Reduced complexity in Power BI and Databricks SQL dashboards

### View Groups

#### New Hire Views

Representative views include:

- New hire event analysis
- Downstream notification analysis
- New hire daily summary
- Notification SLA summary
- Overdue notification detail

#### Recruiting Views

Representative views include:

- Candidate pipeline analysis
- Pipeline daily summary
- Pipeline conversion rates
- Requisition summary
- Requisition analysis
- Candidate education analysis
- Recruiter performance

#### Training Views

Representative views include:

- Learner enrollment analysis
- Enrollment daily summary
- Session utilization
- Instructor workload
- Training coverage

---

## Layer 7: BI Consumption

The platform supports two main BI consumption patterns:

1. Databricks SQL Dashboard
2. Power BI Dashboards

### Databricks SQL Dashboard

Used for New Hire operational monitoring.

Best fit when:

- Audience is small and technical
- Users already work in Databricks
- Near-real-time monitoring is required
- Live SQL execution is acceptable
- No imported BI dataset is needed

Representative use cases:

- New hire event monitoring
- Downstream notification SLA
- Overdue notifications
- Processing time monitoring

### Power BI Dashboards

Used for Recruiting and Training analytics.

Best fit when:

- Audience is broader and business-facing
- Enterprise users are already familiar with Power BI
- Scheduled refresh is sufficient
- Advanced visuals and RLS are needed
- Cached datasets improve user concurrency and dashboard performance

Representative use cases:

- Recruiting pipeline analytics
- Conversion funnel analysis
- Requisition health
- Training enrollment
- Session utilization
- Instructor workload
- Training coverage

---

## Layer 8: Operational Controls

Operational controls span all layers of the lakehouse.

### Data Quality Monitoring

A shared Gold-layer DQ log captures integration-specific checks.

Representative checks:

- Unresolved worker keys
- Unresolved recruiter keys
- Unresolved session keys
- Unmapped candidate stages
- Volume checks
- SLA checks
- Withdrawal rate spikes

### Reconciliation

Reconciliation is applied only when unresolved keys are expected due to timing.

Example:

- New Hire uses daily reconciliation because new workers may not exist in the enterprise worker dimension immediately.

### DQ Alerts

DQ alerts are used where unresolved keys indicate source issues.

Examples:

- Training learner or instructor worker keys should resolve because they represent existing employees.
- If they do not resolve, a DQ alert is generated for investigation.

### Governance

Representative controls include:

- Unity Catalog
- Delta Lake time travel
- Table descriptions and tags
- Access control
- Lineage
- Environment-specific secrets
- Job monitoring
- Alerting
- CI/CD deployment patterns

---

## Layer 9: Analytics Value Delivered

The Databricks Lakehouse layer is designed not only to move and model data, but to unlock analytics that are difficult or impossible from raw operational systems alone.

By combining operational source data, Gold-layer dimensional models, federated worker/organization context, semantic reporting views, and BI consumption patterns, the platform enables analytics across New Hire, Recruiting, Training, and cross-domain workforce use cases.

---

### New Hire Analytics

New Hire analytics focuses on operational visibility, SLA monitoring, worker onboarding events, and downstream notification health.

Representative analytics enabled:

- Processing time trending by Line of Service, company, level, and worker type
- SLA compliance by downstream service
- Overdue notification tracking
- Hire event volume and trend analysis
- Hire reason categorization
- International assignment vs regular hire analysis
- Error analysis by organizational segment
- Historical event trend analysis
- New hire worker key reconciliation monitoring
- Cross-platform correlation between hire volume and headcount growth

Business value:

- Provides operational teams with near-real-time visibility into hire event processing
- Helps identify downstream notification delays
- Enables SLA monitoring and proactive issue resolution
- Connects onboarding events with enterprise worker context through federation

---

### Recruiting Analytics

Recruiting analytics focuses on candidate pipeline movement, conversion, requisition health, recruiter performance, and education-based insights.

Representative analytics enabled:

- Candidate pipeline conversion rates
- Review-to-interview, interview-to-offer, and offer-to-hire analysis
- Time-in-stage analysis
- Candidate drop-off analysis
- Pipeline bottleneck identification
- Evergreen-to-hiring movement tracking
- Offer acceptance rate analysis
- Requisition aging and stale requisition tracking
- Open, posted, filled, and closed requisition monitoring
- Education-based hiring analysis
- Recruiter performance and workload comparison
- Recruiting category and requisition health analysis

Business value:

- Provides recruiting teams with funnel visibility
- Helps identify where candidates drop off
- Enables recruiter and requisition performance monitoring
- Supports leadership visibility into hiring pipeline health

---

### Training Analytics

Training analytics focuses on enrollment, withdrawal, session utilization, instructor workload, and training coverage.

Representative analytics enabled:

- Training enrollment trends
- Enrollment by Line of Service, level, location, company, and department
- Withdrawal rate analysis
- Session capacity utilization
- Full, near-full, available, and empty session tracking
- Instructor workload distribution
- Sessions missing instructor assignments
- Training coverage by organization and management level
- Coverage gap heatmaps
- New vs returning enrollee analysis
- Enrollment momentum and month-over-month trends

Business value:

- Helps L&D teams identify training coverage gaps
- Supports session planning and capacity management
- Enables instructor workload visibility
- Provides HR leadership with organization-level training participation insights

Current limitation:

- The current design supports enrollment visibility.
- Score, pass/fail, completion, and certification analytics require future integration of completion data.

---

### Cross-Domain Analytics

Lakehouse Federation enables cross-domain analytics by linking Databricks integration facts with enterprise worker and organization context from the SQL Server HR DataHub.

Representative cross-domain analytics:

- New hire processing volume vs workforce headcount growth
- Training enrollment by organization, level, and location
- Recruiting pipeline health vs attrition replacement needs
- Training participation patterns for new hires
- Overdue onboarding notifications by organizational segment
- Recruiting demand compared with workforce movement and exits
- Training coverage compared with workforce demographics or role groups
- Future-state training completion impact on performance and attrition

Business value:

- Connects operational integration data with enterprise workforce context
- Enables analytics that are not possible from isolated source systems
- Reduces the need to replicate conformed dimensions
- Preserves a single source of truth for worker and organization context
- Supports cross-domain workforce decision-making

---

## Lakehouse Summary

The lakehouse design includes:

- Bronze raw capture tables
- Silver cleaned/validated tables
- Gold facts and dimensions
- Shared federated views
- Shared DQ utility
- Gold reporting views
- Databricks SQL operational dashboard
- Power BI analytics dashboards

The design demonstrates how a Databricks Lakehouse can extend an enterprise HR DataHub with integration-specific analytics while preserving conformed enterprise context.

---

## Key Architecture Principles

1. Bronze preserves raw source data for audit and reconstruction.
2. Silver standardizes, validates, and enriches data.
3. Gold provides dimensional models and analytics-ready facts.
4. Reporting views simplify BI consumption.
5. Lakehouse Federation avoids conformed dimension replication.
6. Facts stay local to where the data originates.
7. Integration-specific dimensions live locally in Delta Lake.
8. Enterprise conformed dimensions remain in SQL Server.
9. DQ checks and reconciliation are part of the architecture.
10. BI tools are selected based on audience, latency, RLS, and consumption needs.

---

## Related Artifacts

- [High-Level Architecture](../diagrams/databricks-lakehouse-high-level-architecture.png)
- [End-to-End Lakehouse Architecture](../diagrams/databricks-lakehouse-end-to-end-architecture-detailed.png)
- [Medallion Data Flow](../diagrams/medallion-data-flow-integration-vieww.png)
- [Lakehouse Federation Pattern](../diagrams/lakehouse-federation-pattern.png)
- [Gold Layer Models](../diagrams/databricks-gold-layer-star-schemas.png)
- [DQ & Reconciliation Pattern](../diagrams/dq-monitoring-reconciliation-pattern.png)
- [BI Consumption Pattern](../diagrams/bi-consumption-pattern.png)
- [Architecture Principles](architecture-principles.md)
---

## Confidentiality Note

This document is a sanitized portfolio artifact created for data architecture demonstration purposes. It does not include proprietary client documentation, confidential data, production code, credentials, internal system names, or real employee/customer data.