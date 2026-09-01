---
name: Observability operator UI and release
overview: Make operations a monitoring and control plane, inspectable, updateable, and safe through structured telemetry, transaction evidence, AI/model health, operator controls, packaging, and production release gates.
isProject: false
todos:
  - id: telemetry-contract
    content: "Define structured logs, metrics, traces, audit events, correlation ids, and redaction rules"
    status: in_progress
  - id: operator-console
    content: "Replace direct form calls with an operator console for jobs, quotes, risk decisions, AI/model health, and transaction states"
    status: pending
  - id: alerts-runbooks
    content: "Add health checks, alerts, recovery runbooks, and emergency stop behavior"
    status: pending
  - id: packaging
    content: "Package WinForms and workers with configuration validation and safe update behavior"
    status: pending
  - id: release-gate
    content: "Verify release artifacts, supportability, security scans, and production enablement checklist"
    status: pending
---

## Phases

P1 — Make every operation traceable without leaking secrets.
P2 — Give operators safe control and evidence.
P3 — Make failure recovery visible.
P4 — Package and release only verified artifacts.

## Slices

| Phase | Slice | Objective | Owner | Gates | Depends on |
|---|---|---|---|---|---|
| P1 | `telemetry-contract` | Add observability boundaries | direct | redaction tests; schema review | forensic `return-contract` + worker `durable-job-model` |
| P2 | `operator-console` | Build safe UI operations | direct | UI/manual tests | `telemetry-contract` + worker `worker-pipeline` |
| P3 | `alerts-runbooks` | Make incidents actionable | direct | fault injection; runbook drill | `operator-console` + worker `shutdown-recovery` |
| P4 | `packaging` | Produce repeatable desktop/worker artifacts | direct | clean install/upgrade | `alerts-runbooks` + secure `environment-gate` |
| P4 | `release-gate` | Close production readiness | direct | security/release checklist | `packaging` + strategy `strategy-gate` |

## Slice `telemetry-contract`

**Objective**

Introduce structured logging, metrics, traces, and audit events across quote, risk, signing, submission, reconciliation, provider, queue, and UI boundaries. Use operation id, job id, idempotency key fingerprint, chain/network, connector, and wallet fingerprint; never use private keys, seeds, API keys, full calldata when it may contain sensitive data, or full request headers.

**Acceptance criteria**

- Every operation can be traced from UI command to queue job to provider call to receipt/fill.
- Logs distinguish `prepared`, `signed`, `submitted`, `confirmed`, `failed`, and `unknown` without implying success from submission.
- Metrics include queue depth, worker age, quote latency, provider errors, risk rejects, gas, slippage, confirmation time, and unknown operations.
- Audit events are append-only and include actor, reason, policy version, and before/after state.
- Redaction tests scan serialized logs/telemetry for secret markers and raw credential patterns.

**Files to read (context budget)**

