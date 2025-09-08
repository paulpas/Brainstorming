
## Sprint 2 – From Raw Data to Actionable Signals  

Sprint 2 picks up where Sprint 1 left off: we now have **clean, time‑aligned data** (candles, news, tweets) flowing into TimescaleDB.  
The goal of this sprint is to **turn that data into quantitative features, enrich it with LLM‑derived insights, and build a lightweight decision engine** that can be back‑tested and later deployed in real‑time.

At the end of Sprint 2 you should have:

| ✅ | Deliverable |
|----|-------------|
| 1 | **Technical‑indicator materialisation** in TimescaleDB (continuous aggregates). |
| 2 | **Sentiment & embedding pipelines** that turn news/tweets into numeric vectors stored alongside candles. |
| 3 | **Retrieval‑augmented LLM reasoner** (prompt + RAG) that returns a bias + confidence JSON for a given symbol/time. |
| 4 | **Hybrid decision model** (quantitative + LLM) that outputs a trade‑signal probability. |
| 5 | **Back‑testing harness** that re‑plays the whole pipeline on historical data and produces performance metrics. |
| 6 | **Full unit‑test & integration‑test suite** for every new component, plus CI coverage > 80 %. |
| 7 | Updated documentation and a “run‑all‑in‑Docker” script for the end‑to‑end pipeline. |

Below is a **task‑by‑task breakdown** (≈ 2 weeks of work for a small team). Each task lists a concrete “Done” criterion and suggested unit‑test ideas.

---

## 1️⃣ Materialise Technical Indicators (TimescaleDB Continuous Aggregates)

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 1.1 | Write SQL functions that compute EMA, RSI, MACD, Bollinger Bands on the `price_candle` hypertable (using `timescaledb_experimental` or PL/pgSQL). | `SELECT * FROM ema_5m('BTC-USD', now() - interval '1 day') LIMIT 1` returns a numeric value. | `test_indicator_sql.py` – spin up a testcontainers‑Postgres, load a few candle rows, run the function, compare against a reference `pandas_ta` calculation. |
| 1.2 | Create **continuous aggregates** (materialised views) for each indicator with a 5‑minute refresh policy. | `\d+ ema_5m_view` shows a hypertable with `refresh_lag = '5 minutes'`. | `test_continuous_aggregate_refresh.py` – insert a new candle, wait 6 min (or fast‑forward time with `pg_sleep`), assert the view reflects the new value. |
| 1.3 | Add **indexing** on `(symbol, time_bucket)` for fast look‑ups. | `EXPLAIN ANALYZE SELECT * FROM ema_5m_view WHERE symbol='BTC-USD' AND time > now() - interval '1 hour'` shows an index scan. | `test_indicator_index.py` – verify query plan contains `Bitmap Index Scan`. |
| 1.4 | Write a **Python wrapper** (`indicator_store.py`) that fetches the latest indicator row for a symbol and returns a dict. | `get_latest_indicators('BTC-USD')` returns `{'ema':…, 'rsi':…, 'macd_hist':…}`. | `test_indicator_wrapper.py` – mock `asyncpg` connection, assert dict keys/types. |

---

## 2️⃣ Sentiment & Embedding Pipelines

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 2.1 | **Text normalisation** utility (`clean_text.py`): strip URLs, emojis, markdown, lower‑case, keep only ASCII letters. | `clean_text("🚀 BTC up! https://t.co/xyz")` → `"btc up"` | `test_clean_text.py`. |
| 2.2 | **Embedding generation** – integrate `sentence‑transformers` (`all‑MiniLM‑L6‑v2`) to turn each news/tweet into a 384‑dim vector. Store the vector as a `BYTEA` column (`embedding`) in `news_item` / `tweet`. | After ingesting a sample tweet, the DB row contains a non‑null `embedding` of length 384. | `test_embedding_generation.py` – mock a tweet, run `embed_text()`, assert shape and dtype. |
| 2.3 | **Sentiment scoring** – fine‑tune a small DistilBERT classifier on a crypto‑sentiment dataset (or use a pre‑trained model). Store a float `sentiment_score` ∈ [-1, 1]. | `SELECT sentiment_score FROM news_item WHERE id=…` returns a value between -1 and 1. | `test_sentiment_classifier.py` – feed a known‑positive headline, assert score > 0.5. |
| 2.4 | **Temporal aggregation** – create a materialised view `news_sentiment_5m(symbol, bucket_start)` that computes `AVG(sentiment_score)` and `COUNT(*)` per 5‑minute bucket. | Query returns one row per bucket with `avg_sentiment` and `cnt`. | `test_sentiment_aggregate.py`. |
| 2.5 | **Embedding clustering** – nightly job runs K‑means (k≈10) on the day’s embeddings, stores cluster centroids in a table `embedding_cluster`. Each news/tweet gets a `cluster_id`. | After the job, every row in `news_item` has a non‑null `cluster_id`. | `test_embedding_clustering.py` – run on a tiny synthetic dataset, verify cluster IDs are assigned. |

