---
name: Secure environments and local custody
overview: Define reproducible environment generation and protect local EVM/Bitcoin/CEX secrets with Windows-bound encryption and auditable access policies.
isProject: false
todos:
  - id: env-contract
    content: "Define environment profiles, configuration keys, redaction rules, and safe local-generation commands"
    status: in_progress
  - id: dpapi-secret-store
    content: "Implement the Windows-bound encrypted secret store and key envelope lifecycle"
    status: pending
  - id: custody-boundary
    content: "Remove raw-key transport through UI, DTOs, persistence, logs, and connector APIs"
    status: pending
  - id: recovery-and-rotation
    content: "Add explicit secret rotation, backup, recovery, and operator confirmation flows"
    status: pending
  - id: environment-gate
    content: "Prove environment isolation and secret non-disclosure across Development, Test, Paper, and Production"
    status: pending
---

## Phases

P1 — Make configuration reproducible without putting secrets in Git.
P2 — Protect secrets at rest and in memory at the application boundary.
P3 — Make recovery and rotation deliberate, observable, and testable.
P4 — Verify every environment fails closed.

## Slices

| Phase | Slice | Objective | Owner | Gates | Depends on |
|---|---|---|---|---|---|
| P1 | `env-contract` | Define safe configuration and generation | direct | config validation; secret scan | forensic `foundation-gate` |
| P2 | `dpapi-secret-store` | Encrypt secrets using Windows protection | direct | crypto tests; security review | `env-contract` |
| P2 | `custody-boundary` | Keep decrypted material out of UI and persistence models | direct | architecture/security tests | `dpapi-secret-store` |
| P3 | `recovery-and-rotation` | Support rotation and documented recovery | direct | destructive-action review; backup tests | `custody-boundary` |
| P4 | `environment-gate` | Prove environment-specific safety switches | direct | matrix tests; manual production gate | `recovery-and-rotation` |

## Slice `env-contract`

**Objective**

Create the canonical environment contract. Use `appsettings.json` for non-secret defaults and environment variables with the `MRQ__` prefix for overrides. A `.env.example` is a developer template only; the .NET host must not load arbitrary `.env` files in Production. Real secrets are entered through the desktop setup flow or a protected provisioning command and stored in the Windows-bound secret store.

Profiles: `Development`, `Test`, `Paper`, and `Production`. Networks: BSC testnet/mainnet, Bitcoin regtest/testnet/mainnet, and CEX sandbox/live where supported. Include `Ollama:BaseUrl`, `Ollama:Model`, `Ollama:TimeoutMs`, `AI:Enabled`, and `AI:Mode=AdvisoryOnly`; the local model endpoint is not a secret and must not be an execution dependency. Production requires explicit `Execution:Enabled=true`, `Execution:Environment=Production`, an operator confirmation, and a configured risk policy.

**Acceptance criteria**

- A validated schema covers RPC endpoints, chain ids, database path, provider base URLs, rate limits, worker concurrency, queue lease durations, risk limits, execution mode, and secret references.
- No secret value is present in `appsettings*.json`, `.env.example`, scripts, test fixtures, exceptions, logs, or README examples.
- The repository ignores `.env.local`, generated local config, secret envelopes, and application-data folders.
- The documented commands are:
  - `pwsh ./scripts/Initialize-LocalEnvironment.ps1 -Environment Development -Network BscTestnet`
  - `pwsh ./scripts/Initialize-LocalEnvironment.ps1 -Environment Test -Network BitcoinRegtest`
  - `dotnet run --project <worker-project> -- --environment Paper --validate-config`
- The script creates only safe local files and prints names/status, never secret contents.
- Generated Development/Paper environments default to `Execution:Enabled=false`, `AI:Mode=AdvisoryOnly`, and a bounded allowlist of networks/connectors.

**Files to read (context budget)**

- `0 - Client/MetaMask.Client/appsettings.json`.
- `.gitignore`.
- All current resource files containing base URLs and configuration labels.
- Host composition files created by the foundation plan.

**Files to create/modify**

