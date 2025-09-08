
## Sprint 4 – Advanced LLM Orchestration, RL‑Based Policy Optimisation & Multi‑Exchange Scaling  

Sprint 4 takes the **single‑symbol, single‑exchange, rule‑based + LLM hybrid** system built in Sprint 3 and upgrades it to a **truly agentic, self‑optimising trading platform** that can:

1. **Run multiple LLMs (or LLM‑agents) in parallel** – each specialised for a data domain (news, on‑chain metrics, social‑sentiment, macro‑economics).  
2. **Learn a trading policy with reinforcement learning (RL)** that continuously improves from its own execution experience while staying inside the hard risk limits defined earlier.  
3. **Scale horizontally across many symbols (top‑50 crypto pairs) and across several exchanges (Coinbase, Kraken, Binance, KuCoin, etc.)** with a single orchestrator.  
4. **Provide a plug‑and‑play “agent library”** so new data sources or new LLMs can be added without touching the core execution engine.  

At the end of Sprint 4 you will have a **production‑ready, multi‑agent, RL‑augmented trading orchestrator** that can be launched with one command, monitored centrally, and safely stopped at any time.

| ✅ | Sprint 4 Deliverable |
|----|----------------------|
| 1 | **Agent Orchestrator** – a lightweight service that schedules, runs, and aggregates the outputs of multiple LLM agents per symbol/interval. |
| 2 | **RL Policy Engine** – an off‑policy algorithm (e.g., Soft Actor‑Critic or PPO) that receives the aggregated LLM context + technical features and outputs a continuous action (position size, direction, stop‑loss/take‑profit). |
| 3 | **Multi‑Exchange Execution Layer** – abstracted order‑router that can send the same logical order to any of the supported exchanges, handling differing fee structures, order‑book formats, and rate limits. |
| 4 | **Dynamic Symbol Manager** – a registry that can add/remove symbols at runtime, automatically provisioning the required data‑feeds, agents, and risk caps. |
| 5 | **Continuous‑Learning Pipeline** – nightly (or hourly) replay of the most recent trades to update the RL model, with a “safe‑mode” fallback to the rule‑based hybrid model. |
| 6 | **Observability for Agents & RL** – per‑agent latency, token‑usage, reward‑signal dashboards, and automated alerts when an agent’s performance degrades. |
| 7 | **Comprehensive test suite** (unit, integration, fault‑injection, RL‑stability) with CI coverage ≥ 90 %. |
| 8 | **Documentation & Run‑books** for adding new agents, tuning RL hyper‑parameters, and scaling to additional exchanges. |

Below is a **task‑by‑task breakdown** (≈ 2 weeks for a small, focused team). Each task lists a concrete “Done” criterion and suggested unit‑test ideas.

---

## 1️⃣ Agent Orchestrator (Multi‑LLM Coordination)

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 1.1 | **Agent interface definition** (`BaseAgent`) with methods `async def fetch_context(self, symbol, ts) -> dict` and `async def infer(self, context) -> dict`. | Abstract base class can be subclassed; `issubclass(MyAgent, BaseAgent)` is true. | `test_base_agent_abc.py` – ensure `NotImplementedError` is raised for abstract methods. |
| 1.2 | **Implement concrete agents**: <br>• `NewsAgent` (uses OpenAI GPT‑4‑Turbo) <br>• `OnChainAgent` (uses Llama‑2‑70B with LoRA) <br>• `MacroAgent` (calls a proprietary macro‑API). | Each agent returns a JSON payload with keys `bias`, `confidence`, `rationale`. | Mock the respective LLM APIs and assert the returned dict matches schema (`test_news_agent.py`, `test_onchain_agent.py`). |
| 1.3 | **Orchestrator service** (`orchestrator.py`) that, for each symbol & timestamp, runs all registered agents **concurrently** (via `asyncio.gather`) and aggregates their outputs into a single `AggregatedContext`. Aggregation strategy: weighted average of confidences (weights configurable per agent). | `await orchestrator.aggregate('BTC-USD', ts)` returns a dict with fields `bias`, `confidence`, `agent_breakdown`. |
| 1.4 | **Dynamic agent registration** – a YAML file (`agents.yaml`) that lists enabled agents per symbol. Orchestrator watches the file (via `watchdog`) and hot‑reloads without restart. | Adding a new entry to `agents.yaml` makes the orchestrator start invoking that agent on the next tick. | `test_dynamic_agent_reload.py` – modify the file during test, assert new agent is called. |
| 1.5 | **Timeout & fallback** – each agent call has a per‑call timeout (default 2 s). If an agent times out, its contribution is omitted and a warning is logged. | Simulating a 5 s delay in `NewsAgent` results in a log entry “NewsAgent timeout” and the aggregated confidence is computed from the remaining agents. | `test_agent_timeout.py`. |
| 1.6 | **Metrics exporter** – Prometheus counters: `agent_calls_total`, `agent_success_total`, `agent_timeout_total`, `orchestrator_latency_seconds`. | `curl http://localhost:8000/metrics` shows the new metrics. | `test_orchestrator_metrics.py`. |

