---
name: AI ML and local LLM advisor
overview: Add a reproducible data, numerical-ML, and local Ollama/Gemma advisory pipeline without allowing an LLM to sign, approve, broadcast, or override deterministic risk controls.
isProject: false
todos:
  - id: ai-boundary
    content: "Define AI responsibilities, threat model, model registry, and typed advisory contracts"
    status: in_progress
  - id: ollama-adapter
    content: "Integrate the local Ollama model through a timeout-bound structured-output advisor"
    status: pending
  - id: dataset-ingestion
    content: "Build immutable historical and live market-data ingestion with provenance and checksums"
    status: pending
  - id: feature-label-pipeline
    content: "Create leakage-resistant features, labels, time splits, and replayable datasets"
    status: pending
  - id: numerical-signal
    content: "Add a low-latency deterministic and numerical-ML signal path before any LLM promotion"
    status: pending
  - id: llm-evaluation
    content: "Evaluate Gemma advisory quality, latency, stability, and model-drift behavior"
    status: pending
  - id: ai-paper-gate
    content: "Run AI in shadow/paper mode and gate any production influence behind evidence"
    status: pending
---

## Phases

P1 — Bound AI so it cannot become an uncontrolled signer or trader.
P2 — Make local inference observable and structured.
P3 — Build clean, provenance-preserving data for research.
P4 — Establish a fast signal path independent of LLM latency.
P5 — Evaluate Gemma and paper-trade AI influence safely.

## Slices

| Phase | Slice | Objective | Owner | Gates | Depends on |
|---|---|---|---|---|---|
| P1 | `ai-boundary` | Define advisory-only AI contracts and threat model | direct | architecture/security review | strategy `strategy-contract`; secure `environment-gate` |
| P2 | `ollama-adapter` | Integrate local Ollama safely | direct | contract tests; timeout/failure tests | `ai-boundary` |
| P3 | `dataset-ingestion` | Acquire and preserve research data | direct | checksum/provenance tests | strategy `market-data` |
| P3 | `feature-label-pipeline` | Prevent leakage and reproduce datasets | direct | temporal split/golden tests | `dataset-ingestion` |
| P4 | `numerical-signal` | Build the fast day-trading signal path | direct | walk-forward/backtest tests | `feature-label-pipeline`; strategy `risk-engine` |
| P5 | `llm-evaluation` | Benchmark the local LLM as an advisor | direct | offline eval; latency/error budget | `ollama-adapter`; `feature-label-pipeline` |
| P5 | `ai-paper-gate` | Integrate AI in shadow/paper mode only | direct | no-broadcast; drift/manual gate | `numerical-signal`; `llm-evaluation`; strategy `paper-trading` |

## Slice `ai-boundary`

**Objective**

Define AI as two bounded capabilities: a low-latency numerical signal service and a slower local LLM advisor. The numerical path may emit a signal proposal; the LLM may classify regime, summarize context, identify anomalies, and return reason codes. Neither path may access secrets, construct arbitrary calldata/PSBTs, choose a recipient, approve a token, sign, broadcast, withdraw, change risk limits, or enable Production execution.

The initial day-trading scope is spot trading only, on an allowlisted liquid universe, with two baseline strategy families: momentum/breakout and mean reversion. Leveraged derivatives, autonomous sniping of every new token, and unbounded contract discovery are outside this first AI scope.

**Acceptance criteria**

- Typed contracts exist for `MarketSnapshot`, `FeatureVector`, `SignalProposal`, `LlmAdvisory`, `RiskDecision`, and `ModelVersion`.
- Each result includes model id/version, configuration hash, input snapshot id, event time, inference time, confidence, and reason codes.
- LLM output is advisory and expires quickly; missing, malformed, stale, or contradictory output becomes `NoOpinion`.
- DI composition makes it impossible for the AI worker to resolve a signer or broadcast port.
- The threat model covers prompt injection from token metadata/news, malicious contract text, model hallucination, tool-call abuse, model poisoning, data leakage, and denial of service.

**Files to read (context budget)**

