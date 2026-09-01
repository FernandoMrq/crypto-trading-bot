---
name: EVM execution routing and dynamic contracts
overview: Replace the unsafe PancakeSwap-only path with validated EVM quotes, approvals, simulation, typed connector adapters, and controlled runtime discovery of new contracts.
isProject: false
todos:
  - id: evm-models
    content: "Define chain, token, quote, calldata, allowance, simulation, and receipt contracts with integer-safe amounts"
    status: in_progress
  - id: direct-pancake
    content: "Implement safe PancakeSwap V2 and V3 quote and execution adapters"
    status: pending
  - id: aggregator-routing
    content: "Add 0x and optional LI.FI EVM routing behind the same quote and execution ports"
    status: pending
  - id: dynamic-registry
    content: "Add runtime connector and contract metadata discovery with explicit trust, selector, token-universe, and day-trading policies"
    status: pending
  - id: preflight-and-mev
    content: "Add allowance, simulation, token risk, slippage, gas, and MEV preflight gates"
    status: pending
  - id: evm-integration-gate
    content: "Prove EVM behavior with deterministic tests and safe testnet/manual execution checks"
    status: pending
---

## Phases

P1 — Make EVM values and transaction intents correct.
P2 — Support direct DEX and aggregator routes.
P3 — Admit new contracts through policy, not arbitrary execution.
P4 — Verify execution and failure behavior on test networks.

## Slices

| Phase | Slice | Objective | Owner | Gates | Depends on |
|---|---|---|---|---|---|
| P1 | `evm-models` | Introduce integer-safe EVM contracts | direct | domain tests; architecture review | forensic `dependency-direction` + secure `custody-boundary` |
| P2 | `direct-pancake` | Implement Pancake V2/V3 safely | direct | calldata/golden tests; testnet gate | `evm-models` |
| P2 | `aggregator-routing` | Add 0x/LI.FI quote routes | direct | API contract tests; allowlist tests | `direct-pancake` |
| P3 | `dynamic-registry` | Discover and register new contracts | direct | trust-policy tests; no arbitrary selector test | `aggregator-routing` |
| P3 | `preflight-and-mev` | Validate risk before signing | direct | simulation/risk tests; security review | `dynamic-registry` |
| P4 | `evm-integration-gate` | Prove end-to-end EVM operation | direct | BSC testnet/manual; no mainnet funds | `preflight-and-mev` |

## Slice `evm-models`

**Objective**

Replace `decimal`-as-amount and EVM-specific DTO leakage with `BigInteger`/string canonical integer amounts, explicit decimals, chain id, token address, native-token marker, router identity, deadline, slippage basis points, gas policy, and typed transaction intent. Every address is checksum-normalized for display and compared case-insensitively by validated value.

**Acceptance criteria**

- Token decimals are read from the token contract/provider and cached with provenance; no path assumes 18 decimals.
- Amount conversion rejects negative, NaN, infinity, overflow, excessive precision, and culture-dependent separators.
- Native BNB and ERC-20 transfers have different commands and ports.
- A quote contains source, timestamp, expiry, route, expected output, minimum output, price impact, fees, gas estimate, and quote id.
- A transaction intent contains chain id, `to`, `value`, calldata hash, selector, spender/allowance target, and recipient; the signer verifies all before signing.

**Files to read (context budget)**