- `.env.example`.
- `appsettings.example.json` and `appsettings.Development.example.json`.
- `scripts/Initialize-LocalEnvironment.ps1`.
- `scripts/Validate-Configuration.ps1`.
- `.gitignore`.
- `.docs/operations/environments.md`.
- `Directory.Build.props` or host configuration extension.

**Depends on**

Forensic plan `foundation-gate`.

**Complexity**

M.

**Owner**

Direct implementation with security review.

**Gates**

Configuration binding tests, PowerShell `-WhatIf`/dry-run test, secret scanner, and `production-safety-gate`.

**Commit:** `define reproducible environment contract`

## Slice `dpapi-secret-store`

**Objective**

Implement `ISecretStore` backed by Windows DPAPI/Data Protection. The first deployment model runs the WinForms process and workers under the same Windows user, using `CurrentUser` protection and an application-specific purpose string. If a Windows service account is later required, use a dedicated service identity with `LocalMachine` protection plus an ACL and a migration test; do not silently switch scopes.

Store an envelope containing version, purpose, algorithm, protected payload, secret kind, wallet id/provider id, created/rotated timestamps, and key fingerprint. The database stores the envelope only. The decrypted private key exists only within a narrow signing operation and is never returned as a domain DTO.

**Acceptance criteria**

- A database copy without the Windows protection context cannot decrypt the envelope.
- A different purpose, wallet id, or environment cannot decrypt it.
- Tampered ciphertext, wrong version, missing scope, and revoked secret fail closed.
- EVM private keys, Bitcoin seed/WIF/xprv, CEX API keys/secrets, and provider API keys use the same typed store but separate purposes and access policies.
- Logs and diagnostics expose only secret kind, wallet id, and fingerprint suffix.

**Files to read (context budget)**

- New configuration contract from `env-contract`.
- `1 - Core/MetaMask.Domain/Entities/Wallet.cs` and `Application/Moralis/ApiKeyDto.cs`.
- `2 - Adapter/MRQ.CryptoBot.Repository/.../SQLiteContext.cs` and migrations.
- `.docs/audits/2026-09-01-forensic-baseline.md` F-003/F-004/F-011/F-012.

**Files to create/modify**

- `1 - Core/MetaMask.Domain/Ports/ISecretStore.cs` and secret metadata contracts.
- `3 - Infra/MetaMask.Infra/Security/WindowsDataProtectionSecretStore.cs`.
- `2 - Adapter/MRQ.CryptoBot.Repository/**` encrypted envelope mapping.
- `4 - Tests/**` crypto and persistence tests.
- `.docs/operations/secret-custody.md`.

**Depends on**

`env-contract`.

**Complexity**

L — security-sensitive persistence and identity scope.

**Owner**

Direct implementation with `production-safety-gate`.

**Gates**

Tamper, wrong-user, wrong-purpose, rotation, crash, and no-secret-logging tests; manual review of threat model.

**Commit:** `add windows protected local secret store`

## Slice `custody-boundary`

**Objective**

Remove raw private-key fields from `WalletDto`, UI forms, application commands, and generic operation records. The UI selects a wallet id and asks for a signing capability; it does not pass a key string. The signer resolves the secret just before signing, verifies the derived public address, signs, and returns only signed payload metadata or a transaction id.

**Acceptance criteria**

- `rg "PrivateKey|privateKey|ApiKey"` shows no raw secret crossing the UI/application boundary except the typed secret-store implementation and provisioning input.
- The address derived from a provisioned EVM key must match the configured address before signing.
- Bitcoin network, derivation policy, and address must match the secret metadata before signing.
- CEX permissions are read/trade only by default; withdrawal permission is rejected by configuration and tests.
- Signing is disabled in Development/Test unless explicitly using deterministic test keys.

**Files to read (context budget)**

- `0 - Client/MetaMask.Client/Form1.cs:67-76,126-145`.
- `1 - Core/MetaMask.Domain/Adapter/PancakeSwap/WalletDto.cs`.
- `1 - Core/MetaMask.Domain/Entities/Wallet.cs`.
- All connector interfaces introduced by sibling plans.

**Files to create/modify**

- `0 - Client/MetaMask.Client/**` wallet selection/provisioning UI.
- `1 - Core/MetaMask.Domain/**` wallet/signing contracts.
- `1 - Core/MetaMask.Application/**` commands.
- `2 - Adapter/**` signer implementations.
- `4 - Tests/**` boundary tests.

