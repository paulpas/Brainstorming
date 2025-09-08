
## Sprint 3 – Live Trading, Risk Management & Production‑Ready Ops  

Sprint 3 takes the **signal‑generation pipeline** built in Sprint 2 and turns it into a **real‑time, money‑on‑the‑line trading bot** that can run 24/7 on the Coinbase exchange.  
The focus is on **execution safety, capital protection, observability, and the ability to iterate quickly** (e.g., swapping LLMs, tweaking risk parameters, or adding new data sources).

At the end of Sprint 3 you should have a **fully automated, monitored, and kill‑switch‑able trading system** that can be started with a single script and that respects the risk‑management rules you define.

| ✅ | Sprint 3 Deliverable |
|----|----------------------|
| 1 | **Order‑execution engine** that translates a `BUY/SELL/HOLD` decision into signed Coinbase Pro (or Coinbase Advanced Trade) API calls, with guard‑rails for slippage, order‑type selection, and retry logic. |
| 2 | **Risk‑management module** (position sizing, stop‑loss/take‑profit, max‑drawdown, exposure caps, per‑symbol limits). |
| 3 | **Live‑trading loop** (async scheduler) that pulls the latest signal, runs the risk checks, sends the order, and records the outcome. |
| 4 | **Trade‑ledger** (PostgreSQL/TimescaleDB) that stores every order, fill, P&L, and the exact feature snapshot that produced the signal (for future retraining). |
| 5 | **Monitoring & alerting** (Grafana dashboards, Prometheus metrics, Slack/Telegram alerts for failures, large losses, or risk‑limit breaches). |
| 6 | **Graceful shutdown / kill‑switch** that can be invoked via CLI, HTTP endpoint, or a cloud‑function, guaranteeing all open positions are closed or safely parked. |
| 7 | **Comprehensive test suite** (unit, integration, fault‑injection) with CI coverage ≥ 85 % and a “dry‑run” mode that simulates the entire pipeline against historic data. |
| 8 | **Documentation & run‑books** covering deployment, credential handling, emergency procedures, and how to swap the LLM or adjust risk parameters. |

Below is a **task‑by‑task breakdown** (≈ 2 weeks for a small team). Each task lists a concrete “Done” criterion and suggested unit‑test ideas.

---

## 1️⃣ Order‑Execution Engine (Coinbase)

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 1.1 | **API wrapper** (`coinbase_client.py`) that supports: <br>• `GET /accounts` (balance) <br>• `POST /orders` (limit, market, stop) <br>• `GET /fills` (order status) <br>All calls use **signed HMAC‑SHA256** authentication. | `client.get_balance('BTC')` returns a `Decimal` without raising. | `test_coinbase_client_auth.py` – mock `requests` to capture headers, assert correct signature format. |
| 1.2 | **Order‑type selector** – function `choose_order_type(signal, market_depth)` that returns: <br>• `LIMIT` with `post_only` for normal entries <br>• `MARKET` + immediate `STOP_LOSS` for high‑impact news <br>• `TWAP` (split into N sub‑orders) for low‑liquidity altcoins. | For a `BUY` signal with `depth_ratio > 0.8` the function returns `LIMIT`. | `test_order_type_selector.py`. |
| 1.3 | **Slippage guard** – before sending an order, query the order‑book (`GET /products/<symbol>/book?level=2`) and compute the **expected fill price**. Abort if `expected_slippage > max_slippage` (configurable, e.g., 0.2 %). | When the book shows a 0.5 % spread, the order is rejected and logged. | `test_slippage_guard.py` – mock book response, assert abort path. |
| 1.4 | **Retry & idempotency** – on transient network errors or `429 Too Many Requests`, retry with exponential back‑off (max 5 attempts). Store a **client‑generated UUID** with each order; if the same UUID is seen again, skip duplicate submission. | A simulated `500` error is retried 3 times then succeeds; duplicate UUID is ignored. | `test_retry_idempotency.py`. |
| 1.5 | **Rate‑limit awareness** – read `X‑Rate‑Limit‑Remaining` header; if below a threshold, pause the whole execution loop for the reset period. | When header reports `0`, the loop sleeps for the indicated `X‑Rate‑Limit‑Reset` seconds. | `test_rate_limit_handling.py`. |
| 1.6 | **Unit‑test harness** – spin up a **mock Coinbase server** (e.g., using `responses` or `httpx.MockTransport`) and run through a full order lifecycle (create → fill → cancel). | End‑to‑end test passes with no real network calls. | `test_full_order_lifecycle.py`. |

