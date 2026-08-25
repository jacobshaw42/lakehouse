# Lakehouse

A portfolio-grade data engineering project showcasing an end-to-end lakehouse architecture built on AWS. Uses real-time stock market and cryptocurrency data as the domain — with a secondary goal of supporting algorithmic trading strategies.

This project demonstrates production-level thinking across the full data stack: ingestion, storage, transformation, modeling, and serving.

---

## Goals

- **Portfolio showcase** — demonstrate breadth and depth across data engineering and architecture
- **Real data, real problems** — use live market data to drive realistic engineering decisions
- **AWS-native** — leverage managed services (Kinesis, Glue, S3, Athena, RDS, etc.) end to end
- **Extensible** — architecture that can grow from a portfolio project into a real algorithmic trading system

---

## Tech Stack

| Layer | Technology |
|---|---|
| Cloud | AWS |
| Storage | S3 + Apache Iceberg (Bronze / Silver / Gold) |
| Streaming | AWS Kinesis Data Streams, Kinesis Firehose |
| Batch ingestion | AWS Glue, Apache Spark |
| Data catalog | AWS Glue Data Catalog |
| Transformation | dbt (Silver → Gold), Spark (Bronze → Silver) |
| Data modeling | Star schema / Data Vault 2.0 |
| Serving / OLAP | Amazon Athena + Glue Data Catalog |
| Relational (OLTP) | Amazon RDS (PostgreSQL) |
| Orchestration | Apache Airflow (MWAA) or Step Functions |
| Infrastructure | Terraform |
| CI/CD | GitHub Actions |
| Contracts | JSON Schema / Great Expectations |
| Language | Python, SQL |

---

## Architecture Overview

```
Data Sources (APIs)
    │
    ├── Real-time tick / quote data  ──►  Kinesis Data Streams  ──►  Kinesis Firehose  ──►  S3 Bronze
    │
    └── Historical OHLCV data        ──►  Glue / Spark batch job  ──►  S3 Bronze

S3 Bronze (raw, immutable)
    └──►  Spark / Glue (cleanse, validate, dedupe)  ──►  S3 Silver (Iceberg)

S3 Silver (conformed, typed)
    └──►  dbt (model, aggregate, feature engineering)  ──►  S3 Gold (Iceberg)

S3 Gold (curated)
    ├──►  Athena + Glue Data Catalog (analytics / BI)
    └──►  RDS PostgreSQL (trading signals, alerts, operational data)
```

---

## Data Sources

- **Alpaca Markets API** — real-time and historical US equities (free tier available)
- **CoinGecko / Binance API** — cryptocurrency price data
- **Yahoo Finance (yfinance)** — historical OHLCV backup / enrichment

See [`docs/data-sources.md`](docs/data-sources.md) for full ingestion strategy.

---

## Repository Structure

```
lakehouse/
├── docs/                   # Architecture docs, ADRs, data source strategy
│   ├── architecture.md
│   ├── data-sources.md
│   └── decisions/          # Architecture Decision Records (ADRs)
│       ├── 001-table-format.md
│       └── 002-analytics-serving.md
├── infra/                  # Terraform — AWS infrastructure as code
├── ingestion/              # Streaming and batch ingestion pipelines
├── transforms/             # Spark jobs (Bronze → Silver)
├── dbt/                    # dbt models (Silver → Gold)
├── orchestration/          # Airflow DAGs or Step Functions definitions
├── contracts/              # Data contract schemas (JSON Schema / GX)
├── serving/                # Athena views, RDS schema, query layer
├── notebooks/              # EDA, strategy research, signal development
├── tests/                  # Unit and integration tests
└── README.md
```

---

## Medallion Architecture

| Layer | Location | Description |
|---|---|---|
| Bronze | `s3://lakehouse-{env}/bronze/` | Raw, immutable data exactly as received |
| Silver | `s3://lakehouse-{env}/silver/` | Cleansed, typed, deduplicated Iceberg tables |
| Gold | `s3://lakehouse-{env}/gold/` | Aggregated, modeled, feature-engineered tables |

---

## Data Modeling

- **Equities / Crypto** — fact tables for trades, quotes, and candles; dimension tables for symbols, exchanges, and time
- **Trading signals** — operational model in RDS PostgreSQL for signal generation, position tracking, and alert state
- **Analytics** — star schema in Athena (via Glue Data Catalog) for historical performance analysis

---

## Algorithmic Trading (Secondary Goal)

The Gold layer and RDS schema are designed to support a lightweight algorithmic trading workflow:

- Feature engineering on OHLCV data (moving averages, RSI, MACD, Bollinger Bands)
- Signal generation stored in RDS
- Backtesting via notebook layer
- Live execution via Alpaca broker API (paper trading first)

> **Note:** This project is not financial advice. Paper trading before any live capital.

---

## Status

🚧 Active development — building out infrastructure and ingestion layer first.

---

## License

MIT