**Depends on**

`dpapi-secret-store`.

**Complexity**

L.

**Owner**

Direct implementation with security review.

**Gates**

Architecture search, address-mismatch tests, memory/log redaction checks, and manual UI review.

**Commit:** `remove raw keys from application boundaries`

## Slice `recovery-and-rotation`

**Objective**

Add operator-controlled rotation and recovery. Normal recovery is a protected export containing an encrypted secret envelope plus metadata, protected with a user-supplied passphrase using a documented KDF and authenticated encryption. The export is never automatic and is never written to the repository. Restore requires address/network verification and a confirmation step. Windows-profile loss is an explicit operational risk, not hidden by pretending DPAPI is portable.

**Acceptance criteria**

- Rotation creates a new version, revokes the old version after a successful verification, and records an audit event without secret contents.
- Backup/export and restore are tested with wrong passphrase, tampered file, wrong network, wrong address, and duplicate import.
- A lost Windows profile scenario is documented with the recovery file requirement.
- Provider API key rotation can overlap old/new keys without mutable global lists or header races.

**Files to read (context budget)**

- `ISecretStore` and envelope model from prior slices.
- Existing Moralis key entities and configuration UI.
- Database backup plan from forensic foundation.

**Files to create/modify**

- `1 - Core/MetaMask.Application/Security/**`.
- `3 - Infra/MetaMask.Infra/Security/**`.
- `0 - Client/MetaMask.Client/UIs/**`.
- `scripts/Export-LocalSecrets.ps1` and `scripts/Import-LocalSecrets.ps1`.
- `.docs/operations/secret-custody.md`.
- `4 - Tests/**` recovery tests.

**Depends on**

`custody-boundary`.

**Complexity**

L.

**Owner**

Direct implementation with security review.

**Gates**

Cryptographic design review, no-secret artifact scan, recovery drills, and explicit destructive-action confirmation.

**Commit:** `add secret rotation and recovery workflow`

## Slice `environment-gate`

**Objective**

Exercise every environment profile and ensure unsafe combinations fail closed. Paper mode may consume live quotes but must never broadcast. Test mode uses deterministic fake signers or testnet/regtest identities. Production mode requires all risk, custody, provider, and backup checks.

**Acceptance criteria**

- A configuration matrix test covers all profile/network combinations and expected execution permission.
- A worker cannot start with missing secret store, mismatched chain id, production execution without risk policy, or a mainnet endpoint in Test.
- The generated `.env` and JSON examples contain placeholders only.
- `Initialize-LocalEnvironment.ps1` is idempotent and does not overwrite existing secrets without `-Force` plus confirmation.

**Files to read (context budget)**

- All environment artifacts from previous slices.
- Worker host configuration from the queue plan.
- Execution gates from EVM/Bitcoin plans.

**Files to create/modify**

- `4 - Tests/EnvironmentTests/**`.
- `scripts/Initialize-LocalEnvironment.ps1` and validation scripts.
- `.docs/operations/environments.md`.
- CI workflow secret scanning.

**Depends on**

`recovery-and-rotation`.

**Complexity**

M.

**Owner**

Direct implementation.

**Gates**

Configuration matrix, secret scan, clean-room local setup, and manual production approval checklist.

**Commit:** `gate execution by environment and network`

## Security decision

The selected strategy is system-bound encryption, not a custom wallet vault: DPAPI/Data Protection protects ciphertext with the Windows user/service identity; SQLite stores only envelopes; the application obtains a secret only inside a signing scope. This matches the requested local custody while avoiding plaintext database storage. It does not protect against a fully compromised interactive account or process memory; that residual risk is documented and mitigated with least privilege, no withdrawal permissions for CEX, address verification, audit events, and production confirmation.

## Official references

- [.NET cryptography overview](https://learn.microsoft.com/en-us/dotnet/standard/security/cryptographic-services).
- [Nethereum documentation](https://docs.nethereum.com/).
- [Bitcoin Core RPC reference](https://developer.bitcoin.org/reference/rpc/).