---

## 2️⃣ Risk‑Management Module

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 2.1 | **Position‑sizing calculator** (`risk_manager.py`) that implements: <br>• `max_risk_per_trade = risk_pct * equity` (default 1 %) <br>• `size = max_risk_per_trade / (ATR * stop_multiplier)` <br>• enforce `min_order_qty` per exchange. | For a $10 k equity, 1 % risk, ATR = $200, stop = 2 × ATR → size ≈ 0.025 BTC. | `test_position_sizing.py`. |
| 2.2 | **Stop‑loss / take‑profit generator** – given entry price and `risk_reward_ratio` (default 2), compute `stop_price` and `tp_price`. | With entry = $30 000, stop = $29 400, TP = $31 200 (RR = 2). | `test_sl_tp_calculation.py`. |
| 2.3 | **Exposure caps** – per‑symbol max % of equity (e.g., 20 %); global max % of total capital in open positions (e.g., 50 %). | When the bot already holds $2 k of BTC (20 % of $10 k equity) and a new BTC signal arrives, the engine rejects the trade. | `test_exposure_limits.py`. |
| 2.4 | **Draw‑down monitor** – track daily and intra‑day draw‑down; if `drawdown > drawdown_limit` (e.g., –5 %), automatically pause trading and send an alert. | Simulated equity curve that drops 6 % triggers a pause flag. | `test_drawdown_monitor.py`. |
| 2.5 | **Leverage & margin safety** – if using margin, ensure `margin_used / equity < max_margin_pct`. | With 2× leverage and equity = $10 k, margin_used = $15 k → violation (max = 1.5×). | `test_margin_safety.py`. |
| 2.6 | **Risk‑parameter config** – YAML/JSON file (`risk_config.yaml`) that can be hot‑reloaded without restarting the bot. | Changing `risk_pct` from `0.01` to `0.02` takes effect after the next signal. | `test_config_reload.py`. |

---

## 3️⃣ Live‑Trading Loop (Scheduler)

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 3.1 | **Async scheduler** (`trading_loop.py`) that runs every `N` seconds (default 30 s): <br>1. Pull latest signal from the Decision API (`/signal/<symbol>`). <br>2. Pass signal + market data to `risk_manager`. <br>3. If `decision != HOLD` and risk passes → call `order_execution`. | Running the script prints “Signal = BUY → order placed (size = 0.02 BTC)”. |
| 3.2 | **State store** – in‑memory dict `open_positions` keyed by `order_id`; persisted to TimescaleDB after each fill. | After a BUY, `open_positions` contains the order UUID and entry price. |
| 3.3 **Exit logic** – on each tick, check existing positions: <br>• If price reaches `stop_price` or `tp_price` → send `STOP_ORDER`/`MARKET` to close. <br>• If a time‑based exit (e.g., max holding 4 h) → close. | A position opened at $30 000 with stop = $29 400 is automatically closed when the market price drops below that level. |
| 3.4 | **Dry‑run mode** (`--dry-run` flag) – all API calls are mocked; the loop still writes to the trade ledger so you can replay the exact sequence later. | Running with `--dry-run` produces a ledger file but no real orders appear on Coinbase. |
| 3.5 | **Graceful shutdown hook** – on SIGINT/SIGTERM, stop the scheduler, wait for in‑flight orders to settle, then optionally close all open positions (configurable). | `Ctrl‑C` → “Shutting down… closing 2 open positions… done.” |
| 3.6 | **Unit‑test** – use `pytest‑asyncio` to run the loop for 3 iterations with mocked signal and mocked Coinbase client; assert that the expected number of orders were placed and that the ledger contains the right rows. | `test_trading_loop_async.py`. |