---

## 2️⃣ RL Policy Engine (Learning the Trading Policy)

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 2.1 | **Environment wrapper** (`rl_env.py`) that implements the OpenAI‑Gym API (`reset`, `step`, `observation_space`, `action_space`). Observation = concatenation of: <br>• Technical indicator vector (from TimescaleDB) <br>• Aggregated LLM context (bias, confidence) <br>• Position state (size, entry price). Action = continuous vector `[direction ∈ {‑1,0,1}, size ∈ [0,1]]`. | `env.reset()` returns a NumPy array of the correct shape; `env.step(action)` returns `(obs, reward, done, info)`. |
| 2.2 | **Reward design** – primary component: **realised P&L** (scaled by risk‑adjusted Sharpe). Penalties for: <br>• Exceeding max draw‑down <br>• Violating exposure caps <br>• High slippage (> max_slippage). | Running a short episode on historic data yields a total reward that correlates positively with final equity. |
| 2.3 | **Algorithm choice** – implement Soft Actor‑Critic (SAC) using **Stable‑Baselines3** (or a custom PyTorch implementation). Store model checkpoints (`policy.pt`, `critic.pt`). | After 10 k training steps on a 1‑day historic slice, the policy’s average episode reward improves > 5 % over a random baseline. |
| 2.4 | **Training pipeline** (`train_rl.py`) that: <br>1. Loads the latest market data (last 30 days). <br>2. Generates a replay buffer (experience tuples). <br>3. Trains the SAC agent for a configurable number of steps. <br>4. Saves the checkpoint and logs metrics to TensorBoard. | Running `python train_rl.py --steps 20000` produces a `runs/` folder with TensorBoard logs and a checkpoint file. |
| 2.5 | **Online inference wrapper** (`rl_policy.py`) that loads the latest checkpoint and, given an observation, returns an action. Must be **thread‑safe** (use a `torch.no_grad()` context). | `await rl_policy.predict(obs)` returns a dict `{direction: 1, size: 0.03}` within < 10 ms. |
| 2.6 | **Safety guard** – before executing the RL‑suggested order, run the **Risk‑Manager** (from Sprint 3). If the RL action violates any hard limit, fall back to the **Hybrid rule‑based signal**. | Simulating a RL action that would exceed the per‑symbol exposure cap results in the fallback path being taken and logged. |
| 2.7 | **Continuous‑learning job** – a nightly cron (Docker‑Cron or Airflow) that: <br>1. Pulls the last 24 h of executed trades from `trade_ledger`. <br>2. Re‑creates the environment, updates the replay buffer, performs a few gradient steps, and writes a new checkpoint. <br>3. Emits a Prometheus metric `rl_training_success_total`. | After the nightly job runs, the checkpoint file’s modification timestamp updates and the metric increments. |
| 2.8 | **Unit‑tests** – mock the environment to return deterministic observations and rewards; verify that a single training step reduces the TD‑error. | `test_rl_training_step.py`. |

