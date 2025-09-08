
Below is a **Sprint‑1 work‑breakdown** that turns the “big‑picture” goal *“collect market, news and social data → Kafka → TimescaleDB (Coinbase exchange)”* into **tiny, test‑able increments**.  
Each bullet is a **stand‑alone task** that can be completed in a few hours, has a **clear success criterion**, and comes with **unit‑test ideas** (or a ready‑to‑run test file) so you can verify the code automatically in CI.

---

## Sprint‑1 Overview

| Phase | Goal | Primary Tech |
|-------|------|--------------|
| **1️⃣ Data‑Ingestion** | Pull live data from Coinbase, CryptoPanic RSS, and Twitter; push raw JSON to Kafka. | `python‑asyncio`, `websockets`, `aiohttp`, `kafka‑python`, `tweepy‑async` |
| **2️⃣ Persistence** | Consume Kafka topics, normalise the payloads, and write them into TimescaleDB (PostgreSQL). | `psycopg2‑binary` / `asyncpg`, `SQLAlchemy`, `TimescaleDB` |
| **3️⃣ Test Harness** | Unit‑test each component, add integration smoke‑tests, and wire everything into a Docker‑Compose dev stack. | `pytest`, `pytest‑asyncio`, `testcontainers` |

The **minimum viable product (MVP)** at the end of Sprint‑1 is a **running Docker‑Compose** that:

1. Streams **5‑minute candles** from Coinbase Pro (or Coinbase Advanced Trade) into a Kafka topic `price_candle_5m`.
2. Pulls the latest **CryptoPanic headlines** (RSS) into a Kafka topic `news_raw`.
3. Streams **filtered Twitter tweets** (crypto‑related keywords) into a Kafka topic `tweets_raw`.
4. A **consumer** reads each topic, validates the schema, and inserts a row into TimescaleDB tables `price_candle`, `news_item`, `tweet`.

---

## Detailed Task List

### 1️⃣ Set‑up the Development Environment (Day 0)

| # | Sub‑task | Done‑criteria | Unit‑test |
|---|----------|---------------|-----------|
| 1.1 | Create a **Git repo** with a `pyproject.toml` (or `requirements.txt`) that pins the required libraries. | `pip install -r requirements.txt` works on a fresh venv. | `test_requirements_importable.py` – try importing each library. |
| 1.2 | Add a **Docker‑Compose** file that brings up: <br>• `zookeeper` + `kafka` (confluent‑platform) <br>• `timescaledb` (postgres:15‑timescaledb) <br>• `redis` (optional cache) <br>• A **network** `crypto_bot`. | `docker compose up -d` finishes without errors; `docker compose ps` shows all containers `healthy`. | `test_docker_compose_up.sh` – script that runs `docker compose up -d` and checks container health via `docker inspect`. |
| 1.3 | Add **CI pipeline** skeleton (GitHub Actions) that runs `pytest`. | Workflow triggers on push and passes. | N/A (CI config). |

---

### 2️⃣ Coinbase Market‑Data Ingestion

| # | Sub‑task | Done‑criteria | Unit‑test |
|---|----------|---------------|-----------|
| 2.1 | Write an **async WebSocket client** that connects to Coinbase Pro **`/products/<symbol>/candles`** (5‑minute interval). | Client receives at least one candle JSON dict. | `test_coinbase_ws_connection.py` – mock the WS endpoint with `websockets.testing` and assert the first message matches schema. |
| 2.2 | Normalise the raw message into a **canonical dict**: <br>`{symbol, open_time, close_time, open, high, low, close, volume}`. | Function `normalise_candle(raw_msg)` returns dict with correct types. | `test_normalise_candle.py` – feed a sample raw array `[timestamp, low, high, open, close, volume]` and compare. |
| 2.3 | Create a **Kafka producer** wrapper (`KafkaProducerWrapper`) that serialises dict → JSON bytes and publishes to topic `price_candle_5m`. | `producer.send()` called without exception; message appears in Kafka (use `kafka-console-consumer` to verify). | `test_kafka_producer.py` – spin up a **testcontainers‑kafka** container, send a dict, consume it, assert equality. |
| 2.4 | Wire the WS client → normaliser → producer in an **async task** (`coinbase_ingest.py`). | Running `python -m coinbase_ingest` prints “candle sent” every 5 min (or faster in sandbox mode). | `test_ingest_pipeline.py` – patch the WS client to emit a pre‑canned candle, patch the producer to capture the payload, assert the pipeline flow. |
| 2.5 | Add **graceful shutdown** (SIGINT/SIGTERM) that closes the WS and Kafka connections. | `Ctrl‑C` exits cleanly, no “Connection reset by peer” errors in logs. | `test_graceful_shutdown.py` – spawn the ingest coroutine, send a `SIGINT` via `asyncio.get_running_loop().stop()`, verify `close()` called. |