- Strategy contracts from `2026-09-01-strategy-risk-and-backtesting.plan.md`.
- `1 - Core/MetaMask.Domain/Ports/**` and EVM/Bitcoin signer ports.
- Secure environment and worker-host plans.
- Local Ollama model inventory and configuration contract.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/AI/**`.
- `1 - Core/MetaMask.Application/AI/**`.
- `4 - Tests/AI/**` boundary/architecture tests.
- `.docs/ai/threat-model.md`.
- `.docs/ai/ai-contract.md`.

**Depends on**

Strategy `strategy-contract`; secure `environment-gate`.

**Complexity**

M.

**Owner**

Direct implementation with architecture and security review.

**Gates**

Architecture test proving no AI-to-signer dependency, threat-model review, and `production-safety-gate`.

**Commit:** `bound ai to advisory trading contracts`

## Slice `ollama-adapter`

**Objective**

Integrate Ollama over its local HTTP API. Default model is the installed `gemma3:4b`, but the model name, digest/fingerprint, context limit, temperature, timeout, and maximum output tokens are configuration and are persisted with each advisory. Use structured JSON schema output and validate it with a strict DTO. Tool calling is disabled for the first implementation; if later enabled, tools are read-only market/telemetry queries selected from a fixed registry.

**Acceptance criteria**

- Ollama health/model mismatch, timeout, overload, malformed JSON, and unavailable service yield typed `NoOpinion`/provider errors and never block execution indefinitely.
- Temperature is zero or a documented low value for classification; prompts contain no secrets and no raw private keys.
- The adapter records prompt template version, model identifier, latency, token counts if available, and output validation result without storing sensitive raw prompts by default.
- The LLM worker is lower priority than quote, risk, execution, and reconciliation queues.
- A test proves an LLM response containing an instruction to sign/broadcast is treated as untrusted text, not a command.

**Files to read (context budget)**

- AI contracts from `ai-boundary`.
- `5 - Worker/MRQ.CryptoBot.Worker/Hosting/**` and queue priorities.
- Secure environment configuration.
- [Ollama chat API](https://docs.ollama.com/api/chat), [structured outputs](https://docs.ollama.com/capabilities/structured-outputs), and [tool calling](https://docs.ollama.com/capabilities/tool-calling).

**Files to create/modify**

- `2 - Adapter/MetaMask.Integration/AI/Ollama/**`.
- `3 - Infra/MetaMask.Infra/AI/**` HTTP client and resilience.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/AiAdvisorWorker.cs`.
- `1 - Core/MetaMask.Domain/AI/**` model registry/config.
- `4 - Tests/AI/**` adapter/contract tests.
- `.docs/ai/ollama.md`.

**Depends on**

`ai-boundary`.

**Complexity**

M.

**Owner**

Direct implementation with security review.

**Gates**

Recorded HTTP fixtures, timeout/cancellation tests, schema validation, prompt-redaction scan, and no-broadcast architecture test.

**Commit:** `integrate local ollama advisory worker`

## Slice `dataset-ingestion`

**Objective**

Create a separate research/data pipeline that stores immutable raw files and normalized events. Start with Binance Public Data for historical spot/futures klines, trades, and aggregate trades; Binance WebSocket for real-time trades/order-book streams; Coinbase candles/WebSocket for cross-venue comparison; and BSC/PancakeSwap RPC event logs and captured executable quotes for DEX-specific data. Provider metadata such as CoinGecko/Moralis may enrich discovery but is never treated as execution truth without provenance. Binance is the first oscillation dataset: begin with 1m, 5m, 15m, and 1h Spot klines for the allowlisted universe, add aggregate trades/order-book data for the most liquid symbols, and keep Futures data separate from Spot labels.

Binance historical acquisition must distinguish two official paths: (1) `data.binance.vision` monthly and daily archives for complete past months/days of Spot, USD-M Futures, and COIN-M Futures klines/trades/aggTrades, with the corresponding `.CHECKSUM` file; and (2) the public REST market-data API for bounded historical windows and reconciliation, using `startTime`/`endTime` pagination because a single kline request is limited. The rolling `ticker` endpoints are current/on-demand statistics, not a downloadable historical ticker series; historical oscillation labels must therefore be reconstructed from archived candles/trades and validated against live ticker calculations. Our raw copy and checksum are immutable for reproducibility, while the collector must retain source-file version metadata because Binance can later publish corrections to archived files.

**Acceptance criteria**

- Every dataset records source, endpoint/file, retrieval time, timezone/unit, symbol/address, chain, block/sequence cursor, license/terms, and checksum.
- Raw data is immutable; normalization creates versioned derived tables/files.
- Binance microsecond timestamps from newer archives are normalized without accidental millisecond conversion.
- The Binance collector computes and stores versioned oscillation features per symbol/window: log return, high-low range, ATR, rolling standard deviation/realized volatility, Parkinson/Garman-Klass volatility where OHLC quality permits, volume surprise, drawdown, maximum favorable/adverse excursion, and future-horizon movement labels.
- Collection supports a symbol manifest, interval selection, start/end range, resumable downloads, rate limits, checksum verification, and a rolling live-ingestion mode without re-downloading immutable archives.
- Monthly and daily Binance archives are addressable by symbol/market/interval/date (for example `spot/monthly/klines/BTCUSDT/1m/BTCUSDT-1m-2025-08.zip`); the downloader records archive availability, missing-file status, checksum, retrieval time, and source revision.
- REST historical backfill paginates `klines`/`aggTrades` by time or trade ID and is used to repair gaps or verify archive samples, not as the sole bulk-training source.
- Spot and Futures datasets cannot be joined silently; market type, contract, quote asset, funding, and leverage metadata are required before use.
- DEX data preserves pool address, token decimals, reserves/liquidity, swap events, gas, quote time, and actual receipt outcome.
- Missing candles, API gaps, reconnects, duplicates, reorgs, and provider corrections are represented explicitly.

**Files to read (context budget)**

- Strategy `market-data` contracts.
- Existing Moralis adapter and EVM event ports.
- [Binance Public Data](https://github.com/binance/binance-public-data).
- [Binance WebSocket streams](https://developers.binance.com/docs/binance-spot-api-docs/web-socket-streams).
- [Coinbase candles](https://docs.cdp.coinbase.com/api-reference/exchange-api/rest-api/products/get-product-candles).

**Files to create/modify**

- `research/data/**` ingestion/normalization scripts.
- `research/data/binance/**` download, checksum, gap detection, normalization, and oscillation-feature jobs.
- `research/config/binance-symbols.yml` or an equivalent versioned symbol/interval manifest containing no credentials.
- `research/pyproject.toml` and lock file for the research runtime.
- `1 - Core/MetaMask.Domain/MarketData/**` provenance contracts.
- `2 - Adapter/MetaMask.Integration/MarketData/**` collectors.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/MarketDataWorker.cs`.
- `4 - Tests/Data/**` checksum/normalization tests.
- `.docs/ai/data-sources.md`.

**Depends on**

Strategy `market-data`.

**Complexity**

L.

**Owner**

Direct implementation with data-engineering review.

**Gates**

Checksum verification, fixture/reconnect tests, timestamp/microsecond tests, gap/duplicate tests, oscillation calculation golden tests, license/provenance review, and no-secret artifact scan.

**Commit:** `add reproducible market data ingestion`

## Slice `feature-label-pipeline`

**Objective**

Build versioned feature and label generation without future leakage. Binance oscillation features are first-class inputs: multi-window returns/ranges, realized volatility, ATR, volume shock, trend strength, drawdown, excursion, and regime transitions. Other features include spread/order-book state, liquidity/price impact, gas, pool reserves, quote freshness, transaction latency, token-risk signals, and cross-venue divergence. Labels are generated only after the configured horizon closes and include net outcome after fees, gas, slippage, failed execution, and latency.

**Acceptance criteria**

- Train/validation/test sets are chronological and support walk-forward evaluation; random row splitting is forbidden for time series.
- A feature manifest identifies timestamp availability and prevents use of future block/trade/receipt information.
- Dataset versions are content-addressed and can be replayed into the backtest engine.
- Labels distinguish no-trade, executable fill, partial/failed fill, and unknown outcome.
- Contract/token metadata used as a feature is timestamped to avoid hindsight from later classifications.
- Oscillation windows and future horizons are explicitly recorded, and labels never use the candle currently being closed or any later correction that was unavailable at decision time.

**Files to read (context budget)**

- Raw/normalized data contracts.
- Strategy backtest and paper-trading plans.
- Risk and execution evidence models.

**Files to create/modify**

- `research/features/**`.
- `research/features/binance-oscillation/**`.
- `research/labels/**`.
- `research/splits/**`.
- `1 - Core/MetaMask.Domain/AI/**` feature/label manifests.
- `4 - Tests/Data/**` leakage/golden tests.
- `.docs/ai/dataset-versioning.md`.

**Depends on**

`dataset-ingestion`.

**Complexity**

L.

**Owner**

Direct implementation with quantitative review.

**Gates**

Temporal split tests, future-column rejection, reproducibility check, and independent dataset review.

**Commit:** `build leakage resistant ai datasets`

## Slice `numerical-signal`

**Objective**

Implement the fast signal path. Start with deterministic momentum/breakout and mean-reversion baselines, then compare a numerical model such as gradient-boosted classification/regression trained in the research pipeline and exported to a supported inference format. The model proposes a signal; the existing risk engine owns approval. Do not put Gemma in this latency-critical path.

**Acceptance criteria**

- Baselines are measured before ML is accepted; ML must beat a no-trade and simple-rule benchmark after realistic costs, not just gross return.
- Inference has a bounded latency budget and a fail-safe fallback to no-trade.
- Model version, feature manifest, calibration, threshold, and training dataset hash are persisted with every signal.
- Model drift and feature-missing conditions produce `NoOpinion` and an alert.
- No model output can bypass token/contract allowlists, preflight, or risk limits.

**Files to read (context budget)**

- Feature/label pipeline.
- Strategy/risk contracts and worker pipeline.
- Backtest engine.

**Files to create/modify**

- `research/models/**`.
- `1 - Core/MetaMask.Domain/AI/**` signal model metadata.
- `1 - Core/MetaMask.Application/Strategy/**` signal adapter.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/SignalWorker.cs`.
- `4 - Tests/AI/**` inference/fallback tests.
- `.docs/ai/numerical-models.md`.

**Depends on**

`feature-label-pipeline`; strategy `risk-engine`.

**Complexity**

L.

**Owner**

Direct implementation with quantitative and production-safety review.

**Gates**

Walk-forward backtest, latency benchmark, model artifact scan, no-secret test, and risk-boundary review.

**Commit:** `add low latency numerical signal path`

## Slice `llm-evaluation`

**Objective**

Evaluate the installed `gemma3:4b` as a local advisor against labeled market snapshots and incident cases. Fine-tuning is not the starting point: first establish prompt/schema quality, then consider LoRA only if a held-out evaluation proves a stable behavioral improvement. Training examples must contain the information available at decision time and must not encode future outcomes in the prompt.

**Acceptance criteria**

- Evaluation includes valid, invalid, adversarial, stale, ambiguous, and missing-data cases.
- Metrics include schema validity, action/reason-code accuracy, calibration, abstention quality, latency p50/p95, memory/CPU impact, and repeatability at fixed settings.
- A model is not promoted because it produces persuasive explanations; it must improve a predefined decision-support metric without increasing unsafe approvals.
- Model prompt/template, Ollama version, model digest, options, dataset hash, and evaluation report are versioned.
- Gemma license/terms and any fine-tuning artifact obligations are recorded before distribution.

**Files to read (context budget)**

- Ollama adapter and evaluation dataset pipeline.
- [Gemma fine-tuning guidance](https://ai.google.dev/gemma/docs/tune).
- [Gemma intended use](https://ai.google.dev/gemma/intended_use_statement) and [terms](https://ai.google.dev/gemma/terms).

**Files to create/modify**

- `research/evaluation/**`.
- `research/prompts/**`.
- `research/fine-tuning/**` only if the baseline fails the promotion decision.
- `4 - Tests/AI/**` evaluation harness.
- `.docs/ai/evaluation.md`.

**Depends on**

`ollama-adapter`; `feature-label-pipeline`.

**Complexity**

L.

**Owner**

Direct implementation with AI/quantitative review.

**Gates**

Held-out temporal evaluation, adversarial prompt tests, repeatability/latency benchmark, license review, and production-safety gate.

**Commit:** `evaluate local llm advisory quality`

## Slice `ai-paper-gate`

**Objective**

Run numerical and LLM outputs in shadow/paper mode through the real quote/risk/queue/monitoring pipeline. Record what the AI would have proposed, what deterministic risk decided, what quote/fill would have happened, and the counterfactual outcome. Live execution remains independent of LLM availability and cannot be enabled by an AI output.

**Acceptance criteria**

- Paper/shadow mode cannot construct a production signer or broadcast port.
- AI failures degrade to the deterministic strategy or no-trade according to explicit policy; they never trigger a hidden fallback to live execution.
- The report compares signal quality, net outcome, false positives, abstention, latency, drift, and operational cost against non-AI baselines.
- Monitoring exposes model health, stale model/data versions, feature drift, and unusual advisory distributions.
- The live-enablement checklist requires human approval of the model version and a rollback to no-AI mode.

**Files to read (context budget)**

- Strategy `paper-trading` and `strategy-gate`.
- Worker pipeline and observability plans.
- All AI slices.

**Files to create/modify**

- `1 - Core/MetaMask.Application/PaperTrading/**` AI shadow records.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/AiAdvisorWorker.cs` and `SignalWorker.cs`.
- `0 - Client/MetaMask.Client/UIs/**` AI mode/status view.
- `4 - Tests/AI/**` no-broadcast/drift tests.
- `.docs/ai/paper-gate.md`.

**Depends on**

`numerical-signal`; `llm-evaluation`; strategy `paper-trading`.

**Complexity**

L.

**Owner**

Direct implementation with production-safety review.

**Gates**

No-broadcast integration, temporal paper report, drift/rollback test, manual observation period, and explicit production approval.

**Commit:** `gate ai influence through shadow and paper trading`

## Deliberation outcome

The panel rejects “LLM as autonomous trader” for the first live scope because it creates latency, nondeterminism, prompt-injection, and authorization risks at the exact boundary that can move funds. It accepts a local Gemma/Ollama advisor because structured output, model/version recording, timeouts, and a strict no-signer boundary make it useful for context and monitoring. It selects deterministic rules and numerical ML for the fast day-trading path, with paper/backtest evidence before live influence.

## Data and model references

- [Binance Public Data](https://github.com/binance/binance-public-data) — daily/monthly historical candles/trades/aggTrades for Spot and Futures, checksums, timestamp details, and archive correction notes.
- [Binance Spot REST API](https://developers.binance.com/en/docs/products/spot/rest-api) — bounded historical requests with `startTime`/`endTime`, pagination, and market-data endpoint behavior.
- [Binance market-data-only endpoints](https://github.com/binance/binance-spot-api-docs/blob/master/faqs/market_data_only.md) — unauthenticated public-data base URL and endpoint boundary.
- [Binance WebSocket streams](https://developers.binance.com/docs/binance-spot-api-docs/web-socket-streams) — live trade and order-book streams.
- [Coinbase candles](https://docs.cdp.coinbase.com/api-reference/exchange-api/rest-api/products/get-product-candles) — cross-venue candles; historical gaps must be represented.
- [Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs) and [tool calling](https://docs.ollama.com/capabilities/tool-calling).
- [Gemma tuning guidance](https://ai.google.dev/gemma/docs/tune) — LoRA/tuning and held-out testing considerations.
