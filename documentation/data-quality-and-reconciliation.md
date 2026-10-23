# Data Quality and Reconciliation

## Overview

This document explains the data quality monitoring and reconciliation strategy used in the Databricks Lakehouse HR Analytics design.

The platform uses a shared Gold-layer DQ framework to monitor data quality across integration analytics domains such as New Hire, Recruiting, and Training. Each integration writes DQ results into a shared DQ log, but the resolution strategy differs based on the business meaning of the data.

The key principle is:

> The handling of unresolved worker keys depends on who the person is in the business process.

This is a sanitized portfolio artifact created to demonstrate data quality, reconciliation, production reliability, and lakehouse architecture thinking. It is not client documentation and does not include proprietary data, production code, credentials, internal system names, or real employee/customer data.

---

## Architecture Diagram

![DQ Monitoring and Reconciliation Pattern](../diagrams/dq-monitoring-reconciliation-pattern.png)

---

## Shared DQ Monitoring Framework

All Databricks integrations write data quality results to a shared Gold-layer DQ log.

### Shared DQ Log

```text
gold.dq_check_log
```

### Purpose

The shared DQ log provides a common structure for tracking quality checks across all integration analytics pipelines.

It supports:

- Cross-integration DQ monitoring
- Unknown-member tracking
- Unresolved key tracking
- SLA and overdue checks
- Volume checks
- Business rule checks
- Investigation history
- Future DQ dashboarding

### Representative Columns

```text
Check_ID
Integration
Table_Name
Check_Name
Check_Category
Severity
Status
Expected_Value
Actual_Value
Check_DateTime
Batch_ID
Details
```

### Status Values

| Status | Meaning |
|---|---|
| PASS | Check passed successfully |
| WARN | Potential issue requiring monitoring or investigation |
| FAIL | Data quality failure requiring investigation |

### Severity Levels

| Severity | Meaning |
|---|---|
| HIGH | Critical issue that may affect analytics correctness or SLA |
| MEDIUM | Important issue requiring monitoring or follow-up |
| LOW | Informational or lower-priority data quality signal |

### Important Note

In this design, Databricks integration DQ is alert-oriented and non-blocking.

`FAIL` or `WARN` statuses are logged for investigation and alerting, but the pipeline is not automatically blocked. This avoids unnecessary operational interruption for smaller integration analytics workloads while still preserving visibility and accountability.

---

## Why DQ Is Non-Blocking in Databricks

The enterprise SQL Server DataHub may use stronger gating rules for large-scale warehouse loads, where severe DQ issues can block downstream processing.

The Databricks integrations use a lighter-weight approach because:

- The integrations are smaller and more use-case-specific.
- Operational visibility is still needed even when some records have issues.
- Blocking every cycle could interrupt dashboards unnecessarily.
- DQ issues can often be investigated while the pipeline continues.
- Unresolved records are visible through DQ logs and operational dashboards.

This is a deliberate design choice.

---

## DQ Checks by Integration

## 1. New Hire DQ Checks

New Hire analytics monitors event processing, worker key resolution, and downstream notification health.

### Representative DQ Checks

| Check | Severity | Purpose |
|---|---|---|
| Worker_SK_Unresolved_48hrs | HIGH | Identifies new hire records still unresolved after the expected reconciliation window |
| Daily_Volume_Check | MEDIUM | Detects whether expected event volume is present in the last 24 hours |
| Overdue_Notifications | HIGH | Identifies downstream notifications breaching or approaching SLA |

### New Hire DQ Logic

New Hire records may temporarily have unresolved worker keys because a newly hired person may arrive in the event stream before the enterprise worker dimension is refreshed.

Therefore:

- `Worker_SK = -1` is expected temporarily.
- It should resolve after the enterprise worker dimension refreshes.
- Records unresolved beyond the threshold are treated as DQ issues.

---

## 2. Recruiting DQ Checks

