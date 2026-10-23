# Architecture Principles

## Overview

This document summarizes the key architecture principles behind the Databricks Lakehouse HR Analytics design.

## Principles

1. Conformed dimensions remain in the enterprise SQL Server DataHub.
2. Databricks accesses conformed dimensions through Lakehouse Federation instead of persistent replication.
3. Integration-specific dimensions live locally in Delta Lake.
4. Facts stay local to where the data originates.
5. Bronze preserves raw source data for audit and reconstruction.
6. Silver standardizes, validates, and enriches data.
7. Gold provides dimensional models and reporting-ready views.
8. Data quality checks are written to a shared Gold utility table.
9. Reconciliation is used only when unresolved keys are expected due to timing.
10. DQ alerts are used when unresolved keys indicate source data issues.
11. BI tool selection depends on audience, latency, RLS, and visual complexity.
12. Power BI is used for broad enterprise analytics; Databricks SQL is used for operational monitoring.
13. Materialization is an explicit design choice, not the default.
14. Security and governance are designed into each layer.
15. Dimensional modeling is applied where it creates analytical value.

---

## Confidentiality Note

This document is a sanitized portfolio artifact created for architecture demonstration purposes. It does not include proprietary client documentation, confidential data, production code, credentials, internal system names, or real employee/customer data.