---

### 3️⃣ CryptoPanic RSS Ingestion

| # | Sub‑task | Done‑criteria | Unit‑test |
|---|----------|---------------|-----------|
| 3.1 | Write a **poller** (`crypto_panic_poller.py`) that GETs the RSS feed (`https://cryptopanic.com/api/v1/posts/?auth_token=…`). | Returns a list of dicts with at least `title`, `published_at`, `url`, `source`. | `test_crypto_panic_fetch.py` – mock `aiohttp.ClientSession.get` to return a static XML/JSON payload, assert parsing. |
| 3.2 | Normalise each entry into a canonical dict: `{title, published_at, url, source, sentiment}` (sentiment can be `neutral` if not present). | Function `normalise_news(item)` returns correct keys and types. | `test_normalise_news.py` – feed a sample entry, compare. |
| 3.3 | Publish each normalised news item to Kafka topic `news_raw`. | Consumer can read the same JSON payload. | `test_news_kafka_producer.py` – same pattern as 2.3, using testcontainers‑kafka. |
| 3.4 | Schedule the poller to run **every 30 seconds** (or configurable). | Log shows “Fetched 7 new items” repeatedly. | `test_polling_interval.py` – use `freezegun` or `asyncio.sleep` patch to fast‑forward time, verify `fetch` called the expected number of times. |
| 3.5 | Add **duplicate‑filter** (Redis set or in‑memory LRU) to avoid re‑publishing the same article ID. | No duplicate messages appear in Kafka after a restart. | `test_duplicate_filter.py` – feed same ID twice, assert only one `send` call. |

---

### 4️⃣ Twitter Filtered Stream Ingestion

| # | Sub‑task | Done‑criteria | Unit‑test |
|---|----------|---------------|-----------|
| 4.1 | Register a **Twitter Developer v2** filtered stream rule set (e.g., `crypto OR bitcoin OR eth`). | Rule appears in the Twitter dashboard; API returns `200 OK`. | `test_twitter_rule_creation.py` – mock the rule‑creation endpoint, assert payload. |
| 4.2 | Write an **async streaming client** (`twitter_ingest.py`) using `tweepy‑async` or raw `aiohttp` to consume the filtered stream. | Prints tweet JSON for each matching tweet. | `test_twitter_stream_mock.py` – mock the HTTP stream with a few newline‑delimited JSON tweets, assert they are yielded. |
| 4.3 | Normalise tweet into `{tweet_id, created_at, text, author_id, lang, retweet_count, like_count}`. | Function `normalise_tweet(raw)` returns dict with correct types. | `test_normalise_tweet.py`. |
| 4.4 | Publish to Kafka topic `tweets_raw`. | Consumer can read the same dict. | `test_tweet_kafka_producer.py`. |
| 4.5 | Implement **reconnect logic** with exponential back‑off (max 5 retries). | After a simulated network drop, client reconnects and continues streaming. | `test_twitter_reconnect.py` – mock a connection error, verify back‑off schedule and eventual success. |
| 4.6 | Add **rate‑limit handling** (inspect `x-rate-limit-remaining` header). | When limit is reached, client sleeps until reset. | `test_rate_limit_handling.py`. |

---

### 5️⃣ Kafka → TimescaleDB Consumers

