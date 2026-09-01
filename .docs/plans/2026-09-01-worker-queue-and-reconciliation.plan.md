---
name: Worker queue and transaction reconciliation
overview: Separate operator UI, AI advisory, monitoring, and durable execution workers using hosted services, SQLite-backed jobs, bounded channels, idempotency, nonce/UTXO coordination, and graceful shutdown.
isProject: false
todos:
  - id: host-topology
    content: "Create the WinForms host and one or more worker host composition roots with safe defaults"
    status: in_progress
  - id: durable-job-model
    content: "Add durable jobs, leases, idempotency keys, retry classes, and audit transitions"
    status: pending
  - id: worker-pipeline
    content: "Implement quote, risk, execution, receipt, balance, and discovery worker pipelines"
    status: pending
  - id: chain-coordination
    content: "Add EVM nonce coordination, Bitcoin UTXO reservation, and CEX client throttling"
    status: pending
  - id: shutdown-recovery
    content: "Prove crash recovery, graceful shutdown, duplicate delivery, and stuck-job handling"
    status: pending
---

## Phases

P1 — Establish safe host boundaries.
P2 — Make work durable and replay-safe.
P3 — Run market-to-execution flows asynchronously.
P4 — Recover correctly after failures and restarts.

## Slices

| Phase | Slice | Objective | Owner | Gates | Depends on |
|---|---|---|---|---|---|
| P1 | `host-topology` | Add desktop and worker hosts | direct | startup smoke; no broadcast by default | forensic `foundation-gate` + secure `env-contract` |
| P2 | `durable-job-model` | Persist jobs and state transitions | direct | SQLite integration; transition invariants | `host-topology` |
| P3 | `worker-pipeline` | Implement isolated background stages | direct | fake-provider integration tests | `durable-job-model` |
| P3 | `chain-coordination` | Serialize chain-specific resources | direct | concurrency tests | `worker-pipeline` |
| P4 | `shutdown-recovery` | Prove at-least-once recovery without double spend | direct | crash/restart harness | `chain-coordination` |

## Slice `host-topology`

**Objective**

Keep WinForms as an operator/control plane and add a generic host worker executable. Start with one process containing multiple hosted workers for Development and allow separate worker processes for production. Include a lower-priority AI advisor worker and a read-only operations/monitoring worker. Use `BackgroundService`, `PeriodicTimer`, `System.Threading.Channels`, `IHostApplicationLifetime`, and `TimeProvider`; do not introduce Redis/RabbitMQ until a multi-machine requirement exists.

**Acceptance criteria**

- UI submits commands to application services and never calls a blockchain adapter directly.
- Worker host starts with execution disabled and validates configuration before consuming execution jobs.
- Each worker has a unique name, bounded concurrency, cancellation, health state, and correlation id.
- AI and monitoring workers cannot resolve signer/broadcast ports and cannot block quote, risk, execution, or reconciliation queues.
- Separate process mode uses the same ports and database schema as in-process mode.

**Files to read (context budget)**

- `0 - Client/MetaMask.Client/Program.cs` and `Configurations/ConfigureServices.cs`.
- `3 - Infra/MetaMask.Infra/ExternalServiceExtension.cs`.
- `MRQ.CryptoBot.sln`.
- Secure environment host contract from sibling plan.

**Files to create/modify**

- `5 - Worker/MRQ.CryptoBot.Worker/MRQ.CryptoBot.Worker.csproj`.
- `5 - Worker/MRQ.CryptoBot.Worker/Program.cs`.
- `5 - Worker/MRQ.CryptoBot.Worker/Hosting/**`.
- `0 - Client/MetaMask.Client/Program.cs` and composition root.
- `MRQ.CryptoBot.sln`.
- `4 - Tests/**` host smoke tests.

**Depends on**

Forensic `foundation-gate`; secure `env-contract`.

**Complexity**

M.

**Owner**

Direct implementation.

**Gates**

Host startup tests, config validation, cancellation test, and manual safe-mode startup.

**Commit:** `add desktop and worker host topology`

## Slice `durable-job-model`

**Objective**

Add a durable job/outbox model with `JobId`, idempotency key, operation type, wallet/account scope, chain/network, payload hash, state, attempts, lease owner/expiry, next-attempt time, last error code, and audit timestamps. State transitions are append-only events plus a current projection. A lease expiry makes work available again; a completed idempotency key returns the previous result.

**Acceptance criteria**

- Duplicate commands with the same idempotency key cannot create two broadcasts.
- State transitions reject illegal jumps and preserve the reason code.
- Lease acquisition is atomic under SQLite concurrency.
- Retry policy distinguishes validation/revert (no retry), rate limit/provider outage (bounded retry), timeout/unknown broadcast (reconciliation), and process crash (lease recovery).
- No job payload contains a raw private key or seed.

**Files to read (context budget)**

