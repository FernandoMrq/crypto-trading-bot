---
name: Forensic baseline and modernization
overview: Restore a buildable, testable foundation and move the solution from the 2022 desktop-only shape to a supported .NET host model with bounded spot day trading, local AI advisory, and monitoring without implementing trading behavior yet.
isProject: false
todos:
  - id: baseline-inventory
    content: "Record the bounded audit baseline, preserve the tracked database, and make missing external dependencies explicit"
    status: in_progress
  - id: return-contract
    content: "Replace the unavailable MRQ.ReturnContent project reference with an internal result and error contract"
    status: pending
  - id: target-framework
    content: "Upgrade the solution to the supported .NET LTS target and align package versions"
    status: pending
  - id: dependency-direction
    content: "Restore ports-and-adapters dependency direction and split chain-specific contracts from application use cases"
    status: pending
  - id: persistence-migration
    content: "Create a safe SQLite migration path for the existing context database and new operational records"
    status: pending
  - id: foundation-gate
    content: "Prove clean restore, build, test discovery, migration validation, and desktop/worker startup"
    status: pending
---

## Phases

P1 — Establish an auditable, reproducible baseline.
P2 — Make the repository buildable and modern without changing trading semantics.
P3 — Introduce the durable application foundation required by later workstreams.
P4 — Demonstrate a green foundation across all runtime modes.

## Slices

| Phase | Slice | Objective | Owner | Gates | Depends on |
|---|---|---|---|---|---|
| P1 | `baseline-inventory` | Capture the current repository, refs, supported modes, and known blockers | direct | forensic review; secret scan; no production behavior changes | — |
| P2 | `return-contract` | Remove the unavailable external result-library boundary | direct | build; unit tests for success/failure/cancellation | `baseline-inventory` |
| P2 | `target-framework` | Move projects and packages to the selected supported LTS | direct | restore; build; package audit | `return-contract` |
| P3 | `dependency-direction` | Make application code depend on ports, not adapters or repositories | direct | architecture test; build; unit tests | `target-framework` |
| P3 | `persistence-migration` | Version the local database and add operational persistence primitives | direct | migration script; SQLite integration tests; backup/rollback check | `dependency-direction` |
| P4 | `foundation-gate` | Verify every project and runtime entry point before feature work | direct | clean CI restore/build/test; smoke startup; manual gate | `persistence-migration` |

## Slice `baseline-inventory`

**Objective**

Create the evidence record that all later plans use. The boundary is the `master` checkout at `7ef7ef3`, its visible `origin/master` and `origin/Nethereum` refs, all tracked source/config/test files, the tracked `context.db`, and the WinForms runtime. The future boundary includes a WinForms operator process plus one or more same-machine worker processes, a read-only operations/monitoring plane, and a local Ollama/Gemma advisory worker. The supported trading boundary is BSC/EVM, native Bitcoin, Lightning, CEX, and cross-chain routing; the day-trading scope is bounded spot strategies over an allowlisted liquid universe; mainnet execution is gated behind paper/testnet/regtest/sandbox proof. A distributed broker, browser-wallet-only custody, LLM-authorized signing, leveraged derivatives, and automatic arbitrary-contract execution are non-goals for the first delivery.

**Acceptance criteria**

- The plan records that all in-boundary flows were examined under the repository and dependency limitations; it does not claim all possible bugs were found.
- The missing `C:/@me/Library/MRQ.ReturnContent/MRQ.ReturnContent.csproj` reference, absent assets files, empty test project references, hardcoded RPC/API settings, raw private-key UI path, and tracked database are recorded as confirmed findings.
- No secret value is copied into the plan or any generated artifact.
- The current database is preserved until a schema/data migration plan has passed backup and restore checks.

**Files to read (context budget)**