---

## 3️⃣ Retrieval‑Augmented LLM Reasoner (RAG)

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 3.1 | **Prompt template** – store a Jinja2 template (see Sprint 1 description) that expects a JSON blob and asks for bias + confidence. | Template renders without errors given a sample dict. | `test_prompt_render.py`. |
| 3.2 | **RAG service** (`llm_reasoner.py`) that: <br>1. Pulls the latest candle + indicators (from 1.4). <br>2. Pulls the latest sentiment aggregates (from 2.4). <br>3. Pulls the top‑N recent news/tweets (by `published_at`). <br>4. Builds the JSON payload, calls the LLM, parses the JSON response. | `await get_reason('BTC-USD')` returns `{'bias':'UP','confidence':0.81,'rationale':…}`. | `test_llm_reasoner_mock.py` – mock the OpenAI/Anthropic API to return a fixed JSON, assert parsing. |
| 3.3 | **Caching layer** – store the LLM output in Redis with a TTL of 30 s keyed by `symbol:timestamp`. Subsequent calls within the TTL return the cached result. | Two consecutive calls within 10 s hit Redis (log shows “cache hit”). | `test_llm_cache.py` – use a fake Redis client, assert `get` called before `api`. |
| 3.4 | **Fallback path** – if the LLM call fails (timeout, rate‑limit, or malformed JSON), return `{'bias':'NEUTRAL','confidence':0.0}` and log the incident. | Simulated API error leads to neutral output, no exception bubbles up. | `test_llm_fallback.py`. |
| 3.5 | **Rate‑limit handling** – read `x‑rate‑limit‑remaining` header; if < 5, back‑off exponentially and retry up to 3 times. | When the mock header reports 0, the client sleeps and retries, then succeeds. | `test_llm_rate_limit.py`. |

---

## 4️⃣ Hybrid Decision Engine (Quant + LLM)

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 4.1 | **Quantitative model** – train a Gradient‑Boosted Tree (XGBoost) on the technical indicators + sentiment aggregates to predict a binary “price‑up‑in‑Δt” label. Save the model as `quant_model.pkl`. | `predict_quant(['BTC-USD', …])` returns a probability `p_q ∈ [0,1]`. | `test_quant_model.py` – load a pre‑saved model, feed a synthetic feature vector, assert output shape. |
| 4.2 | **Meta‑learner** – logistic regression (or shallow MLP) that takes three inputs: `p_q`, `llm_confidence`, and one‑hot encoded `bias` (UP/DOWN/NEUTRAL). Train on the last 30 days of back‑tested data. Save as `meta_model.pkl`. | `predict_final(features)` returns `p_f`. | `test_meta_learner.py` – mock inputs, assert probability range and that bias encoding works. |
| 4.3 | **Decision rule** – implement a function `signal(p_f, bias, confidence)` that returns `BUY`, `SELL`, or `HOLD` based on thresholds (e.g., `p_f > 0.65 → BUY`). | Unit test verifies each branch with boundary values. | `test_signal_logic.py`. |
| 4.4 | **API endpoint** (`/signal/<symbol>`) that returns the full JSON: `{bias, confidence, p_quant, p_final, decision, rationale}`. | `curl http://localhost:8000/signal/BTC-USD` returns a 200 JSON with all fields. | `test_signal_endpoint.py` – spin up FastAPI test client, mock model predictions, assert response schema. |
| 4.5 | **Model‑version endpoint** (`/model-info`) that reports timestamps and git SHA for reproducibility. | Returns JSON with `quant_model_version`, `meta_model_version`. | `test_model_info.py`. |

---