---

## 3️⃣ Multi‑Exchange Execution Layer

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 3.1 | **Exchange abstraction** (`exchange/base.py`) with methods `async def place_order(self, symbol, side, size, price, order_type)`, `async def get_balance(self, asset)`, `async def cancel_order(self, order_id)`. Concrete subclasses: `CoinbaseProExchange`, `KrakenExchange`, `BinanceExchange`. | Each subclass implements the abstract methods and can be instantiated via a factory (`ExchangeFactory.create('coinbase')`). |
| 3.2 | **Fee & precision handling** – each exchange class loads its **fee schedule** and **price/size precision** from a JSON config (`exchange_config.json`). The `place_order` method automatically rounds to the allowed precision and adds the fee to the order size if needed. | Placing a 0.123456 BTC order on Binance (precision 0.000001) results in a rounded size of 0.123456. |
| 3.3 | **Unified order router** (`router.py`) that receives a logical order (symbol, side, size, price, order_type) and dispatches it to **all enabled exchanges** according to a **routing strategy** (e.g., “best‑price”, “split‑equally”, “priority list”). | With a split‑equally strategy across Coinbase and Kraken, a 0.02 BTC order results in two 0.01 BTC sub‑orders, each sent to the respective exchange. |
| 3.4 | **Exchange‑specific error handling** – map each exchange’s error codes to a common set (`RateLimit`, `InsufficientFunds`, `InvalidOrder`). The router retries only on transient errors. | Simulating a `RateLimit` error from Kraken triggers exponential back‑off and a retry; after 3 failures the order is marked `FAILED`. |
| 3.5 | **Order‑status synchroniser** – a background task polls each exchange for open orders, updates the `trade_ledger` with fills, and reconciles any mismatches (e.g., partial fills). | After a partial fill on Binance, the ledger shows `filled_qty = 0.006` and `remaining_qty = 0.004`. |
| 3.6 | **Metrics** – counters per exchange: `orders_sent_total`, `orders_filled_total`, `orders_failed_total`, `average_latency_seconds`. | `curl http://localhost:8000/metrics` includes `orders_sent_total{exchange="coinbase"} 42`. |
| 3.7 | **Unit‑tests** – mock each exchange’s HTTP API (using `responses` or `httpx.MockTransport`) and verify routing logic, fee rounding, and error mapping. | `test_router_split_strategy.py`, `test_exchange_error_mapping.py`. |

---

## 4️⃣ Dynamic Symbol Manager

| # | Sub‑task | Done‑criterion | Unit‑test |
|---|----------|----------------|-----------|
| 4.1 | **Symbol registry** (`symbols.yaml`) – each entry contains: <br>• `symbol` (e.g., `BTC-USD`) <br>• `exchanges` list (ordered) <br>• `risk_limits` (max % equity, max % per‑exchange) <br>• `agents` list (which agents to run). | Adding a new entry for `ETH-USDT` makes the orchestrator start pulling data for that pair on the next tick. |
| 4.2 | **Manager service** (`symbol_manager.py`) that watches the registry file, validates entries (symbol exists on each listed exchange, risk limits are sane), and updates the orchestrator, risk manager, and data‑feed subscriptions accordingly. | Removing `BTC-USD` from the file causes the orchestrator to stop processing it within 30 s. |
| 4.3 | **Automatic data‑feed provisioning** – when a new symbol is added, the manager creates the necessary Kafka topics (`price_<symbol>`, `news_<symbol>`, `tweets_<symbol>`) and starts the corresponding ingestion workers (reuse the generic Coinbase WS client with the new symbol). | After adding `SOL-USD`, a new Kafka topic `price_SOL-USD` appears and receives candle messages. |
| 4.4 | **Per‑symbol risk caps** – the Risk‑Manager now receives the symbol‑specific limits from the manager and enforces them individually. | A trade that would push `SOL-USD` exposure above its 10 % cap is rejected, while other symbols remain unaffected. |
| 4.5 | **Unit‑tests** – modify the `symbols.yaml` during test, assert that the manager creates/destroys the expected Kafka topics and that the orchestrator’s internal symbol list updates. | `test_symbol_manager_add_remove.py`. |

