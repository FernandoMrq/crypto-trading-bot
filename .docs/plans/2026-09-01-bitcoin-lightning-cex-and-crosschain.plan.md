---
name: Bitcoin Lightning CEX and cross-chain connectors
overview: Add separate UTXO, Lightning, CEX, and cross-chain ports so Bitcoin execution never reuses EVM account/allowance assumptions.
isProject: false
todos:
  - id: bitcoin-domain
    content: "Define Bitcoin network, address, UTXO, fee, transaction, PSBT, and confirmation contracts"
    status: in_progress
  - id: bitcoin-node-index
    content: "Implement Bitcoin Core RPC and NBXplorer-backed wallet/index synchronization"
    status: pending
  - id: bitcoin-sign-submit
    content: "Implement validated PSBT signing, broadcast, fee policy, and confirmation reconciliation"
    status: pending
  - id: lightning
    content: "Add Lightning payment/invoice adapter with settlement and retry semantics"
    status: pending
  - id: cex
    content: "Add CEX market/order connectors with permission and idempotency controls"
    status: pending
  - id: crosschain
    content: "Add cross-chain route orchestration with explicit Bitcoin and EVM signer boundaries"
    status: pending
  - id: multichain-gate
    content: "Verify testnet/regtest/sandbox scenarios and reconcile all external operation types"
    status: pending
---

## Phases

P1 — Establish Bitcoin-native domain semantics.
P2 — Track UTXOs, sign PSBTs, and reconcile confirmations.
P3 — Add Lightning and CEX operations without mixing custody models.
P4 — Orchestrate cross-chain routes and verify all modes.

## Slices

| Phase | Slice | Objective | Owner | Gates | Depends on |
|---|---|---|---|---|---|
| P1 | `bitcoin-domain` | Define UTXO/PSBT contracts | direct | domain tests; architecture review | forensic `dependency-direction` + secure `custody-boundary` |
| P2 | `bitcoin-node-index` | Synchronize wallet state | direct | regtest integration | `bitcoin-domain` |
| P2 | `bitcoin-sign-submit` | Sign and submit safely | direct | PSBT tests; regtest | `bitcoin-node-index` |
| P3 | `lightning` | Add Lightning settlement | direct | simulator/test node | `bitcoin-sign-submit` |
| P3 | `cex` | Add sandbox CEX execution | direct | recorded API/sandbox | `bitcoin-domain` + worker `durable-job-model` |
| P4 | `crosschain` | Compose routes across signers | direct | route state-machine tests | `bitcoin-sign-submit` + EVM `aggregator-routing` |
| P4 | `multichain-gate` | Verify all operation types | direct | regtest/testnet/sandbox checklist | `lightning` + `cex` + `crosschain` |

## Slice `bitcoin-domain`

**Objective**

Create Bitcoin-native contracts: network, derivation policy, address/script, UTXO, fee rate, coin selection, PSBT, signed transaction, txid, confirmation depth, and wallet balance. Do not expose EVM `BigInteger`/router/path/allowance fields in Bitcoin types.

**Acceptance criteria**

- Network mismatch, invalid address, dust, fee over policy, duplicate outpoint, and insufficient confirmed balance fail before signing.
- PSBT inputs/outputs, amounts, change address, locktime, fee, and network are inspectable and hashable before the signer runs.
- Confirmation policy is configurable per operation and persisted.
- The signer receives a validated PSBT and returns a signed PSBT/transaction, never an arbitrary transaction blob from an untrusted provider.

**Files to read (context budget)**

