# MRQ.CryptoBot

Automated trading bot for Binance Smart Chain (BSC) with a Windows Forms UI and a layered architecture.

## Overview

MRQ.CryptoBot is a desktop app built with **.NET 6 / C#** that automates buying and selling tokens on BSC through **PancakeSwap**. It monitors prices in real time, executes orders based on configurable rules, and keeps a local history of operations.

### Key features

- Automated buy/sell of BSC tokens (PancakeSwap)
- Real-time price and liquidity monitoring
- Control of slippage, gas limit, and balance limits
- Locally encrypted wallet (secure login/logout)
- **Moralis** integration for market data
- **BSCScan** integration for current token values
- **Telegram** notifications *(planned)*
- Latency checks against the blockchain and the internet
- Local persistence with **SQLite** via Entity Framework Core
- Log of executed operations to a local file

---

## Architecture

The solution uses a layered architecture with clear separation of concerns:

```
┌──────────────────────────────────────────────────┐
│                0 - Client (WinForms)              │
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│              1 - Core                             │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────┐  │
│  │ Application │→ │ Business │→ │   Domain    │  │
│  └─────────────┘  └──────────┘  └──────┬──────┘  │
│                                         │         │
│                                  ┌──────▼──────┐  │
│                                  │   Shared    │  │
│                                  └─────────────┘  │
└──────────────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼──────┐ ┌─────▼──────┐      │
│ 2 - Adapter  │ │ Repository │      │
│  (Nethereum, │ │  (EF Core) │      │
│   Moralis)   │ └────────────┘      │
└──────────────┘                     │
        │                            │
┌───────▼─────────────────────────── ▼──┐
│             3 - Infra (DI / IoC)      │
└───────────────────────────────────────┘
```

### Module dependencies

| Module | Depends on |
|--------|------------|
| `MetaMask.Client` | Application |
| `MetaMask.Application` | Business |
| `MetaMask.Business` | Domain, Integration, Repository |
| `MetaMask.Domain` | Shared |
| `MetaMask.Shared` | — |
| `MetaMask.Integration` | Domain |
| `MetaMask.Repository` | Domain |

> Full details: [`Dependencia.md`](./Dependencia.md) and [`Arquitetura.md`](./Arquitetura.md).

---

## Technologies

| Technology | Role |
|------------|------|
| .NET 6 / C# | Main platform |
| Windows Forms | Desktop UI |
| [Nethereum](https://nethereum.com/) | BSC contract interaction |
| [Moralis](https://moralis.io/) | Token price/balance data |
| Entity Framework Core + SQLite | Local persistence |
| Microsoft.Extensions.DependencyInjection | Dependency injection |
| BSCScan API | Current token values |

---

## Prerequisites

- [.NET 6 SDK](https://dotnet.microsoft.com/download/dotnet/6.0)
- Windows (the UI targets `net6.0-windows` / Windows Forms)
- Visual Studio 2022 or later (recommended)
- External library **`MRQ.ReturnContent`** — must be available at `..\Library\MRQ.ReturnContent\` relative to the repository root

---

## How to run

1. Clone the repository:

```bash
git clone <repository-url>
cd MRQ.CryptoBot
```

2. Ensure the `MRQ.ReturnContent` library is on the expected path (`..\Library\MRQ.ReturnContent\`).

3. Restore packages and build the solution:

```bash
dotnet restore MRQ.CryptoBot.sln
dotnet build MRQ.CryptoBot.sln
```

4. Run the client project:

```bash
dotnet run --project "0 - Client/MetaMask.Client/MRQ.CryptoBot.Client.csproj"
```

Or open `MRQ.CryptoBot.sln` in Visual Studio and press **F5**.

---

## Directory layout

```
MRQ.CryptoBot/
├── 0 - Client/
│   └── MetaMask.Client/          # WinForms project (UI)
├── 1 - Core/
│   ├── MRQ.CryptoBot.Application/
│   ├── MRQ.CryptoBot.Business/
│   ├── MRQ.CryptoBot.Domain/
│   └── MRQ.CryptoBot.Shared/
├── 2 - Adapter/
│   ├── MRQ.CryptoBot.Integration/ # Nethereum, Moralis, PancakeSwap
│   └── MRQ.CryptoBot.Repository/  # EF Core + SQLite
├── 3 - Infra/                     # DI / IoC wiring
├── 4 - Tests/
│   └── PancakeSwapAdapterTest/
├── Arquitetura.md                 # API sketch and orchestrators
├── Dependencia.md                 # Module dependency graph
└── MRQ.CryptoBot.sln
```

---

## Tests

```bash
dotnet test MRQ.CryptoBot.sln
```

Tests live under `4 - Tests/PancakeSwapAdapterTest/`.

---

## Conventions

- Methods should return via **`ReturnContent`** (pattern from the external library).
- Asynchronous operations follow `Async` / `await`.
- Business rules live in **Business** only; external adapters belong in **Integration**.