- `MRQ.CryptoBot.sln` — project list and dependency graph.
- All `*.csproj` files — target frameworks, package versions, and project references.
- `0 - Client/MetaMask.Client/Form1.cs:67-146` — UI-to-use-case flow.
- `1 - Core/MetaMask.Business/OperationBusiness.cs:9-45` — swap and transfer flow.
- `2 - Adapter/MetaMask.Integration/Nethereum/PancakeSwapAdapter.cs:13-190` — EVM execution.
- `2 - Adapter/MetaMask.Integration/Moralis/MoralisTokenPriceAdapter.cs:12-205` — provider calls and shared state.
- `2 - Adapter/MRQ.CryptoBot.Repository/MRQ.CryptoBot.Repository/SQLiteContext.cs:1-40` — database wiring.
- `4 - Tests/PancakeSwapAdapterTest/PancakeSwapAdapterTest.cs` — current verification.
- `git log --all --graph --decorate --oneline -80` and path history for deleted/renamed automation and adapter files.

**Files to create/modify**

- `.docs/audits/2026-09-01-forensic-baseline.md`.
- No production file changes in this slice.

**Depends on**

None.

**Complexity**

S — evidence capture and safe documentation only.

**Owner**

Direct implementation by the primary agent.

**Gates**

Read-only repository audit, `git diff --check`, and secret redaction review.

**Commit:** `record forensic baseline and execution boundary`

## Slice `return-contract`

**Objective**

Introduce an internal, immutable result contract with typed error codes, correlation id, operation id, cancellation, and structured diagnostics. Migrate application and adapters away from the unavailable `MRQ.ReturnContent` project. Preserve user-visible messages through an adapter rather than keeping mutable shared `Returned` instances.

**Acceptance criteria**

- The solution restores without the external project path.
- Success, rejected validation, provider failure, timeout, cancellation, and on-chain revert are distinguishable.
- Sensitive values are excluded from errors, logs, serialization, and UI messages.
- Every public async method accepts `CancellationToken`.
- Unit tests cover each result category and ensure a failed call cannot reuse a previous call's object or state.

**Files to read (context budget)**

- `1 - Core/MetaMask.Domain/MRQ.CryptoBot.Domains.csproj`.
- `1 - Core/MetaMask.Shared/MRQ.CryptoBot.Shareds.csproj` and `Util.cs`.
- All usages of `MRQ.ReturnContent`, `ReturnedExtension`, `ReturnedState`, and `State` found by search.
- `1 - Core/MetaMask.Application/TokenPriceApplication.cs`.

**Files to create/modify**

- `1 - Core/MetaMask.Shared/Results/*`.
- `1 - Core/MetaMask.Domain/...` interfaces that currently expose `Returned`.
- `1 - Core/MetaMask.Application/*` and `1 - Core/MetaMask.Business/*` call paths.
- `2 - Adapter/MetaMask.Integration/*` implementations.
- `0 - Client/MetaMask.Client/*` result presentation boundary.
- `4 - Tests/PancakeSwapAdapterTest/*` focused unit tests.

**Depends on**

`baseline-inventory`.

**Complexity**

M — cross-layer contract migration; no trading behavior changes.

**Owner**

Direct implementation by the primary agent.

**Gates**

`dotnet restore`, `dotnet build`, focused unit tests, API review, and `production-safety-gate` for error/secret handling.

**Commit:** `replace unavailable result dependency with internal contract`

## Slice `target-framework`

**Objective**

Upgrade the target to the current supported .NET LTS selected in the implementation kickoff (recommended: `net10.0`; WinForms `net10.0-windows`) and update packages as a coherent lockstep change. Nethereum must be upgraded from 4.1.1 to the current compatible release; EF Core must match the target runtime; Bitcoin packages are introduced only in their dedicated workstream.

**Acceptance criteria**

- No project targets end-of-life .NET 6.
- Package versions are centrally auditable and have no known incompatible transitive runtime mix.
- Windows desktop build remains explicit and worker projects are cross-platform where possible.
- Restore is reproducible from a clean checkout using the documented SDK version.

**Files to read (context budget)**