---

## 5️⃣ Continuous‑Learning Pipeline (RL + Hybrid Fallback)

| # | Sub‑task | Done‑criterion |
|---|----------|----------------|
| 5.1 | **Replay buffer builder** – a nightly job (`build_replay.py`) that reads the last 48 h of `trade_ledger`, reconstructs the full observation (technical + aggregated LLM context) for each timestep, and stores the tuples `(obs, action, reward, next_obs, done)` in a **SQLite** or **Redis** buffer. |
| 5.2 | **Hybrid fallback policy** – a lightweight model (`fallback_policy.py`) that replicates the rule‑based hybrid signal from Sprint 3. The RL trainer loads this policy and uses it as a **behavior‑cloning baseline** for the first few epochs. |
| 5.3 | **Safety‑first deployment** – the production orchestrator runs the RL policy **only if** the latest validation metrics (average episode reward, Sharpe, max‑drawdown) exceed pre‑defined thresholds; otherwise it automatically switches to the fallback policy. |
| 5.4 | **A/B testing harness** – a separate “shadow” process runs the RL policy in parallel on a **paper‑trading** account, logs its decisions, and compares performance against the live fallback policy. Results are stored in `ab_test_results` table for post‑mortem analysis. |
| 5.5 | **Unit‑test** – simulate a small replay buffer (10 steps), run one training epoch, and assert that the loss decreases and that the fallback policy is correctly loaded when validation fails. |
| 5.6 | **Metrics** – Prometheus gauges: `rl_validation_reward`, `rl_validation_sharpe`, `rl_policy_active` (1 = RL, 0 = fallback). Alerts fire if `rl_policy_active` flips unexpectedly. |

---

## 6️⃣ Observability for Agents & RL

| # | Sub‑task | Done‑criterion |
|---|----------|----------------|
| 6.1 | **Per‑agent latency & token usage** – each agent reports `agent_latency_seconds` and `agent_tokens_used` (if using OpenAI/Anthropic). Exported as Prometheus metrics with label `agent_name`. |
| 6.2 | **RL reward & policy drift dashboards** – Grafana panels that plot: <br>• Episode reward over time <br>• Policy entropy (to detect collapse) <br>• Distribution of actions (size, direction). |
| 6.3 | **Agent performance monitor** – compute a rolling **information‑ratio**: (average reward of agent‑augmented signal – baseline reward) / std‑dev. If the ratio drops below a threshold for 3 consecutive windows, raise an alert. |
| 6.4 | **Logging** – structured JSON logs include `agent`, `symbol`, `timestamp`, `bias`, `confidence`, `reward`, `action`. Logs are shipped to Loki/Elastic for searchable audit. |
| 6.5 | **Unit‑test** – mock an agent to emit a known latency and token count; assert that the Prometheus counters increase accordingly (`test_agent_metrics.py`). |

---

## 7️⃣ Testing, CI & Documentation (Wrap‑Up)

