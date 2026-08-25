# Architecture

## High-Level Design

The system is a cloud-native lakehouse built on AWS, following the medallion architecture pattern (Bronze → Silver → Gold). Data flows from external market APIs through streaming or batch ingestion, lands in S3 as raw Bronze data, gets refined through Silver and Gold transformation layers, and is served to analytics and operational consumers.

```
External APIs (Alpaca, CoinGecko, Binance, yfinance)
          │
          ├─── Streaming path ──► Kinesis Data Streams ──► Kinesis Firehose ──► S3 Bronze
          │
          └─── Batch path ──────► AWS Glue / Spark ──────────────────────────► S3 Bronze
                                                                                    │
                                              ┌─────────────────────────────────────┘
                                              ▼
                                    Spark / Glue (cleanse, validate, dedupe)
                                              │
                                              ▼
                                         S3 Silver (Iceberg)
                                              │
                                    dbt (model, aggregate, features)
                                              │
                                              ▼
                                         S3 Gold (Iceberg)
                                              │
                          ┌───────────────────┴───────────────────┐
                          ▼                                       ▼
                 Amazon Athena                          RDS PostgreSQL
               (analytics / BI)                  (signals, alerts, operational)
```

---

## Layer Definitions

### Bronze (Raw)
- **Location:** `s3://lakehouse-{env}/bronze/{source}/{asset_type}/year=YYYY/month=MM/day=DD/`
- **Format:** JSON (streaming), Parquet (batch)
- **Rules:** Immutable. Never overwrite. Append-only. Exact copy of source data.
- **Schema:** Inferred at read time via Glue Data Catalog.

### Silver (Conformed)
- **Location:** `s3://lakehouse-{env}/silver/{domain}/{table}/`
- **Format:** Apache Iceberg (Parquet + metadata layer)
- **Rules:** Typed, validated, deduplicated. No business logic. Conforms to defined schemas.
- **Schema:** Enforced. Breaking changes require a new table version.
- **Key transformations:** timestamp normalization (UTC), null handling, type casting, deduplication on event ID.

### Gold (Curated)
- **Location:** `s3://lakehouse-{env}/gold/{domain}/{table}/`
- **Format:** Apache Iceberg
- **Rules:** Business logic applied. Aggregated or feature-engineered. Optimized for consumption.
- **Consumers:** Amazon Athena (analytics), RDS (operational), notebooks (research).

---

## AWS Services

| Service | Role |
|---|---|
| S3 | Primary storage for all lakehouse layers |
| AWS Glue | Data Catalog, Spark job execution (ETL), crawlers |
| Kinesis Data Streams | Real-time ingestion buffer |
| Kinesis Firehose | Delivery stream from Kinesis → S3 Bronze |
| AWS Lambda | Lightweight event-driven transforms, alerting |
| Amazon Athena | OLAP serving layer — queries Iceberg tables on S3 via Glue Data Catalog |
| RDS (PostgreSQL) | OLTP operational store for signals, positions, alerts |
| MWAA (Managed Airflow) | Batch orchestration (or Step Functions as lighter alternative) |
| Step Functions | Lightweight orchestration alternative to Airflow |
| IAM | Least-privilege role-based access between services |
| CloudWatch | Logging, metrics, alerting for pipeline health |
| Secrets Manager | API keys, DB credentials |
| Terraform | Infrastructure as code for all AWS resources |
| GitHub Actions | CI/CD for infra and pipeline deployments |

---

## Data Modeling

### Athena (Analytics — Star Schema via Glue Data Catalog)

**Fact tables:**
- `fact_trades` — individual trade events (symbol, price, volume, timestamp)
- `fact_candles` — OHLCV bars (1m, 5m, 1h, 1d) per symbol
- `fact_quotes` — bid/ask snapshots

**Dimension tables:**
- `dim_symbol` — ticker, name, exchange, asset class, sector
- `dim_exchange` — exchange name, region, timezone
- `dim_date` — date, day of week, week, month, quarter, year, is_market_day
- `dim_time` — time of day at minute granularity

### RDS PostgreSQL (Operational)

**Tables:**
- `signals` — generated trading signals (symbol, strategy, direction, confidence, timestamp)
- `positions` — open/closed paper or live positions
- `alerts` — price or indicator threshold alerts
- `backtest_runs` — backtest metadata and results summary
- `strategy_config` — strategy parameters and versioning

---

## Ingestion Patterns

### Streaming (Real-Time)
- Alpaca WebSocket → Python producer → Kinesis Data Streams
- Kinesis Firehose buffers and writes to S3 Bronze (JSON, partitioned by date)
- Latency target: < 30 seconds Bronze landing

### Batch (Historical / Scheduled)
- Airflow DAG triggers Glue Spark job nightly
- Glue job pulls prior day OHLCV from Alpaca REST API or yfinance
- Writes Parquet to S3 Bronze, partitioned by date
- Subsequent Spark job promotes Bronze → Silver

---

## Data Quality

- JSON Schema contracts defined per source in `contracts/`
- Great Expectations suites run at Silver promotion step
- Glue Data Catalog tracks schema versions
- Failed records routed to `s3://lakehouse-{env}/quarantine/` for inspection

---

## Environments

| Env | Purpose |
|---|---|
| `dev` | Local development, small data samples |
| `staging` | Full pipeline, non-production AWS account or prefix |
| `prod` | Live data, monitored, cost-controlled |

Terraform workspaces or variable files manage environment separation.

---

## Security

- All S3 buckets: private, versioned, encrypted at rest (SSE-S3 or SSE-KMS)
- IAM roles: least privilege, no wildcard `*` actions in prod
- API keys stored in AWS Secrets Manager, never in code or env files
- No PII in scope — market data only