Recruiting analytics monitors recruiter resolution, requisition resolution, candidate stage mapping, source volume, and education enrichment completeness.

### Representative DQ Checks

| Check | Severity | Purpose |
|---|---|---|
| Recruiter_SK_Unresolved | MEDIUM | Detects recruiter records that do not resolve to the enterprise worker dimension |
| Requisition_SK_Unresolved | HIGH | Detects fact rows where requisition dimension lookup failed |
| Stage_SK_Unmapped | HIGH | Detects new or unmapped candidate pipeline stages |
| Daily_Volume_Check | MEDIUM | Detects missing or paused recruiting pipeline activity |
| Education_Data_Completeness | LOW | Monitors candidate education enrichment coverage |

### Recruiting DQ Logic

Recruiting candidates are external applicants. They are not expected to exist in the enterprise worker dimension.

Therefore:

- Candidate identity is modeled inside recruiting facts/dimensions.
- Candidate-worker reconciliation is not required.
- Recruiter identity should resolve to the enterprise worker dimension.
- Unresolved recruiter keys are treated as DQ issues.

---

## 3. Training DQ Checks

Training analytics monitors learner and instructor worker resolution, session resolution, volume patterns, and withdrawal anomalies.

### Representative DQ Checks

| Check | Severity | Purpose |
|---|---|---|
| Worker_SK_Invalid | HIGH | Detects learner records that do not resolve to enterprise worker dimension |
| Instructor_SK_Invalid | HIGH | Detects instructor records that do not resolve to enterprise worker dimension |
| Session_SK_Unresolved | HIGH | Detects enrollment or instructor records that do not resolve to session dimension |
| Daily_Volume_Check | LOW | Tracks activity volume, often informational |
| Withdrawal_Rate_Spike | MEDIUM | Detects unusual withdrawal patterns that may indicate event cancellation or source issue |

### Training DQ Logic

Training learners and instructors are expected to be existing employees.

Therefore:

- `Worker_SK = -1` is not an expected timing issue.
- Unresolved workers indicate source data problems.
- DQ alerts are generated for manual investigation.
- Automatic reconciliation is not applied.

---

## Reconciliation Strategy

The reconciliation strategy differs by integration because the business meaning of an unresolved worker key differs by domain.

---

## 1. New Hire — Automated Reconciliation

### Business Context

New Hire records represent people joining the organization.

A new hire event may arrive before the enterprise worker dimension is updated by the nightly warehouse process.

### Why `Worker_SK = -1` Is Expected

At the time the event is captured:

- The new person exists in the event source.
- The new person may not yet exist in the enterprise worker dimension.
- Other reference dimensions such as job, company, location, department, and action reason may already resolve.
- Worker identity can still be displayed using fallback fields.

### Pattern

1. New hire event arrives.
2. Gold fact is loaded.
3. `Worker_SK` may be set to unresolved / unknown.
4. Fallback identity attributes are stored in the fact.
5. After the enterprise worker dimension refreshes, reconciliation runs.
6. The reconciliation job updates the fact with the resolved `Worker_SK`.
7. If still unresolved beyond the threshold, a DQ alert is raised.

### Representative Fallback Fields

- Worker ID
- First name
- Last name
- Work email

These allow the dashboard to remain useful while the surrogate key is temporarily unresolved.

### Reconciliation Job

Representative notebook:

```text
04_reconcile_worker_sk
```

Representative process:

```text
1. Find unresolved new hire fact rows.
2. Join unresolved Worker_IDs to federated worker dimension.
3. Retrieve resolved Worker_SK.
4. MERGE update fact_new_hire_event.
5. Mark Is_Worker_Resolved = true.
6. Log unresolved records after threshold to dq_check_log.
```

### Alert Threshold

Typical resolution expectation:

```text
Expected: within 24 hours
Investigate: unresolved beyond 48 hours
Critical: unresolved beyond 72 hours
```

---