## 5️⃣ Back‑Testing Harness

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 5.1 | **Historical data loader** – a script that pulls candle + indicator + sentiment data for a given date range from TimescaleDB and builds a Pandas DataFrame aligned on 5‑minute bars. | `load_history('2023‑01‑01','2023‑01‑31')` returns a DataFrame with no NaNs (except at the very start). | `test_history_loader.py`. |
| 5.2 | **LLM replay** – batch‑process the historical timestamps through the RAG service (use the cached‑mode to avoid real API calls). Store the LLM outputs in a temporary table `llm_replay`. | After running, `SELECT COUNT(*) FROM llm_replay` equals the number of bars. | `test_llm_replay.py` – mock the LLM call to return deterministic outputs, assert row count. |
| 5.3 | **Signal generation** – run the hybrid decision engine on each bar, produce a column `signal` (BUY/SELL/HOLD). | DataFrame now has a `signal` column with only the three allowed values. | `test_signal_generation.py`. |
| 5.4 | **Position‑sizing & P&L calculator** – simple risk model (1 % of equity per trade, stop‑loss = 1 × ATR, TP = 2 × ATR). Compute equity curve. | Plot (`matplotlib`) shows a non‑flat equity curve; final return > 0% on a test dataset. | `test_pnl_calculator.py` – feed a tiny synthetic series where a BUY is always profitable, assert final equity = initial * (1+gain). |
| 5.5 | **Performance metrics** – compute CAGR, Sharpe, Sortino, max‑drawdown, win‑rate, profit‑factor. Export a JSON report (`backtest_report.json`). | JSON contains all required keys with numeric values. | `test_backtest_report.py`. |
| 5.6 | **Visualization notebook** – Jupyter notebook that loads `backtest_report.json` and plots equity curve, draw‑down, and signal heatmap. | Notebook runs end‑to‑end without manual edits. | Not a unit test, but include a sanity‑check script that runs `nbconvert --execute`. |

---

## 6️⃣ Testing, CI & Documentation

| # | Sub‑task | Done‑criterion |
|---|----------|----------------|
| 6.1 | **Add pytest‑asyncio** to the CI matrix for all async components. |
| 6.2 | **Coverage** – enforce `coverage run -m pytest && coverage report -m` ≥ 80 % (exclude integration‑only scripts). |
| 6.3 | **Integration test** (`tests/integration/test_end_to_end.py`) that spins up Docker‑Compose, runs the full pipeline for a 10‑minute window, then checks that the back‑test report file exists and contains a non‑null `cagr`. |
| 6.4 | **Documentation** – update `README.md` with sections: <br>• “Run the full pipeline (Docker)”, <br>• “How to retrain the models”, <br>• “Interpret the back‑test report”. |
| 6.5 | **Run‑all script** (`run_all.sh`) that: <br>1. `docker compose up -d` <br>2. Wait for DB & Kafka health <br>3. Launch the ingestion services (Coinbase, News, Twitter) <br>4. Launch the RAG + decision service <br>5. Execute the back‑test script <br>6. Print a summary table. |
| 6.6 | **Kill‑switch** (`stop_all.sh`) that gracefully stops all containers and removes temporary volumes. |

---

## 7️⃣ Timeline (2 Weeks)

| Day | Focus |
|-----|-------|
| **Day 1‑2** | Indicator SQL functions + continuous aggregates (Task 1). |
| **Day 3‑4** | Sentiment & embedding pipelines (Task 2). |
| **Day 5** | RAG prompt & LLM wrapper (Task 3.1‑3.3). |
| **Day 6** | LLM fallback, caching, rate‑limit handling (Task 3.4‑3.5). |
| **Day 7** | Quantitative model training & meta‑learner (Task 4). |
| **Day 8** | API endpoints for signal & model info (Task 4.4‑4.5). |
| **Day 9** | Back‑testing data loader & LLM replay (Task 5.1‑5.3). |
| **Day 10** | P&L calculator, performance metrics (Task 5.4‑5.5). |
| **Day 11** | Unit‑test implementation for all new modules. |
| **Day 12** | Integration test, CI pipeline update, coverage check. |
| **Day 13** | Documentation, `run_all.sh` / `stop_all.sh`. |
| **Day 14** | Buffer / bug‑fix day, sprint demo preparation. |

---

## 8️⃣ Sprint‑2 Demo Checklist

- **Live dashboard** (Grafana) showing the latest technical indicators and sentiment aggregates.  
- **REST call** `GET /signal/BTC-USD` returning a fresh LLM‑augmented decision.  
- **Back‑test report** (JSON + plotted equity curve) for the last 30 days, with key metrics displayed.  
- **All unit & integration tests** passing in CI.  

Once the demo is approved, Sprint 3 can focus on **risk‑management, order‑execution guard, live‑trading loop, and advanced reinforcement‑learning / multi‑LLM ensembles**.

Good luck, and happy coding! 🚀