- `2 - Adapter/MetaMask.Integration/Nethereum/PancakeSwapAdapter.cs:32-190`.
- `2 - Adapter/MetaMask.Integration/Nethereum/SwapExactTokensForTokens.cs`.
- `1 - Core/MetaMask.Domain/Adapter/PancakeSwap/*.cs`.
- `1 - Core/MetaMask.Business/OperationBusiness.cs:32-44`.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Trading/Evm/**`.
- `1 - Core/MetaMask.Domain/Ports/Evm/**`.
- `1 - Core/MetaMask.Shared/Amounts/**`.
- `2 - Adapter/MetaMask.Integration/Nethereum/**` minimal typed ABI models.
- `4 - Tests/**` amount/address/calldata tests.

**Depends on**

Forensic `dependency-direction`; secure `custody-boundary`.

**Complexity**

L.

**Owner**

Direct implementation with public-contract and security review.

**Gates**

Property tests for conversion, ABI encoding golden tests, architecture test, and `production-safety-gate`.

**Commit:** `define integer safe evm transaction contracts`

## Slice `direct-pancake`

**Objective**

Implement separate PancakeSwap V2 and V3 adapters. V2 supports pair-path quoting and `swapExactTokensForTokens`/native variants with explicit router configuration. V3 supports pool/fee-tier selection and the official SwapRouter/Quoter contract interfaces. The adapter must query `allowance`, `decimals`, `balanceOf`, quote output, estimate gas, and wait for a receipt whose status is successful before marking the order complete.

**Acceptance criteria**

- Router and factory addresses are network-scoped configuration records with bytecode/code-hash verification.
- Approval is a separate idempotent operation, bounded to exact amount by default; unlimited approvals are disabled unless policy explicitly permits them.
- The path never relies on commented code or implicit WBNB insertion; route construction is explicit and pool existence is checked.
- Transfer supports native BNB and ERC-20 separately, including gas reserve checks and token decimals.
- Receipt status, logs, actual amounts, and final balances are persisted.

**Files to read (context budget)**

- `PancakeSwapAdapter.cs:18-190`.
- `SwapExactTokensForTokens*.cs`.
- `ConfigurationStaticDto.cs` and router/token entities.
- [PancakeSwap Router V2 documentation](https://developer.pancakeswap.finance/contracts/v2/router-v2).

**Files to create/modify**

- `2 - Adapter/MetaMask.Integration/PancakeSwap/**`.
- `2 - Adapter/MetaMask.Integration/Nethereum/**`.
- `1 - Core/MetaMask.Domain/Ports/Evm/**`.
- `1 - Core/MetaMask.Application/Trading/**`.
- `4 - Tests/**` unit/calldata/testnet adapter tests.

**Depends on**

`evm-models`.

**Complexity**

L.

**Owner**

Direct implementation with EVM/security review.

**Gates**

Mock-RPC tests for allowance/decimals/receipt/revert, BSC testnet smoke with a disposable wallet, and no-mainnet-funds manual gate.

**Commit:** `implement safe pancake v2 and v3 execution`

## Slice `aggregator-routing`

**Objective**

Add a quote/execution adapter for 0x Swap API v2 on supported EVM chains and a REST-based LI.FI adapter for routes that include cross-chain steps. Treat API responses as untrusted: validate chain id, allowance target, transaction target, value, calldata selector, recipient, expiry, and token amounts before signing. LI.FI PSBT/native-Bitcoin outputs belong to the Bitcoin plan and are not passed to the EVM signer.

**Acceptance criteria**

- A quote provider cannot choose an unapproved spender or transaction target.
- API key headers are injected per request by typed clients, never via global `DefaultRequestHeaders.Clear()` or static rotation.
- 0x AllowanceHolder/Permit2 flows are explicit and policy-controlled.
- Provider outages, stale quotes, rate limits, and malformed calldata produce typed failures and no broadcast.
- Route fees, gas, bridge fees, and minimum output are visible before operator approval.

**Files to read (context budget)**

- EVM models from `evm-models` and direct adapters.
- Existing Moralis HTTP adapter pattern.
- [0x API overview](https://docs.0x.org/api-reference/api-overview).
- [LI.FI SDK/API architecture](https://docs.li.fi/introduction/lifi-architecture/bitcoin-overview) and REST specification.

**Files to create/modify**

- `2 - Adapter/MetaMask.Integration/Routing/ZeroEx/**`.
- `2 - Adapter/MetaMask.Integration/Routing/LiFi/**`.
- `3 - Infra/MetaMask.Infra/Http/**` typed clients/resilience.
- `1 - Core/MetaMask.Domain/Ports/Routing/**`.
- `4 - Tests/**` provider contract and validation tests.

**Depends on**

`direct-pancake`.

**Complexity**

L.

**Owner**

Direct implementation with integration/security review.

**Gates**

Recorded HTTP fixtures, malformed-response tests, spender/selector allowlist tests, and provider terms/quota review.

**Commit:** `add validated evm aggregator routing`

## Slice `dynamic-registry`

**Objective**

Support new contracts and routers without shipping a new binary for every address. Store network-scoped metadata: contract address, code hash, role, protocol/version, approved ABI fragment, allowed selectors, factory/router provenance, activation status, token-universe eligibility, and operator approval. Discovery can ingest official protocol metadata, provider metadata, and observed pool events, but discovery does not grant live execution permission or inclusion in the day-trading universe.

**Acceptance criteria**

- A discovered contract starts `PendingReview`; only approved metadata can be used in Paper/Production.
- Bytecode absence, code-hash mismatch, proxy implementation mismatch, chain mismatch, or unsupported selector blocks signing.
- Arbitrary ABI blobs and arbitrary calldata are never executable by default.
- Operator can approve, revoke, and pin a connector version with an audit event.
- New token discovery triggers metadata/risk checks before it appears as executable.
- Automatic day trading is restricted to an allowlisted liquidity/volume universe; newly discovered tokens default to observation/paper mode and require explicit approval.

**Files to read (context budget)**

- Connector and contract models from previous slices.
- Existing `Router`, `RoutersForSwap`, and configuration UI entities.
- `Arquitetura.md` router/orchestrator intent.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Contracts/**`.
- `1 - Core/MetaMask.Application/Contracts/**`.
- `2 - Adapter/MRQ.CryptoBot.Repository/**` contract registry migrations.
- `2 - Adapter/MetaMask.Integration/Discovery/**`.
- `0 - Client/MetaMask.Client/UIs/**` approval/review UI.
- `4 - Tests/**` policy tests.

**Depends on**

`aggregator-routing`.

**Complexity**

L.

**Owner**

Direct implementation with security review.

**Gates**

Threat-model review, selector/code-hash tests, and explicit `production-safety-gate`.

**Commit:** `add policy controlled dynamic contract registry`

## Slice `preflight-and-mev`

**Objective**

Build the mandatory preflight pipeline: token/address validation, balance and gas reserve, allowance, quote freshness, slippage/price impact, token-risk provider, `eth_call` simulation, gas estimation, nonce, recipient, day-trading universe eligibility, and optional Tenderly simulation. Add configurable private-RPC/MEV protection where available; never claim sandwich protection from a normal public RPC.

**Acceptance criteria**

- A transaction cannot be signed if simulation reverts, token risk is above policy, output is below minimum, gas reserve is insufficient, or target/selector differs from the approved intent.
- Token-risk results are cached with timestamp and fail closed for new unreviewed tokens in Production.
- Simulation and risk evidence is persisted with the quote/order.
- MEV mode is visible as `public`, `private`, or `unknown`; private routing failure does not silently fall back for high-risk orders.
- Retry after a preflight failure requires a new quote or explicit operator action.

**Files to read (context budget)**

- Transaction intent and dynamic registry models.
- [GoPlus Security API](https://docs.gopluslabs.io/docs/getting-started).
- [Tenderly documentation](https://github.com/Tenderly/tenderly-docs).
- [BNB Chain MEV user guide](https://docs.bnbchain.org/bnb-smart-chain/validator/mev/user-guide/).

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Risk/**`.
- `1 - Core/MetaMask.Application/Risk/**`.
- `2 - Adapter/MetaMask.Integration/Risk/**` and simulation clients.
- `5 - Worker/MRQ.CryptoBot.Worker/Workers/RiskWorker.cs`.
- `4 - Tests/**` risk/simulation tests.

**Depends on**

`dynamic-registry`.

**Complexity**

L.

**Owner**

Direct implementation with production-safety review.

**Gates**

Security review, malicious-token fixtures, revert/simulation tests, and manual MEV mode verification.

**Commit:** `gate evm execution with simulation and token risk`

## Slice `evm-integration-gate`

**Objective**

Run deterministic unit tests plus controlled BSC testnet scenarios for quote, approve, swap, transfer, revert, stale quote, insufficient gas, and receipt reconciliation. No fork feature is required in the application; the test harness may use mocks and testnet identities only.

**Acceptance criteria**

- CI has no dependency on public chain availability for ordinary tests.
- Testnet manual checklist captures chain id, contract target, calldata hash, tx hash, receipt status, and final balance delta.
- A test cannot accidentally use a mainnet RPC or production secret.
- Mainnet execution remains disabled until the production gate is manually signed off.

**Files to read (context budget)**

- All EVM adapter/preflight files.
- Environment matrix from secure plan.
- Worker execution/reconciliation path.

**Files to create/modify**

- `4 - Tests/Evm/**`.
- `scripts/Run-BscTestnetSmoke.ps1`.
- `.docs/operations/evm-testnet-checklist.md`.
- CI configuration for offline tests.

**Depends on**

`preflight-and-mev`.

**Complexity**

M.

**Owner**

Direct implementation.

**Gates**

Build/test/coverage, testnet smoke, security checklist, and production-safety gate.

**Commit:** `verify evm execution on test networks`

## Technology rationale

Nethereum remains the .NET EVM foundation because it covers RPC, typed contract calls, ERC-20 allowance/decimals, signing, gas, and receipts. PancakeSwap is a direct protocol adapter for known router contracts; 0x is the first external aggregator for executable quote/calldata; LI.FI is a cross-chain route source whose Bitcoin/PSBT outputs are handled by another signer boundary. Runtime discovery is a registry and policy problem, not permission to execute arbitrary new ABI data.