- `1 - Core/MetaMask.Domain/Entities/Wallet.cs` and existing token DTOs.
- Secure signer/secret-store interfaces.
- [Bitcoin Core PSBT documentation](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md).
- [NBitcoin repository](https://github.com/MetacoSA/NBitcoin).

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Trading/Bitcoin/**`.
- `1 - Core/MetaMask.Domain/Ports/Bitcoin/**`.
- `1 - Core/MetaMask.Shared/Amounts/**`.
- `4 - Tests/Bitcoin/**` domain tests.

**Depends on**

Forensic `dependency-direction`; secure `custody-boundary`.

**Complexity**

L.

**Owner**

Direct implementation with security review.

**Gates**

NBitcoin serialization/golden tests, network mismatch tests, and production-safety review.

**Commit:** `define bitcoin native transaction contracts`

## Slice `bitcoin-node-index`

**Objective**

Integrate Bitcoin Core RPC for node operations and NBXplorer for wallet/address/UTXO indexing where appropriate. Keep node RPC and indexer ports separate so the application can reconcile independently. Support regtest first, then testnet and mainnet behind environment gates.

**Acceptance criteria**

- Wallet sync is incremental and records tip height/hash, scan cursor, and provider provenance.
- UTXO states include confirmed, immature, reserved, spent, and unknown.
- Reorg detection invalidates affected projections and re-runs reconciliation.
- RPC authentication comes from the secure environment store and never appears in URLs/logs.

**Files to read (context budget)**

- Bitcoin domain ports.
- Worker queue/job model.
- [Bitcoin Core RPC reference](https://developer.bitcoin.org/reference/rpc/).
- [NBXplorer repository](https://github.com/btcpayserver/NBXplorer).

**Files to create/modify**

- `2 - Adapter/MetaMask.Integration/Bitcoin/CoreRpc/**`.
- `2 - Adapter/MetaMask.Integration/Bitcoin/Nbxplorer/**`.
- `3 - Infra/MetaMask.Infra/Http/**` or RPC clients.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/BitcoinSyncWorker.cs`.
- `4 - Tests/Bitcoin/**` regtest integration.

**Depends on**

`bitcoin-domain`.

**Complexity**

L.

**Owner**

Direct implementation.

**Gates**

Regtest node tests, reorg/scan cursor tests, auth redaction, and no-mainnet endpoint test.

**Commit:** `sync bitcoin wallet state from node and indexer`

## Slice `bitcoin-sign-submit`

**Objective**

Build PSBT validation, coin selection, fee estimation, signing through the secure Bitcoin signer, broadcast, and confirmation reconciliation. Persist the exact pre-sign intent and final txid. Never retry a broadcast merely because the RPC response timed out; reconcile by txid/transaction content first.

**Acceptance criteria**

- Output addresses/amounts and change are compared to the approved intent before signing.
- Fee policy prevents accidental fee overpayment and dust outputs.
- Broadcast rejection, timeout, mempool conflict, replacement, and confirmation are distinct states.
- Same idempotency key cannot spend the same UTXO twice.
- Regtest tests mine blocks and prove confirmation transitions.

**Files to read (context budget)**

- Bitcoin domain and sync adapters.
- Secret store and job coordinator.
- [Bitcoin Core send/raw transaction RPC docs](https://developer.bitcoin.org/reference/rpc/).

**Files to create/modify**

- `1 - Core/MetaMask.Application/Bitcoin/**`.
- `2 - Adapter/MetaMask.Integration/Bitcoin/**` signer/submission.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/BitcoinExecutionWorker.cs`.
- `4 - Tests/Bitcoin/**` PSBT/regtest tests.

**Depends on**

`bitcoin-node-index`.

**Complexity**

L.

**Owner**

Direct implementation with production-safety review.

**Gates**

PSBT golden tests, regtest broadcast/confirm tests, crash/retry test, and manual fee/recipient review.

**Commit:** `add validated bitcoin psbt execution`

## Slice `lightning`

**Objective**

Add a Lightning payment port using BTCPayServer.Lightning abstraction, supporting invoice creation/payment lookup and provider adapters for the selected node implementations. Treat Lightning payment ids/preimages/status as separate from on-chain txids and make settlement/retry rules explicit.

**Acceptance criteria**

- Invoice amount, expiry, destination, and description hash are validated before payment.
- Payment states distinguish pending, succeeded, failed, expired, and unknown.
- A timeout does not cause a duplicate payment without provider status lookup.
- Secrets/macaroons are in the secure store with least-privilege scopes.
- Tests use a simulator or test node, never production funds.

**Files to read (context budget)**

- Bitcoin job/reconciliation ports.
- [BTCPayServer.Lightning](https://github.com/btcpayserver/BTCPayServer.Lightning).
- Secure environment policy.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Trading/Lightning/**`.
- `2 - Adapter/MetaMask.Integration/Lightning/**`.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/LightningWorker.cs`.
- `4 - Tests/Lightning/**`.

**Depends on**

`bitcoin-sign-submit`.

**Complexity**

M.

**Owner**

Direct implementation.

**Gates**

Provider simulator/test-node tests, idempotency tests, and secret-scope review.

**Commit:** `add lightning payment connector`

## Slice `cex`

**Objective**

Add a CEX adapter boundary using CCXT-compatible semantics for market data and order management, while keeping exchange-specific capabilities explicit. Default API permissions are read/trade only; withdrawals and address management are outside the first execution scope. Use exchange client order ids and reconcile open orders/fills after timeouts.

**Acceptance criteria**

- Symbol precision, min quantity/notional, fees, order type, time-in-force, and sandbox/live mode are validated.
- API keys are not logged and withdrawal permission is rejected by policy.
- An order timeout triggers lookup by client id before retry.
- Rate limits and exchange errors map to typed retry classes.
- Tests use recorded fixtures and an exchange sandbox where available.

**Files to read (context budget)**

- Worker job/idempotency contracts.
- Secure custody policy.
- [CCXT repository/manual](https://github.com/ccxt/ccxt/wiki/Manual).

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Trading/Cex/**`.
- `2 - Adapter/MetaMask.Integration/Cex/**`.
- `3 - Infra/MetaMask.Infra/Http/**` throttling/resilience.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/CexExecutionWorker.cs`.
- `4 - Tests/Cex/**` fixtures/sandbox tests.

**Depends on**

Forensic `bitcoin-domain`; worker `durable-job-model`.

**Complexity**

L.

**Owner**

Direct implementation with security review.

**Gates**

Fixture tests, sandbox smoke, permission policy tests, and provider terms/quota review.

**Commit:** `add idempotent cex trading connector`

## Slice `crosschain`

**Objective**

Orchestrate cross-chain routes from LI.FI or another approved route provider as a state machine with independent EVM and Bitcoin steps. Each step has its own intent, signer, tx/payment id, confirmation policy, timeout, and compensation/manual-recovery path. Do not mutate returned PSBTs or calldata without rebuilding and revalidating the intent.

**Acceptance criteria**

- Route status distinguishes quote, approval, source submitted, source confirmed, destination pending, completed, failed, and manual recovery.
- Provider route changes after approval invalidate the operation.
- EVM calldata and Bitcoin PSBT are validated by their own preflight pipelines.
- Bridge/route fees, expiry, slippage, and destination recipient are shown before approval.
- A partial failure creates an operator action with all evidence and no blind retry.

**Files to read (context budget)**

- EVM aggregator and Bitcoin execution contracts.
- Worker job/reconciliation model.
- [LI.FI architecture/API](https://docs.li.fi/api-reference/openapi-spec).

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Trading/CrossChain/**`.
- `1 - Core/MetaMask.Application/CrossChain/**`.
- `2 - Adapter/MetaMask.Integration/Routing/LiFi/**`.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/CrossChainWorker.cs`.
- `0 - Client/MetaMask.Client/UIs/**` route review.
- `4 - Tests/CrossChain/**` state-machine tests.

**Depends on**

Bitcoin `bitcoin-sign-submit`; EVM `aggregator-routing`.

**Complexity**

XL — asynchronous partial completion and multiple custody models.

**Owner**

Direct implementation with architecture/security review.

**Gates**

State-machine/property tests, recorded API fixtures, no-live-funds integration, and production-safety gate.

**Commit:** `orchestrate validated cross chain routes`

## Slice `multichain-gate`

**Objective**

Run regtest, BSC testnet, Lightning simulator/test node, and CEX sandbox/fixture checks through the same durable worker/reconciliation path. Mainnet is a configuration option but remains off until the separate production checklist is approved.

**Acceptance criteria**

- All external operation types have a durable idempotency key and reconciliation strategy.
- Test reports show no EVM types used in Bitcoin logic and no raw secret in artifacts.
- Provider unavailability and partial cross-chain failure produce deterministic operator-visible outcomes.
- Environment generation produces the required test profiles without real credentials.

**Files to read (context budget)**

- All connector and worker slices.
- Environment and custody plans.
- `.docs/operations/*` runbooks.

**Files to create/modify**

- `4 - Tests/Integration/**`.
- `scripts/Run-MultichainSmoke.ps1`.
- `.docs/operations/multichain-test-matrix.md`.

**Depends on**

`lightning`, `cex`, `crosschain`.

**Complexity**

L.

**Owner**

Direct implementation.

**Gates**

All offline tests, selected external sandbox/testnet checks, coverage, and manual production gate.

**Commit:** `verify multichain connector recovery paths`

## Technology rationale

Use NBitcoin for Bitcoin-native parsing, address/script/transaction/PSBT operations; Bitcoin Core RPC for node truth; NBXplorer for wallet/UTXO indexing; BTCPayServer.Lightning for a provider abstraction; CCXT-compatible adapters for CEXs; and LI.FI REST for cross-chain route orchestration. These are separate ports because UTXO/PSBT, Lightning payment, exchange order, and EVM calldata have different failure and signing semantics.
