# MRQ.CryptoBot

Bot de trading automatizado para a Binance Smart Chain (BSC), com interface Windows Forms e arquitetura em camadas.

## Visão Geral

O MRQ.CryptoBot é uma aplicação desktop desenvolvida em **.NET 6 / C#** que automatiza operações de compra e venda de tokens na BSC via **PancakeSwap**. Ele monitora preços em tempo real, executa ordens com base em regras configuráveis e mantém histórico local das operações.

### Principais funcionalidades

- Compra e venda automatizada de tokens na BSC (PancakeSwap)
- Monitoramento de preços e liquidez em tempo real
- Controle de slippage, gas limit e limites de saldo
- Carteira criptografada localmente (login/logout seguro)
- Integração com **Moralis** para dados de mercado
- Integração com **BSCScan** para valor corrente dos tokens
- Notificações via **Telegram** *(planejado)*
- Verificação de latência com a blockchain e internet
- Persistência local com **SQLite** via Entity Framework Core
- Log de operações executadas em arquivo local

---

## Arquitetura

O projeto segue uma arquitetura em camadas com separação clara de responsabilidades:

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

### Dependências entre módulos

| Módulo | Depende de |
|---|---|
| `MetaMask.Client` | Application |
| `MetaMask.Application` | Business |
| `MetaMask.Business` | Domain, Integration, Repository |
| `MetaMask.Domain` | Shared |
| `MetaMask.Shared` | — |
| `MetaMask.Integration` | Domain |
| `MetaMask.Repository` | Domain |

> Detalhes completos em [`Dependencia.md`](./Dependencia.md) e [`Arquitetura.md`](./Arquitetura.md).

---

## Tecnologias

| Tecnologia | Uso |
|---|---|
| .NET 6 / C# | Plataforma principal |
| Windows Forms | Interface gráfica |
| [Nethereum](https://nethereum.com/) | Interação com contratos BSC |
| [Moralis](https://moralis.io/) | Dados de preço/balanço de tokens |
| Entity Framework Core + SQLite | Persistência local |
| Microsoft.Extensions.DependencyInjection | Injeção de dependências |
| BSCScan API | Valor corrente dos tokens |

---

## Pré-requisitos

- [.NET 6 SDK](https://dotnet.microsoft.com/download/dotnet/6.0)
- Windows (a interface usa `net6.0-windows` / Windows Forms)
- Visual Studio 2022 ou superior (recomendado)
- Biblioteca externa **`MRQ.ReturnContent`** — deve estar disponível em `..\Library\MRQ.ReturnContent\` relativa à raiz do repositório

---

## Como executar

1. Clone o repositório:

```bash
git clone <url-do-repositorio>
cd MRQ.CryptoBot
```

2. Certifique-se de que a biblioteca `MRQ.ReturnContent` está disponível no caminho esperado (`..\Library\MRQ.ReturnContent\`).

3. Restaure os pacotes e compile a solução:

```bash
dotnet restore MRQ.CryptoBot.sln
dotnet build MRQ.CryptoBot.sln
```

4. Execute o projeto cliente:

```bash
dotnet run --project "0 - Client/MetaMask.Client/MRQ.CryptoBot.Client.csproj"
```

Ou abra `MRQ.CryptoBot.sln` no Visual Studio e pressione **F5**.

---

## Estrutura de diretórios

```
MRQ.CryptoBot/
├── 0 - Client/
│   └── MetaMask.Client/          # Projeto WinForms (UI)
├── 1 - Core/
│   ├── MRQ.CryptoBot.Application/
│   ├── MRQ.CryptoBot.Business/
│   ├── MRQ.CryptoBot.Domain/
│   └── MRQ.CryptoBot.Shared/
├── 2 - Adapter/
│   ├── MRQ.CryptoBot.Integration/ # Nethereum, Moralis, PancakeSwap
│   └── MRQ.CryptoBot.Repository/  # EF Core + SQLite
├── 3 - Infra/                     # Wiring de DI / IoC
├── 4 - Tests/
│   └── PancakeSwapAdapterTest/
├── Arquitetura.md                 # Esboço de API e orquestradores
├── Dependencia.md                 # Grafo de dependências entre módulos
└── MRQ.CryptoBot.sln
```

---

## Testes

```bash
dotnet test MRQ.CryptoBot.sln
```

Os testes ficam em `4 - Tests/PancakeSwapAdapterTest/`.

---

## Convenções

- Todos os métodos devem retornar via **`ReturnContent`** (padrão definido na biblioteca externa).
- Operações assíncronas seguem o padrão `Async` / `await`.
- Regras de negócio residem exclusivamente na camada **Business**; adaptadores externos ficam em **Integration**.