- All `*.csproj` files.
- `MRQ.CryptoBot.sln`.
- Official .NET support policy and Nethereum/EF Core release documentation linked in the plan references.

**Files to create/modify**

- `global.json`.
- `Directory.Build.props` and `Directory.Packages.props`.
- All `*.csproj` files.
- `README.md` and `.docs/operations/toolchain.md`.

**Depends on**

`return-contract`.

**Complexity**

M — package/runtime compatibility and Windows build concerns.

**Owner**

Direct implementation by the primary agent.

**Gates**

Clean restore/build on the documented SDK, package vulnerability scan, and test discovery.

**Commit:** `upgrade solution to supported dotnet lts`

## Slice `dependency-direction`

**Objective**

Move blockchain, market-data, storage, clock, signer, and queue boundaries into Domain/Application ports. The Business project must stop referencing `MetaMask.Integration` and the repository project directly. Keep implementations in adapter/infra projects and make WinForms and workers depend on the application host composition root.

**Acceptance criteria**

- Architecture tests fail if Core references Adapter or Infrastructure namespaces/projects.
- EVM, Bitcoin, CEX, and cross-chain ports do not share EVM-specific DTOs.
- Use cases receive interfaces for quotes, balances, signing, submission, receipts, persistence, time, and risk decisions.
- Existing manual price/balance/swap flows are represented as application commands, even if execution remains disabled until later plans.

**Files to read (context budget)**