## 2. Recruiting — No Candidate Reconciliation

### Business Context

Recruiting records represent external applicants and requisitions.

Candidates are not employees at the time of recruiting pipeline analysis.

### Why Candidate Reconciliation Is Not Needed

Candidates are not expected to exist in the enterprise worker dimension.

Therefore:

- Candidate identity remains in recruiting-specific facts/dimensions.
- Candidate attributes are modeled as recruiting-domain attributes.
- Candidate-to-worker reconciliation is intentionally avoided.

### What Does Resolve to Worker

Recruiter identity should resolve to the enterprise worker dimension because recruiters are employees.

If recruiter resolution fails:

- It may indicate an email mismatch.
- It may indicate missing or incorrect recruiter data.
- It is treated as a DQ issue.
- It is not automatically reconciled.

### Pattern

```text
Candidate identity → recruiting fact/dimension
Recruiter identity → federated worker dimension
Unresolved recruiter → DQ warning / investigation
```

---

## 3. Training — DQ Alert, Not Reconciliation

### Business Context

Training records represent learners and instructors.

Both learners and instructors are expected to be existing employees.

### Why `Worker_SK = -1` Is Unexpected

Unlike New Hire, Training does not represent a person before they enter the enterprise worker dimension.

If a learner or instructor does not resolve:

- The Worker_ID may be incorrect.
- The source system may contain a bad custom field.
- The person may not exist in the enterprise worker dimension as expected.
- The source data must be investigated.

### Pattern

```text
Training fact load
    ↓
Worker_SK / Instructor_Worker_SK lookup
    ↓
If unresolved:
    - Log DQ FAIL
    - Show unresolved identifiers for investigation
    - Alert operations team
    - Fix source data manually
```

### Why Not Auto-Fix

Automatic reconciliation is not used because it could mask source data quality issues.

The correct resolution is to fix the source record or mapping.

---

## Comparison Matrix

| Topic | New Hire | Recruiting | Training |
|---|---|---|---|
| Person type | New person joining organization | External candidate | Existing employee |
| Expected in enterprise worker dimension? | Yes, after dimension refresh | No, not as candidate | Yes, already |
| Unresolved Worker_SK meaning | Expected timing lag | Not applicable for candidate | Data quality issue |
| Resolution method | Automated reconciliation | No candidate reconciliation | DQ alert and manual source correction |
| Auto-fix? | Yes, daily MERGE | No | No |
| Fallback identity | Name/email in fact | Candidate identity in recruiting fact | Worker_ID retained for investigation |
| Alert threshold | Unresolved beyond 48 hours | Recruiter unresolved | Any unresolved learner/instructor |
| DQ behavior | Alert after threshold | Alert recruiter/stage/req issues | Alert source data issues |

---

## Core Principle

The handling of unresolved worker keys is determined by business context:

```text
New person joining organization
    → Worker_SK lag is expected
    → Automated reconciliation

External applicant
    → Not an employee yet
    → No worker reconciliation

Existing employee
    → Should already exist
    → DQ alert and source correction
```

This prevents incorrect automation and ensures that data quality issues are handled appropriately.

---

## Design Principles

1. DQ checks are shared through a common Gold-layer log.
2. DQ rules are integration-specific because business context differs.
3. Unknown member handling does not mean the same thing in every domain.
4. New Hire unresolved keys are timing issues until they exceed threshold.
5. Recruiting candidates are external and should not be reconciled to workers.
6. Training unresolved workers indicate source data quality issues.
7. Reconciliation should fix timing gaps, not mask bad source data.
8. DQ alerting should preserve operational visibility.
9. DQ history enables trend analysis and recurring issue detection.
10. Data quality is part of platform architecture, not an afterthought.

---

## Confidentiality Note

This document is a sanitized portfolio artifact created for data architecture demonstration purposes. It does not include proprietary client documentation, confidential data, production code, credentials, internal system names, or real employee/customer data.