| # | Sub‑task | Done‑criterion |
|---|----------|----------------|
| 7.1 | **Fault‑injection suite** – use `toxiproxy` to simulate: <br>• LLM latency spikes (10 s) <br>• Partial Kafka message loss <br>• Exchange API 503 bursts. Verify that the orchestrator degrades gracefully (fallback policy, circuit‑breaker). |
| 7.2 | **CI pipeline** – add stages: `lint → unit → integration → RL‑stability → smoke → coverage`. RL‑stability runs a short training loop on a tiny synthetic dataset to ensure no NaNs or exploding gradients. |
| 7.3 | **Integration test** – spin up the full stack (Kafka, TimescaleDB, Redis, two mock exchanges, two LLM mock servers). Run the orchestrator for 5 minutes in “paper‑trading” mode, then assert: <br>• At least one RL‑generated order was placed <br>• All agents produced outputs <br>• No unhandled exceptions. |
| 7.4 | **Documentation** – update `README.md` with sections: <br>• “Adding a new LLM agent” (step‑by‑step) <br>• “Tuning RL hyper‑parameters” (learning rate, entropy coefficient) <br>• “Scaling to N symbols & M exchanges” (resource sizing). |
| 7.5 | **Run‑books** – emergency procedures: <br>• “Force‑switch to fallback policy” <br>• “Terminate all open positions across exchanges” <br>• “Rollback RL checkpoint”. |
| 7.6 | **Versioning** – each deployment writes a `deployment_manifest.json` containing: <br>• Git SHA <br>• List of enabled agents <br>• RL checkpoint hash <br>• Exchange config version. This file is stored in the ledger for audit. |
| 7.7 | **Coverage target** – `coverage run -m pytest && coverage report -m` must show **≥ 90 %** overall, with **≥ 80 %** on RL‑related modules. |

---

## 8️⃣ Timeline (2 Weeks)

| Day | Focus |
|-----|-------|
| **Day 1‑2** | Agent base class, concrete agents, orchestrator concurrency, timeout handling. |
| **Day 3** | Prometheus metrics for agents, dynamic agent registration. |
| **Day 4‑5** | RL environment wrapper, reward design, SAC implementation, training script. |
| **Day 6** | RL safety guard (fallback integration) and nightly training job. |
| **Day 7** | Exchange abstraction layer, routing strategies, fee/precision handling. |
| **Day 8** | Multi‑exchange order router, status synchroniser, metrics. |
| **Day 9** | Symbol manager (registry, hot‑reload, data‑feed provisioning). |
| **Day 10** | Continuous‑learning pipeline (replay buffer, A/B shadow runner). |
| **Day 11** | Observability dashboards (Grafana panels, agent performance alerts). |
| **Day 12** | Fault‑injection tests, RL‑stability CI stage. |
| **Day 13** | Full‑stack integration test, documentation & run‑books. |
| **Day 14** | Buffer / bug‑fix day, sprint demo preparation. |

---

## 9️⃣ Sprint 4 Demo Checklist

1. **Start the full system** with `./run_all.sh`. Health endpoint returns `status: ok`.  
2. **Add a new symbol** (`SOL-USD`) to `symbols.yaml` – the orchestrator begins ingesting candles, agents start producing context, and the RL policy begins generating actions.  
3. **Observe agent metrics** in Grafana (latency < 2 s, token usage displayed).  
4. **Watch the RL policy** – a live plot shows the action direction and size changing over time; the fallback policy is shown as a secondary line.  
5. **Place a real (paper‑trading) order** on two exchanges simultaneously via the router; verify the ledger records both sub‑orders and their fills.  
6. **Trigger a fault** (e.g., block the NewsAgent with a 10 s delay) – the orchestrator logs a timeout, the aggregated confidence drops, and the RL policy adapts (action magnitude reduces).  
7. **Run the nightly training job** (manually trigger `python train_rl.py --steps 50000`). Confirm a new checkpoint appears and the Prometheus metric `rl_training_success_total` increments.  
8. **Switch to fallback** – modify the validation threshold to force the system into fallback mode; confirm `rl_policy_active` metric flips to 0 and the decision engine now uses the rule‑based hybrid signal.  
9. **Execute the kill‑switch** (`POST /shutdown` or `./stop_bot.sh`). All open positions on every exchange are closed, the orchestrator stops, and a final entry `deployment_status: stopped` is written to the ledger.  

When the demo passes, the platform is ready for **Sprint 5**, where you can explore **meta‑learning across markets, hierarchical multi‑agent coordination, and full‑scale production deployment on cloud‑native infrastructure**.

--- 

**Good luck!** With the modular agent/orchestrator design, the RL policy, and the multi‑exchange execution layer in place, you now have a **scalable, self‑optimising crypto‑trading bot** that can be extended indefinitely while staying safely bounded by the risk framework you built in earlier sprints. 🚀