- `1 - Core/MetaMask.Business/MRQ.CryptoBot.Business.csproj`.
- `1 - Core/MetaMask.Domain/Adapter/**` and `Business/**` interfaces.
- `0 - Client/MetaMask.Client/Configurations/ConfigureServices.cs`.
- `3 - Infra/MetaMask.Infra/ExternalServiceExtension.cs`.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Ports/**`.
- `1 - Core/MetaMask.Application/UseCases/**`.
- `1 - Core/MetaMask.Business/**`.
- `2 - Adapter/**` registrations and implementations.
- `3 - Infra/**` composition root.
- `4 - Tests/**` architecture and unit tests.

**Depends on**

`target-framework`.

**Complexity**

L — public contracts and broad dependency cleanup.

**Owner**

Direct implementation by the primary agent with architecture review.

**Gates**

Build, architecture test, unit tests, public contract review, and `production-safety-gate`.

**Commit:** `restore application ports and adapter boundaries`

## Slice `persistence-migration`

**Objective**

Make EF Core configuration-driven and add migrations for wallets, encrypted secrets, networks, tokens, connector registrations, quotes, orders, transaction attempts, receipts, jobs, leases, and audit events. Treat the tracked `context.db` as an input requiring backup, schema inspection, and explicit migration; never silently delete it.

**Acceptance criteria**

- SQLite path comes from configuration and is placed under an application-data directory, not the working directory.
- Migrations run explicitly and are idempotent; startup does not mutate production data without a configured policy.
- Secret columns contain only encrypted envelopes, never raw private keys or provider keys.
- Order and transaction records support idempotency, state transitions, retries, and reconciliation.
- A backup/restore test proves the old database can be copied before migration and restored on failure.

**Files to read (context budget)**

- `0 - Client/MetaMask.Client/appsettings.json`.
- `2 - Adapter/MRQ.CryptoBot.Repository/MRQ.CryptoBot.Repository/SQLiteContext.cs`.
- Existing `Migrations/**` and model snapshot.
- `2 - Adapter/MRQ.CryptoBot.Repository/MRQ.CryptoBot.Repository/Service/EntityService.cs`.
- `0 - Client/MetaMask.Client/context.db` metadata only; do not print secret fields.

**Files to create/modify**

- `2 - Adapter/MRQ.CryptoBot.Repository/**` entities, context, migrations, repositories.
- `1 - Core/MetaMask.Domain/Entities/**` and persistence ports.
- `scripts/Backup-LocalDatabase.ps1`.
- `.docs/operations/database-migrations.md`.
- `4 - Tests/**` SQLite integration tests.

**Depends on**

`dependency-direction`.

**Complexity**

L — schema, migration, data retention, and secret handling.

**Owner**

Direct implementation with database/security review.

**Gates**

Migration SQL review, backup/restore test, SQLite integration tests, and `production-safety-gate`.

**Commit:** `make sqlite persistence configuration driven`

## Slice `foundation-gate`

**Objective**

Prove the foundation is green before adding real connectors. Add a WinForms composition-root smoke test, a worker-host smoke test, and a clean CI command set. This slice cannot hide missing external projects, skipped tests, or warnings caused by unsupported runtime assumptions.

**Acceptance criteria**

- `dotnet restore MRQ.CryptoBot.sln --locked-mode` succeeds from a clean checkout.
- `dotnet build MRQ.CryptoBot.sln --configuration Release --no-restore` succeeds.
- `dotnet test MRQ.CryptoBot.sln --configuration Release --no-restore --collect:"XPlat Code Coverage"` discovers non-placeholder tests and passes.
- WinForms and worker hosts start with safe execution disabled by default.
- The gate reports external-network tests separately and never treats a live-money test as CI verification.

**Files to read (context budget)**

- `README.md` build/run sections.
- `MRQ.CryptoBot.sln`.
- All new host/bootstrap files from prior slices.
- `.docs/operations/toolchain.md`.

**Files to create/modify**

- `.github/workflows/build.yml` or the repository's selected CI configuration.
- `scripts/Verify-Build.ps1`.
- `README.md`.
- `4 - Tests/**` smoke tests.

**Depends on**

`persistence-migration`.

**Complexity**

M — verification and host wiring.

**Owner**

Direct implementation by the primary agent.

**Gates**

All commands above, coverage threshold for touched code, and manual startup check in Development mode.

**Commit:** `establish clean build and host verification gate`

## Forensic evidence ledger

| ID | Status | Severity | Evidence | Impact / gap |
|---|---|---:|---|---|
| F-001 | CONFIRMED | Blocker | Solution and Domain/Integration projects reference missing `C:/@me/Library/MRQ.ReturnContent/MRQ.ReturnContent.csproj`; build fails with MSB3202. | No clean restore/build; exact intended external API must be replaced or restored. This plan chooses replacement with an internal contract. |
| F-002 | CONFIRMED | Blocker | `dotnet build --no-restore` reports missing `project.assets.json` in repository, tests, infra, and shared projects. | Baseline cannot verify package graph until restore is reproducible. |
| F-003 | CONFIRMED | Critical | `Form1.cs:67-76` copies a raw private key from UI input into `WalletDto`; `Wallet.cs` and `WalletDto.cs` expose it as a string. | Key can leak through UI, memory, logs, dumps, and persistence. Secure custody plan is mandatory. |
| F-004 | CONFIRMED | Critical | Moralis adapter initializes provider keys in source and rotates a mutable static list; `context.db` is tracked. | Credential exposure and non-isolated concurrency. Values are intentionally not reproduced here. |
| F-005 | CONFIRMED | Critical | Pancake adapter uses hardcoded RPC/router settings, assumes 18 decimals, has no approval flow, and transfer sends native value while accepting a token. | Incorrect or unsafe transactions are possible. EVM execution plan must replace the flow. |
| F-006 | CONFIRMED | High | Adapter and Business objects retain mutable `_returned` state across async calls; HTTP status is not checked before deserialization. | Cross-request corruption and false-positive success. Result/HTTP boundaries must be isolated. |
| F-007 | CONFIRMED | High | `OperationBusiness.cs:39` compares origin balance with itself and casts failed/null provider results without checking status. | Trade sizing can be wrong or throw. Quote/risk plan must own validation. |
| F-008 | CONFIRMED | High | Test project has no production project references and only an empty `[Fact] Test1`. | Existing green status would not prove any trading behavior. |
| F-009 | HISTORICAL | Medium | Git history shows earlier orchestration, database, automation placeholders, and deleted/renamed adapters, but no Bitcoin implementation. | Reuse candidates are limited to domain intent and migration history; no historical BTC implementation was found. |
| F-010 | DOCUMENTED | Medium | README/Arquitetura describe automation, encryption, BSCScan, Telegram, and orchestration beyond current code. | Documentation is aspirational and must be reconciled with executable scope. |
| F-011 | HYPOTHESIS | High | Tracked `context.db` contains schema entities named `Wallet` and `ApiKey`, potentially with sensitive data. | Confirm schema and redact/migrate without printing values before changing or removing the file. |
| F-012 | UNKNOWN | High | Worker identity, backup/recovery policy, and whether one Windows user will run UI and workers are not encoded in the repository. | This plan assumes same-user local workers first and requires an explicit service-account gate before production deployment. |
| F-013 | CONFIRMED | Medium | Local environment inspection found Ollama `0.32.9` with `gemma3:4b`, approximately 4.3B parameters and Q4_K_M quantization. | Model availability is confirmed, but trading quality, p50/p95 latency, GPU/CPU profile, and drift behavior are not; those become AI-plan gates. |
| F-014 | DOCUMENTED | High | User clarified that execution is intended for automated spot day trading with AI/LLM assistance and a funded MetaMask account. | The funded account must be a dedicated bot hot wallet with capped exposure; the primary wallet is outside unattended execution. |
| F-015 | UNKNOWN | High | No repository data pipeline, labeled strategy dataset, LLM evaluation set, or model registry exists. | Historical data sources, licensing, leakage controls, and validation methodology are covered by the AI/ML plan. |

## Official references

- [.NET support policy](https://dotnet.microsoft.com/en-us/platform/support/policy) — target a supported LTS, not .NET 6.
- [EF Core what is new](https://learn.microsoft.com/en-us/ef/core/what-is-new/) — align EF Core with the selected runtime.
- [Nethereum documentation](https://docs.nethereum.com/) — EVM RPC, contracts, transactions, receipts, and signing APIs.
- [.NET cryptography overview](https://learn.microsoft.com/en-us/dotnet/standard/security/cryptographic-services) — use platform cryptography primitives and authenticated encryption designs.

## Summary

Investigated boundary: the `master` checkout at `7ef7ef3`, all visible local/remote refs and relevant history, tracked source/configuration/tests, the WinForms entry point, current Moralis/PancakeSwap/SQLite flows, the tracked `context.db` metadata boundary, and the proposed same-machine WinForms plus worker runtime. External library assumptions were checked against current official documentation for supported .NET, EF Core, Nethereum, and .NET cryptography. All in-boundary flows were examined under the missing external-project and missing-restore limitations; this is not a claim that every possible bug was found.

Confirmed defects/blockers: the solution cannot restore/build because `MRQ.ReturnContent` is missing and assets files are absent; tests do not reference production projects; private keys and provider credentials enter unsafe mutable paths; RPC/router/provider settings are hardcoded; asynchronous adapters share mutable result state; HTTP failures are treated as successful data; token decimals, approvals, native-vs-ERC-20 transfer semantics, balance validation, receipt status, and transaction sizing are unsafe or incomplete; the SQLite configuration is not wired and the tracked database requires a controlled migration. The newly bounded product intent is automated spot day trading with deterministic/numerical signals, optional local Gemma/Ollama advisory, a separate monitoring/control plane, and a dedicated funded bot wallet with capped exposure. Historical reuse candidates: the repository contains useful domain intent, migration history, router configuration concepts, and earlier orchestration placeholders, but no historical Bitcoin/Lightning implementation was found. Unresolved questions: recovery after Windows profile loss, worker identity if deployed as a service, production backup/restore ownership, provider quotas, exact strategy parameters, model latency/quality, dataset licensing, and the minimum evidence period for live enablement. The plans below turn those unknowns into explicit gates rather than silently assuming them.