- Persistence entities/migrations from forensic foundation.
- Result/error contract.
- Worker host composition root.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Jobs/**`.
- `1 - Core/MetaMask.Application/Jobs/**`.
- `2 - Adapter/MRQ.CryptoBot.Repository/**` job repositories/migrations.
- `5 - Worker/MRQ.CryptoBot.Worker/Queue/**`.
- `4 - Tests/**` concurrency and transition tests.

**Depends on**

`host-topology`.

**Complexity**

L.

**Owner**

Direct implementation with persistence review.

**Gates**

Parallel SQLite test, duplicate-delivery test, transition property tests, and migration review.

**Commit:** `add durable idempotent job model`

## Slice `worker-pipeline`

**Objective**

Implement isolated pipelines: market/contract discovery, quote collection, numerical strategy signal, optional AI advisory, risk approval, execution submission, receipt reconciliation, balance synchronization, and operations monitoring. Each stage communicates through typed jobs and persists its output. Live quote ingestion may be frequent; AI is advisory and lower priority; execution is always gated by a risk-approved command.

**Acceptance criteria**

- A quote cannot directly broadcast; it must produce a risk decision and an execution intent.
- A failed receipt reconciliation leaves the operation `Unknown` and schedules lookup, never marks success from a submitted hash alone.
- UI can observe progress and cancel a pending job without corrupting an in-flight signed transaction.
- Backpressure prevents quote polling from starving execution/reconciliation queues.
- AI latency, Ollama outage, malformed output, or model drift cannot delay or authorize execution; the configured fallback is deterministic strategy or no-trade.
- Each worker has metrics for latency, retries, queue depth, and terminal failures.

**Files to read (context budget)**

- All ports from EVM, Bitcoin, CEX, and strategy plans.
- Job model and queue implementation.
- Existing `Form1.cs` event handlers.

**Files to create/modify**

- `1 - Core/MetaMask.Application/Workers/**`.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/**`.
- `5 - Worker/MRQ.CryptoBot.Worker/Queue/**`.
- `0 - Client/MetaMask.Client/**` status/cancellation commands.
- `4 - Tests/**` pipeline tests.

**Depends on**

`durable-job-model`.

**Complexity**

L.

**Owner**

Direct implementation with production-safety review.

**Gates**

Fake-provider pipeline tests, cancellation/backpressure tests, metrics assertions, and no-live-broadcast integration test.

**Commit:** `run trading work through durable worker pipeline`

## Slice `chain-coordination`

**Objective**

Add resource coordinators: per-EVM-account nonce manager with pending transaction reconciliation; per-Bitcoin-wallet UTXO reservation and fee estimation coordination; per-CEX account request throttling and order idempotency. Use semaphores/leases only around the smallest critical section and persist the decision.

**Acceptance criteria**

- Two EVM jobs for one account cannot reuse a nonce; replacement/unknown nonce states reconcile from chain.
- Two Bitcoin jobs cannot spend the same reserved UTXO; abandoned reservations expire safely.
- CEX retries cannot duplicate an order when the exchange supports client order ids.
- Separate accounts and chains can progress concurrently within configured limits.
- A provider outage does not cause unbounded worker growth.

**Files to read (context budget)**

- EVM transaction and Bitcoin transaction ports.
- Job repository and lease implementation.
- Nethereum receipt/nonce APIs and Bitcoin Core/NBXplorer integration contracts.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Coordination/**`.
- `1 - Core/MetaMask.Application/Coordination/**`.
- `2 - Adapter/**` chain-specific coordinators.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/**`.
- `4 - Tests/**` concurrency tests.

**Depends on**

`worker-pipeline`.

**Complexity**

L.

**Owner**

Direct implementation with concurrency review.

**Gates**

Stress tests with deterministic fake clocks, crash injection around lease/submit boundaries, and manual review of retry policy.

**Commit:** `coordinate chain resources and idempotent submissions`

## Slice `shutdown-recovery`

**Objective**

Prove graceful shutdown and restart recovery. On shutdown, stop intake, drain safe read-only work, persist cancellation/lease state, and leave signed/broadcast operations for reconciliation. On startup, recover expired leases and scan unresolved transaction attempts before accepting new execution.

**Acceptance criteria**

- Kill/restart tests do not create duplicate execution for the same idempotency key.
- Unknown EVM hashes, Bitcoin txids, Lightning payments, and CEX order ids are reconciled before retry.
- Stuck leases generate an operator-visible alert and remain auditable.
- `CancellationToken` reaches provider calls and worker loops.
- Worker process exits within the configured shutdown timeout unless a non-cancellable OS/network call is explicitly documented.

**Files to read (context budget)**

- Worker host lifetime code.
- Job state machine and chain coordinators.
- Connector reconciliation contracts.

**Files to create/modify**

- `5 - Worker/MRQ.CryptoBot.Worker/Hosting/**`.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/**`.
- `1 - Core/MetaMask.Application/Jobs/**`.
- `4 - Tests/**` restart/crash harness.
- `.docs/operations/worker-recovery.md`.

**Depends on**

`chain-coordination`.

**Complexity**

L.

**Owner**

Direct implementation with production-safety review.

**Gates**

Crash/restart integration suite, graceful shutdown manual test, and operational runbook review.

**Commit:** `prove worker restart and reconciliation recovery`

## Technology decision

Use .NET Generic Host plus `BackgroundService` for lifecycle, bounded `Channel<T>` for in-process backpressure, and SQLite/EF Core for durable jobs and leases. This keeps the first deployment simple and testable on one Windows machine while preserving a port that can later map to RabbitMQ, Azure Service Bus, or another broker if a multi-machine requirement appears. Do not use a process-local queue as the source of truth.
