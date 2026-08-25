# ADR 001: Apache Iceberg as Open Table Format

**Date:** 2025-08-24  
**Status:** Accepted

## Context

The Silver and Gold layers require an open table format to support ACID transactions, schema evolution, time travel, and efficient query performance on S3. The two leading candidates were Apache Iceberg and Delta Lake.

## Decision

Use **Apache Iceberg** for both Silver and Gold layers.

## Rationale

- **Athena compatibility:** Athena has native first-class support for Iceberg with no extra connector required. Delta Lake on Athena requires a third-party connector that lags behind the Delta spec.
- **AWS ecosystem alignment:** AWS Glue 3.0+ has built-in Iceberg support. AWS has clearly positioned Iceberg as its preferred open table format (see Lake Formation, EMR, and Athena native support).
- **Partition evolution:** Iceberg allows changing the partitioning strategy without rewriting existing data — important as data volumes grow and query patterns change.
- **Schema evolution:** Iceberg's schema evolution is explicit and safe (add, rename, drop columns without full rewrites).
- **Row-level operations:** Native support for row-level deletes and updates, useful for correcting bad market data records post-ingestion.

## Alternatives Considered

- **Delta Lake:** Strong choice, especially with Databricks. Rejected because this project is AWS-native without Databricks, and Athena + Delta is a second-class experience.

## Consequences

- Glue jobs must use the Iceberg connector (bundled with Glue 3.0+, no extra JAR required).
- dbt must use the `dbt-athena` adapter with Iceberg table configuration.
- Athena queries use standard SQL — no format-specific syntax required.