| # | Sub‑task | Done‑criteria | Unit‑test |
|---|----------|---------------|-----------|
| 5.1 | Define **SQL schema** (TimescaleDB hypertables): <br>• `price_candle (symbol TEXT, open_time TIMESTAMPTZ, close_time TIMESTAMPTZ, open NUMERIC, high NUMERIC, low NUMERIC, close NUMERIC, volume NUMERIC)` <br>• `news_item (title TEXT, published_at TIMESTAMPTZ, url TEXT, source TEXT, sentiment TEXT)` <br>• `tweet (tweet_id TEXT PRIMARY KEY, created_at TIMESTAMPTZ, text TEXT, author_id TEXT, lang TEXT, retweet_cnt INT, like_cnt INT)`. | `psql` shows tables, `SELECT * FROM price_candle LIMIT 1` works. | `test_timescale_schema.sql` – run via `pytest-postgresql` fixture, assert tables exist. |
| 5.2 | Write a **generic async consumer** (`kafka_to_ts.py`) that: <br>1. Subscribes to a topic. <br>2. Deserialises JSON. <br>3. Calls a **handler** function specific to the topic. | Consumer runs and logs “processed 1 record”. | `test_generic_consumer.py` – use testcontainers‑kafka, produce a test message, assert handler called with correct dict. |
| 5.3 | Implement **handler for price candles** that inserts into `price_candle` hypertable using `asyncpg`. | After sending a test candle, a row appears in the DB. | `test_price_handler.py` – spin up a testcontainers‑postgres (Timescale extension), run handler, query row. |
| 5.4 | Implement **handler for news items** (same pattern). | Row appears with correct `sentiment`. | `test_news_handler.py`. |
| 5.5 | Implement **handler for tweets** (same pattern). | Row appears, primary‑key conflict is handled (ignore duplicate). | `test_tweet_handler_duplicate.py`. |
| 5.6 | Add **batching** (optional for sprint 1, but nice to have): accumulate up to 500 rows or 2 seconds, then bulk‑INSERT. | Log shows “batch of 23 inserted”. | `test_batch_insert.py` – feed 10 messages quickly, assert a single `INSERT` statement executed. |
| 5.7 | Add **error handling & dead‑letter queue**: on DB error, publish the original payload to Kafka topic `dlq_<original>`. | Faulty message ends up in `dlq_price_candle`. | `test_dead_letter.py`. |

---

### 6️⃣ Integration Smoke Tests (End‑to‑End)

| # | Test | Description |
|---|------|-------------|
| 6.1 | **Kafka‑to‑DB round‑trip** | Start the full Docker‑Compose stack, run the three producers (Coinbase, CryptoPanic, Twitter) for 1 minute, then query TimescaleDB to ensure at least one row per table. |
| 6.2 | **Graceful shutdown** | Send `SIGTERM` to the Docker‑Compose project, verify all containers stop cleanly and no partial rows are left. |
| 6.3 | **Idempotency** | Restart the producers; ensure no duplicate rows appear (thanks to primary‑key constraints and duplicate‑filter). |

All smoke tests can be scripted with `pytest` + `testcontainers` and run in CI after the unit tests.

---

## Example Unit‑Test Files (Skeleton)

Below are a few concrete test snippets you can copy‑paste into your `tests/` folder. They use **pytest** and **testcontainers** (Docker‑based test fixtures) – install with:

```bash
pip install pytest pytest-asyncio testcontainers[postgres,kafka] aiohttp websockets
```

### `tests/test_coinbase_ws_connection.py`

```python
import asyncio
import json
import pytest
import websockets
from unittest import mock

# The function we want to test (simplified)
from coinbase_ingest import normalise_candle, CoinbaseWSClient

@pytest.fixture
def sample_raw():
    # Coinbase sends an array: [time, low, high, open, close, volume]
    return [1696502400, 26000.0, 26200.0, 26100.0, 26150.0, 123.45]

def test_normalise_candle(sample_raw):
    out = normalise_candle(sample_raw, symbol="BTC-USD")
    assert out["symbol"] == "BTC-USD"
    assert out["open_time"] == 1696502400
    assert out["open"] == 26100.0
    assert out["volume"] == 123.45
    # ensure types
    for k in ("open", "high", "low", "close", "volume"):
        assert isinstance(out[k], float)

@pytest.mark.asyncio
async def test_ws_client_receives_message(monkeypatch):
    # Mock the websocket server
    async def fake_ws(*_, **__):
        class Dummy:
            async def __aenter__(self): return self
            async def __aexit__(self, *exc): pass
            async def recv(self):
                # Return a JSON string that the client expects
                return json.dumps({
                    "type": "match",
                    "product_id": "BTC-USD",
                    "price": "26150.00",
                    "size": "0.01",
                    "time": "2023-10-05T12:00:00.000Z"
                })
        return Dummy()
    monkeypatch.setattr(websockets, "connect", fake_ws)

    client = CoinbaseWSClient(symbol="BTC-USD")
    # we only need the first message, so we stop after one iteration
    async for msg in client.stream():
        assert msg["product_id"] == "BTC-USD"
        break
```

### `tests/test_kafka_producer.py`

