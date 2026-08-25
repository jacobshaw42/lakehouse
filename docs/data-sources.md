# Data Sources & Ingestion Strategy

## Overview

This project ingests financial market data from multiple sources covering US equities and cryptocurrency. Each source serves a distinct role: real-time streaming, historical backfill, or enrichment.

---

## Sources

### 1. Alpaca Markets API

**URL:** https://alpaca.markets  
**Asset class:** US Equities (stocks, ETFs)  
**Access:** Free paper trading account; market data requires Unlimited plan (~$9/mo) for real-time, or free for 15-min delayed  
**Auth:** API key + secret (stored in AWS Secrets Manager)

**Endpoints used:**

| Endpoint | Type | Data |
|---|---|---|
| `/v2/stocks/{symbol}/trades` | REST | Historical trades |
| `/v2/stocks/{symbol}/bars` | REST | OHLCV bars (1m, 5m, 1d, etc.) |
| `/v2/stocks/{symbol}/quotes` | REST | Historical bid/ask |
| `wss://stream.data.alpaca.markets/v2/iex` | WebSocket | Real-time trades + quotes |

**Ingestion pattern:**
- Real-time: Python WebSocket consumer → Kinesis Data Streams
- Historical: Airflow-triggered Glue job → Alpaca REST → S3 Bronze

**Rate limits:** 200 requests/min (free tier)

**Symbols to start with:**
- Large-cap equities: AAPL, MSFT, NVDA, AMZN, GOOGL, TSLA, SPY, QQQ
- Expand to a broader watchlist as pipeline stabilizes

---

### 2. CoinGecko API

**URL:** https://www.coingecko.com/en/api  
**Asset class:** Cryptocurrency  
**Access:** Free tier available (50 calls/min); Pro plan for higher limits  
**Auth:** Optional API key for higher rate limits

**Endpoints used:**

| Endpoint | Type | Data |
|---|---|---|
| `/coins/{id}/market_chart` | REST | Historical price, volume, market cap |
| `/coins/{id}/ohlc` | REST | OHLCV (limited granularity on free tier) |
| `/simple/price` | REST | Current spot price (polling) |

**Ingestion pattern:**
- Batch polling every 5 minutes via Lambda or Airflow → S3 Bronze
- No native WebSocket; simulate near-real-time via scheduled polling

**Coins to start with:** BTC, ETH, SOL, BNB

**Limitations:** Free tier has limited OHLC granularity (daily only for historical); minute-level requires Pro.

---

### 3. Binance API

**URL:** https://binance-docs.github.io/apidocs  
**Asset class:** Cryptocurrency  
**Access:** Public (no auth required for market data)  
**Auth:** API key needed only for trading; not required for data ingestion

**Endpoints used:**

| Endpoint | Type | Data |
|---|---|---|
| `GET /api/v3/klines` | REST | OHLCV candlestick data |
| `GET /api/v3/trades` | REST | Recent trades |
| `wss://stream.binance.com:9443/ws/{symbol}@trade` | WebSocket | Real-time trade stream |
| `wss://stream.binance.com:9443/ws/{symbol}@kline_{interval}` | WebSocket | Real-time candles |

**Ingestion pattern:**
- Real-time: Python WebSocket consumer → Kinesis Data Streams (same stream as equities, differentiated by `asset_class` field)
- Historical: Glue batch job → Binance REST klines → S3 Bronze

**Pairs to start with:** BTCUSDT, ETHUSDT, SOLUSDT, BNBUSDT

**Advantages over CoinGecko:** More granular OHLCV (1m resolution), WebSocket support, no auth required.

---

### 4. Yahoo Finance (yfinance)

**URL:** https://pypi.org/project/yfinance/  
**Asset class:** US Equities, ETFs, Indices, Crypto (limited)  
**Access:** Free, unofficial API (scraping-based — treat as fragile)  
**Auth:** None

**Usage:** Backfill and enrichment only. Not a primary source.

**Use cases:**
- Historical OHLCV backfill when Alpaca history is insufficient
- Fundamental data (P/E, EPS, market cap) for dimension enrichment
- Index data (S&P 500 constituent list)

**Ingestion pattern:**
- On-demand Glue job or notebook → yfinance → S3 Bronze (clearly tagged as `source=yfinance`)

**Limitations:** No SLA, can break without notice. Never use as primary source for production signals.

---

## Ingestion Architecture

```
Alpaca WebSocket  ──┐
Binance WebSocket ──┼──► Python Producer (EC2 / ECS) ──► Kinesis Data Streams ──► Firehose ──► S3 Bronze
                    │
Alpaca REST  ───────┐
Binance REST ───────┼──► AWS Glue Spark Job (scheduled) ──────────────────────────────────► S3 Bronze
CoinGecko REST ─────┘
yfinance ───────────┘
```

---

## Bronze Schema Conventions

All Bronze records include these envelope fields regardless of source:

| Field | Type | Description |
|---|---|---|
| `_source` | string | Source identifier: `alpaca`, `binance`, `coingecko`, `yfinance` |
| `_ingested_at` | timestamp | UTC timestamp when record was written to Bronze |
| `_ingestion_type` | string | `streaming` or `batch` |
| `_raw` | string | Original JSON payload (for streaming records) |

The rest of the schema is source-specific and schema-on-read at the Bronze layer.

---

## Partitioning Strategy

Bronze S3 paths follow Hive-style partitioning:

```
s3://lakehouse-{env}/bronze/
  └── {source}/
      └── {asset_class}/
          └── {data_type}/
              └── year=YYYY/month=MM/day=DD/
                  └── {timestamp}_{uuid}.json|parquet
```

Examples:
```
bronze/alpaca/equities/trades/year=2025/month=08/day=24/
bronze/binance/crypto/klines/year=2025/month=08/day=24/
bronze/coingecko/crypto/prices/year=2025/month=08/day=24/
```

---

## API Key Management

All API keys and secrets are stored in **AWS Secrets Manager** under the path:

```
lakehouse/{env}/{source}/api_credentials
```

Examples:
- `lakehouse/dev/alpaca/api_credentials`
- `lakehouse/prod/binance/api_credentials`

Python ingestion code retrieves credentials at runtime via `boto3.client('secretsmanager')`. Keys are never hardcoded or committed to the repository.

---

## Incremental Load Strategy

| Source | Strategy | Checkpoint mechanism |
|---|---|---|
| Alpaca REST | Page by `start`/`end` timestamp | Last ingested timestamp stored in RDS `ingestion_checkpoints` table |
| Binance REST | Page by `startTime`/`endTime` (kline) | Same RDS checkpoint table |
| CoinGecko REST | `from`/`to` Unix timestamp | Same RDS checkpoint table |
| yfinance | `start`/`end` date params | Manual or backfill only |

---

## Cost Considerations

- Kinesis Data Streams: ~$0.015/shard-hour + $0.014/million records — keep shard count minimal in dev
- Firehose: $0.029/GB delivered — near zero cost at this data volume
- Glue: $0.44/DPU-hour — optimize job bookmarks, avoid full scans
- S3: $0.023/GB/month — compress Parquet, use lifecycle rules to Glacier for Bronze older than 90 days
- Free tier API sources (Alpaca free, Binance public, CoinGecko free) minimize external data costs
