# ADR 002: Amazon Athena as Analytics Serving Layer

**Date:** 2026-08-24  
**Status:** Accepted

## Context

The Gold layer needs an OLAP query engine for analytics over Iceberg tables on S3. The two primary candidates were Amazon Athena and Amazon Redshift Serverless.

## Decision

Use **Amazon Athena** backed by the Glue Data Catalog.

## Rationale

- **Cost model:** Athena charges $5/TB scanned with zero idle cost. At the query volume of a portfolio or small trading project, this is effectively free. Redshift Serverless has a minimum base capacity charge regardless of utilization.
- **No data movement:** Athena queries Iceberg tables directly on S3. Redshift Serverless would require Redshift Spectrum (external tables with added complexity) or a separate load job, undermining the lakehouse architecture.
- **Operational simplicity:** No cluster, no connections to manage, no VPC configuration needed in early development stages.
- **Native Glue integration:** Athena uses the Glue Data Catalog natively — the same catalog used by Glue ETL jobs, ensuring schema consistency across the pipeline.

## Alternatives Considered

- **Redshift Serverless:** Better choice at scale with concurrent BI users, complex analytical workloads requiring sub-second latency, or materialized views with auto-refresh. Documented as the production scaling path if this project outgrows Athena.

## Consequences

- Analytics queries are written in Athena SQL (Trino/Presto dialect).
- Gold layer dbt models target the Athena/Glue catalog.
- Query results land in a dedicated S3 results bucket.
- Cost monitoring via CloudWatch to catch unexpectedly large scans.