```python
import json
import pytest
from testcontainers.kafka import KafkaContainer
from kafka import KafkaProducer, KafkaConsumer

@pytest.fixture(scope="module")
def kafka():
    with KafkaContainer() as kafka:
        kafka.start()
        yield kafka

def test_produce_and_consume(kafka):
    producer = KafkaProducer(
        bootstrap_servers=kafka.get_bootstrap_server(),
        value_serializer=lambda v: json.dumps(v).encode("utf-8")
    )
    consumer = KafkaConsumer(
        "test_topic",
        bootstrap_servers=kafka.get_bootstrap_server(),
        auto_offset_reset="earliest",
        enable_auto_commit=True,
        value_deserializer=lambda v: json.loads(v.decode("utf-8")),
        consumer_timeout_ms=5000,
    )
    payload = {"foo": "bar", "num": 42}
    producer.send("test_topic", payload)
    producer.flush()

    msgs = list(consumer)
    assert len(msgs) == 1
    assert msgs[0].value == payload
```

### `tests/test_timescale_insert.py`

```python
import asyncio
import asyncpg
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="module")
def pg():
    with PostgresContainer("timescale/timescaledb:2.12.0-pg15") as pg:
        pg.start()
        # Enable Timescale extension
        conn = pg.get_connection()
        conn.exec("CREATE EXTENSION IF NOT EXISTS timescaledb;")
        conn.close()
        yield pg

@pytest.mark.asyncio
async def test_price_insert(pg):
    conn = await asyncpg.connect(dsn=pg.get_connection_url())
    await conn.execute("""
        CREATE TABLE IF NOT EXISTS price_candle (
            symbol TEXT,
            open_time TIMESTAMPTZ,
            close_time TIMESTAMPTZ,
            open NUMERIC,
            high NUMERIC,
            low NUMERIC,
            close NUMERIC,
            volume NUMERIC
        );
        SELECT create_hypertable('price_candle', 'open_time', if_not_exists => TRUE);
    """)

    # Insert a single row
    await conn.execute("""
        INSERT INTO price_candle (symbol, open_time, close_time, open, high, low, close, volume)
        VALUES ($1,$2,$3,$4,$5,$6,$7,$8);
    """, "BTC-USD", "2023-10-05T12:00:00Z", "2023-10-05T12:05:00Z",
       26100.0, 26200.0, 26000.0, 26150.0, 123.45)

    row = await conn.fetchrow("SELECT * FROM price_candle LIMIT 1;")
    assert row["symbol"] == "BTC-USD"
    assert float(row["open"]) == 26100.0
    await conn.close()
```

---

## Sprint‑1 Checklist (What “Done” Looks Like)

| ✅ | Milestone |
|----|-----------|
| 1 | Docker‑Compose dev stack (`kafka`, `timescaledb`, `redis`) runs healthy. |
| 2 | Coinbase WS client streams 5‑min candles → Kafka topic `price_candle_5m`. |
| 3 | CryptoPanic RSS poller pushes normalized news to `news_raw`. |
| 4 | Twitter filtered stream pushes normalized tweets to `tweets_raw`. |
| 5 | Three **Kafka consumers** write to TimescaleDB tables (`price_candle`, `news_item`, `tweet`). |
| 6 | Unit tests for every module pass (`pytest -q` → 100%). |
| 7 | Integration smoke test verifies at least one row per table after a 1‑minute run. |
| 8 | CI pipeline runs on every push and reports success. |
| 9 | Documentation (`README.md`) explains how to start the stack, run the producers, and view the DB. |
|10| Kill‑switch script (`./stop.sh`) cleanly tears down the stack. |

Once the checklist is green, you have a **robust data‑pipeline foundation** on which the later sprints (feature engineering, LLM reasoning, back‑testing, risk management) can be built safely.

---

### Next Steps (After Sprint‑1)

1. **Feature Store** – materialise technical indicators on the TimescaleDB side (continuous aggregates).  
2. **LLM Retrieval‑Augmented Reasoner** – build the RAG pipeline that pulls the latest news/tweets for a symbol and calls GPT‑4‑Turbo (or a self‑hosted Llama‑2).  
3. **Meta‑Learner** – train the logistic‑regression / XGBoost fusion model.  
4. **Execution Guard & Risk Engine** – position sizing, stop‑loss/take‑profit, draw‑down monitoring.  
5. **Back‑testing Framework** – replay the raw data and the LLM decisions to evaluate performance.

Happy coding! 🚀