---

## 4️⃣ Trade Ledger (Persisted Audit Trail)

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 4.1 | **Schema design** – table `trade_ledger` (JSONB `feature_snapshot`, `signal`, `decision`, `order_id`, `status`, `entry_price`, `exit_price`, `size`, `pnl`, timestamps). | `\d trade_ledger` shows all columns, `feature_snapshot` is `JSONB`. |
| 4.2 | **Insert helper** (`ledger.py::record_trade`) that is called after every order event (placed, filled, cancelled, closed). | After a mock fill, a row appears with `status='FILLED'`. |
| 4.3 | **Historical query API** (`GET /ledger?symbol=BTC&from=2024-01-01`) – returns JSON list of trades for UI/analysis. | `curl http://localhost:8000/ledger?symbol=BTC` returns a 200 with an array. |
| 4.4 | **Data integrity checks** – foreign‑key‑like constraint that `order_id` is unique; `feature_snapshot` must contain the same keys that the model expects. | Attempting to insert a duplicate `order_id` raises an exception caught and logged. |
| 4.5 | **Unit‑test** – insert a fake trade, query it back, compare fields. | `test_ledger_insert_and_query.py`. |

---

## 5️⃣ Monitoring, Alerting & Observability

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 5.1 | **Prometheus metrics exporter** (`metrics.py`) exposing: <br>• `bot_up` (1/0) <br>• `signals_total`, `orders_total`, `orders_success`, `orders_failed` <br>• `equity`, `drawdown`, `open_positions` <br>• `latency_seconds` (time from signal fetch → order submit). | `curl http://localhost:8000/metrics` returns a text payload with the above metrics. |
| 5.2 | **Grafana dashboards** – import JSON dashboards that plot equity curve, draw‑down, signal frequency, order success rate, and a real‑time “order book heatmap”. | Opening Grafana shows the dashboards with live data (use the dev Docker network). |
| 5.3 | **Alerting** – Prometheus rule `ALERT HighDrawdown` (drawdown > 5 %) → send a webhook to Slack/Telegram via Alertmanager. | Simulating a draw‑down > 5 % triggers a Slack message (captured by a mock webhook). |
| 5.4 | **Health‑check endpoint** (`GET /health`) that returns JSON `{status: "ok", equity: 12345.67}` and HTTP 200. | `curl http://localhost:8000/health` → 200. |
| 5.5 | **Log aggregation** – configure Python `logging` to output JSON lines to stdout; Docker‑compose collects them with `fluentd` or ` Loki`. | Logs appear in Grafana Loki UI with searchable fields (`level`, `module`, `order_id`). |
| 5.6 | **Unit‑test** – mock Prometheus client, call metric increment functions, assert the internal counter changed. | `test_metrics_increment.py`. |

---

## 6️⃣ Graceful Shutdown / Kill‑Switch

| # | Sub‑task | Done‑criterion |
|---|----------|----------------|
| 6.1 | **CLI command** `./stop_bot.sh` that sends `SIGTERM` to the main process and then runs `./close_all_positions.py`. |
| 6.2 | **HTTP endpoint** `POST /shutdown` (protected by an API key) that triggers the same flow. |
| 6.3 | **Auto‑close script** (`close_all_positions.py`) that iterates over `open_positions`, sends market orders to exit, waits for fills, logs the result, and finally writes a “bot_stopped” entry to the ledger. |
| 6.4 | **Verification test** – start the bot in dry‑run mode, open two mock positions, invoke the shutdown endpoint, assert both positions are marked `CLOSED` in the ledger. |
| 6.5 | **Documentation** – a run‑book page “Emergency Procedure” with step‑by‑step screenshots. |