- Internal result contract.
- Job/audit entities.
- Existing `ReturnedExtension` logging calls.
- Secret custody policy.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Observability/**`.
- `1 - Core/MetaMask.Application/Observability/**`.
- `3 - Infra/MetaMask.Infra/Telemetry/**`.
- `2 - Adapter/**` provider instrumentation.
- `4 - Tests/Observability/**`.

**Depends on**

Forensic `return-contract`; worker `durable-job-model`.

**Complexity**

M.

**Owner**

Direct implementation with security review.

**Gates**

Redaction tests, telemetry schema review, and no-sensitive-data manual inspection.

**Commit:** `add structured operation telemetry and audit events`

## Slice `operator-console`

**Objective**

Replace direct async void button calls with application commands, cancellation, validation feedback, and an operation/monitoring view. Operations owns health, queue depth, stale jobs, provider state, model health, risk rejects, emergency stop, and transaction evidence; it does not decide trading strategy. Operators can inspect quote, route, risk decision, preflight evidence, transaction hash/txid/order id, confirmation, and errors. Wallet setup uses wallet id and address verification rather than a private-key textbox.

**Acceptance criteria**

- UI handlers are exception-safe, cancellation-aware, and do not block the UI thread.
- Invalid addresses/amounts/network/recipient are rejected before queue submission.
- The UI clearly shows Development/Test/Paper/Production and broadcast enabled/disabled.
- The UI clearly shows AI mode (`Disabled`, `Shadow`, `Paper`, `AdvisoryOnly`) and Ollama/model health without implying that AI approval authorizes a trade.
- Operator approval is required for first use of a contract/token/route in Production.
- No UI log displays secret contents or unredacted provider responses.

**Files to read (context budget)**

- `0 - Client/MetaMask.Client/Form1.cs:67-151`.
- `Form1.Designer.cs` controls and current configuration form.
- Application command/query ports.
- Telemetry and worker status contracts.

**Files to create/modify**

- `0 - Client/MetaMask.Client/Form1.cs` and views/view-models.
- `0 - Client/MetaMask.Client/UIs/**`.
- `1 - Core/MetaMask.Application/Operator/**`.
- `4 - Tests/Client/**` where UI automation is practical.

**Depends on**

`telemetry-contract`; worker `worker-pipeline`.

**Complexity**

L.

**Owner**

Direct implementation with UI/security review.

**Gates**

WinForms manual smoke, exception/cancellation tests, and `production-safety-gate`.

**Commit:** `move operator actions onto safe application commands`

## Slice `alerts-runbooks`

**Objective**

Add health checks and operational runbooks for provider outage, stuck queue, unknown transaction, nonce conflict, UTXO conflict, Lightning unknown payment, CEX order uncertainty, database lock, DPAPI profile loss, and emergency stop. Alerts must identify the next safe action and avoid auto-retrying irreversible operations blindly.

**Acceptance criteria**

- Health status separates liveness, readiness, provider health, database health, and execution permission.
- Unknown operations page the operator and enter reconciliation, not an automatic duplicate retry.
- Emergency stop blocks new signing/broadcast and remains active after restart until explicitly cleared.
- Model/feature drift, Ollama outage, invalid advisory schema, abnormal advisory distribution, and AI latency budget breach generate monitoring events and fall back to no-AI mode.
- Runbooks include evidence collection, safe commands, backup paths, and rollback boundaries.

**Files to read (context budget)**

- Worker shutdown/recovery plan.
- Connector reconciliation implementations.
- Secret backup/recovery documentation.
- Telemetry contract.

**Files to create/modify**

- `3 - Infra/MetaMask.Infra/Health/**`.
- `5 - Worker/MRQ.CryptoBot.Worker/Health/**`.
- `.docs/operations/runbooks/**`.
- `0 - Client/MetaMask.Client/UIs/**` emergency controls.
- `4 - Tests/Operations/**` fault injection.

**Depends on**

`operator-console`; worker `shutdown-recovery`.

**Complexity**

L.

**Owner**

Direct implementation with production-safety review.

**Gates**

Fault injection, restart drill, alert review, and destructive-action confirmation.

**Commit:** `add operational health alerts and recovery runbooks`

## Slice `packaging`

**Objective**

Produce repeatable Windows desktop and worker artifacts. The package must include the selected .NET runtime strategy, migration command, environment initialization/validation scripts, safe defaults, version/build metadata, and an upgrade path that preserves the database and encrypted secret envelopes.

**Acceptance criteria**

- Clean machine setup can install, initialize Development, validate config, and start in safe mode.
- Upgrade preserves database and secrets; downgrade is explicitly unsupported or has a tested migration path.
- Worker identity/DPAPI scope is checked during install and documented.
- Artifacts are signed or their integrity verification is documented; build provenance is recorded.
- Production packaging cannot ship with sample credentials or execution enabled accidentally.

**Files to read (context budget)**

- Project/worker hosts and environment scripts.
- Database migration and secret recovery docs.
- CI build workflow.

**Files to create/modify**

- `scripts/Publish-Release.ps1`.
- `scripts/Install-Worker.ps1` or selected service/task setup.
- `.github/workflows/release.yml`.
- `README.md` and `.docs/operations/installation.md`.
- Packaging project files.

**Depends on**

`alerts-runbooks`; secure `environment-gate`.

**Complexity**

L.

**Owner**

Direct implementation with release/security review.

**Gates**

Clean-machine install/upgrade, artifact scan, database/secret preservation test, and manual production configuration review.

**Commit:** `package desktop and worker release artifacts`

## Slice `release-gate`

**Objective**

Close the release with a single evidence checklist: build/test/coverage, dependency vulnerabilities, secret scan, environment validation, migration backup, testnet/regtest/sandbox results, worker recovery, strategy paper evidence, contract approvals, and emergency stop. Update README so it no longer presents planned capabilities as implemented.

**Acceptance criteria**

- Release is blocked if any required check is missing or a live operation remains `Unknown`.
- The final checklist records exact artifact version, source commit, SDK/package lock, test reports, and operator approver.
- Mainnet execution is opt-in and disabled in new installs.
- Release notes include known limitations and recovery instructions.

**Files to read (context budget)**

- All sibling plan gates and runbooks.
- `README.md`, `Arquitetura.md`, `Dependencia.md`.
- CI/release scripts.

**Files to create/modify**

- `.docs/operations/production-readiness.md`.
- `.github/workflows/release.yml`.
- `README.md`, `Arquitetura.md`, `Dependencia.md`.
- `scripts/Verify-Release.ps1`.

**Depends on**

`packaging`; strategy `strategy-gate`.

**Complexity**

M.

**Owner**

Direct implementation with release and production-safety review.

**Gates**

Full release checklist, clean-room verification, security review, and explicit operator approval.

**Commit:** `close production readiness and release evidence`

## Release policy

No plan slice authorizes real-money execution by itself. A green build proves software integrity, not profitability or safety against every market/contract/provider failure. Live enablement requires the secure environment gate, EVM/Bitcoin/CEX test evidence, strategy gate, recovery drill, and operator approval.

Operations boundary: monitoring and control are part of Operations; strategy, AI inference, risk approval, execution, and reconciliation remain separate application capabilities. Operations may observe and stop the system, but it must not silently alter strategy limits or authorize a broadcast.