---

## 7️⃣ Testing, CI & Documentation (Wrap‑Up)

| # | Sub‑task | Done‑criterion |
|---|----------|----------------|
| 7.1 | **Fault‑injection tests** – use `toxiproxy` or a custom mock to simulate: <br>• Coinbase API latency spikes (5 s) <br>• Order‑book depth collapse (slippage > max) <br>• Websocket disconnects. Verify the bot logs the error, respects the guard, and continues. |
| 7.2 | **CI pipeline** – add stages: `lint → unit → integration → smoke → coverage`. Fail the pipeline if coverage < 85 % or any integration test times out. |
| 7.3 | **Dry‑run back‑test runner** (`backtest_live.py`) that replays a historic period using the **live‑trading loop** but with `--dry-run`. Output a performance report identical to the one from Sprint 2, proving the loop works end‑to‑end. |
| 7.4 | **README / Run‑books** – sections: <br>• “One‑click start” (`./run_all.sh`) <br>• “How to change risk parameters” <br>• “How to swap the LLM model” (replace `LLM_ENDPOINT` env var) <br>• “Emergency shutdown”. |
| 7.5 | **Versioning & reproducibility** – store the git SHA, Docker image tags, and model checksum in a `deployment_info.json` file that is written at startup and also persisted to the ledger. |
| 7.6 | **Demo checklist** – a short script that: <br>1. Starts the stack (`docker compose up -d`). <br>2. Waits for health checks. <br>3. Fires a mock signal (via a test endpoint). <br>4. Verifies an order appears in the ledger. <br>5. Triggers the kill‑switch and confirms all positions are closed. |

---

## 8️⃣ Timeline (2 Weeks)

| Day | Focus |
|-----|-------|
| **Day 1‑2** | Coinbase client wrapper, auth tests, order‑type selector. |
| **Day 3** | Slippage guard, retry/idempotency, rate‑limit handling. |
| **Day 4‑5** | Risk‑manager (position sizing, stop‑loss/TP, exposure caps, draw‑down monitor). |
| **Day 6** | Live‑trading loop (async scheduler, dry‑run flag, graceful shutdown). |
| **Day 7** | Trade ledger schema, insert helper, API endpoint. |
| **Day 8** | Prometheus exporter, Grafana dashboards, alert rules. |
| **Day 9** | Kill‑switch CLI & HTTP endpoint, auto‑close script. |
| **Day 10** | Fault‑injection tests (latency, slippage, API failures). |
| **Day 11** | Integration test (full stack, dry‑run back‑test). |
| **Day 12** | CI pipeline update, coverage enforcement. |
| **Day 13** | Documentation, run‑books, versioning script. |
| **Day 14** | Buffer / bug‑fix day, sprint demo preparation. |

---

## 9️⃣ Sprint‑3 Demo Checklist

1. **Start the system** with `./run_all.sh`. Health endpoint returns `status: ok`.  
2. **Signal flow** – `curl http://localhost:8000/signal/BTC-USD` returns a fresh decision.  
3. **Risk check** – the bot decides to BUY, size is calculated, and a **LIMIT** order is placed on the Coinbase sandbox.  
4. **Ledger entry** – the order appears in `trade_ledger` with the full feature snapshot.  
5. **Monitoring** – Grafana shows the equity curve moving, Prometheus metric `orders_success_total` increments, and no alerts fire.  
6. **Kill‑switch** – invoking `POST /shutdown` (or `./stop_bot.sh`) closes any open position, writes a “bot_stopped” record, and the process exits cleanly.  
7. **All tests** – CI pipeline passes with ≥ 85 % coverage, integration test reports “PASS”.  

Once the demo is approved, **Sprint 4** can focus on **advanced LLM orchestration (multi‑model ensembles), reinforcement‑learning policy optimisation, and scaling the bot to multiple symbols / exchanges**.

Good luck, and may your risk‑adjusted returns be ever in your favour! 🚀
