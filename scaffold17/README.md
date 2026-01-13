# Scaffold17

A complete Stellar/Soroban development stack for hackathon builders. Build, deploy, and explore smart contracts with a ticketing DApp and block explorer.

## What's Inside

| Component | Description | Port |
|-----------|-------------|------|
| **StellarTix** | NFT ticketing DApp with Soroban smart contracts | `5173` |
| **Stellar Explorer** | Block explorer for Stellar/Soroban | `3000` |
| **Wallet Backend** | GraphQL API & blockchain indexer | `8002` |

---

## Quick Start (5 min)

```bash
# 1. Start local Stellar network
stellar network start local

# 2. Start backend services (PostgreSQL, Redis, Indexer, API)
cd soroscan-mvp/stellar-explorer/wallet-backend
docker build -t wallet-backend:latest .
docker-compose -f docker-compose.local.yaml up -d

# 3. Deploy ticketing contracts & start DApp
cd connectiondemo
npm install && npm run install:contracts && npm run dev

# 4. Start block explorer (new terminal)
cd soroscan-mvp/stellar-explorer
npm install && npm run dev
```

**You're live:**
- Ticketing DApp: http://localhost:5173
- Block Explorer: http://localhost:3000
- GraphQL API: http://localhost:8002/graphql/query

---

## Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Stellar Local Network (:8000)                │
│                   Soroban RPC + Horizon + Friendbot             │
└───────────────────────────────┬─────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐      ┌────────────────┐      ┌───────────────┐
│  StellarTix   │      │ Wallet Backend │      │    Explorer   │
│  React DApp   │      │   Go + GraphQL │      │   Next.js     │
│    :5173      │      │     :8002      │      │    :3000      │
└───────────────┘      └────────┬───────┘      └───────┬───────┘
        │                       │                      │
        │              ┌────────┴────────┐             │
        │              ▼                 ▼             │
        │      ┌────────────┐    ┌───────────┐         │
        │      │ PostgreSQL │    │   Redis   │         │
        │      │   :5432    │    │   :6379   │         │
        │      └────────────┘    └───────────┘         │
        │                                              │
        └──────────────── GraphQL API ─────────────────┘
```

### Ticketing Platform (StellarTix)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (React + Vite)                  │
│  ┌──────────┐  ┌────────────┐  ┌───────────┐  ┌──────────────┐  │
│  │  Events  │  │  Purchase  │  │ My Tickets│  │   Debugger   │  │
│  │   Page   │  │   Modal    │  │   Page    │  │    Tools     │  │
│  └────┬─────┘  └─────┬──────┘  └─────┬─────┘  └──────────────┘  │
│       │              │               │                          │
│       └──────────────┼───────────────┘                          │
│                      ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Stellar Wallets Kit (Freighter)             │   │
│  └──────────────────────────────┬───────────────────────────┘   │
└─────────────────────────────────┼───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Soroban Smart Contracts                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Ticketing Contract                      │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────────────┐  │ │
│  │  │   Events    │  │   Tickets   │  │   NFT Ownership    │  │ │
│  │  │  (metadata, │  │  (purchase, │  │  (transfer, enum,  │  │ │
│  │  │   pricing)  │  │   history)  │  │   balance)         │  │ │
│  │  └─────────────┘  └─────────────┘  └────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                  │
│                    Payment: XLM or USDC                         │
└─────────────────────────────────────────────────────────────────┘
```

**Smart Contract Features:**
- NFT-based tickets (SEP-50 compliant)
- XLM/USDC payment options
- Event capacity tracking
- Admin controls for event management

### Block Explorer (Stellar Explorer)

```
┌─────────────────────────────────────────────────────────────────┐
│                   Next.js Frontend (:3000)                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │Dashboard │ │   Txns   │ │  Blocks  │ │Contracts │ │ Tokens │ │
│  │  Stats   │ │  Browser │ │  Browser │ │  Browser │ │  List  │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └───┬────┘ │
│       └────────────┴────────────┴────────────┴───────────┘      │
│                              │                                  │
│  ┌───────────────────────────▼──────────────────────────────┐   │
│  │                    URQL GraphQL Client                   │   │
│  │              (WebSocket subscriptions enabled)           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────┬───────────────────────────────┘
                                  │ GraphQL/WebSocket
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Wallet Backend API (:8002)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                 GraphQL Resolvers                        │   │
│  │   Transactions │ Blocks │ Contracts │ Tokens │ Accounts  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌───────────────────────────▼──────────────────────────────┐   │
│  │                    Query Services                        │   │
│  │             (Indexed data from PostgreSQL)               │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Wallet Backend (Indexer + API)

```
┌─────────────────────────────────────────────────────────────────┐
│                         Stellar RPC                             │
│                        (:8000/soroban/rpc)                      │
└───────────────────────────────┬─────────────────────────────────┘
                                │ Polling
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Ingest Service (:8004)                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Ledger Processor                       │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐  │   │
│  │  │ Transaction│  │  Contract  │  │    State Change    │  │   │
│  │  │  Indexer   │  │  Indexer   │  │      Tracker       │  │   │
│  │  └────────────┘  └────────────┘  └────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
        ┌───────────────────────┴───────────────────────┐
        ▼                                               ▼
┌───────────────────┐                          ┌─────────────────┐
│    PostgreSQL     │                          │      Redis      │
│  ┌─────────────┐  │                          │  ┌───────────┐  │
│  │Transactions │  │                          │  │  Caching  │  │
│  │ Operations  │  │                          │  │  Queues   │  │
│  │ Contracts   │  │                          │  └───────────┘  │
│  │  Accounts   │  │                          └─────────────────┘
│  │   Tokens    │  │
│  │StateChanges │  │
│  └─────────────┘  │
└─────────┬─────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     API Service (:8002)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │   /graphql/query    │   /health   │   WebSocket subs    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

> For detailed wallet-backend documentation, see [wallet-backend/](./soroscan-mvp/stellar-explorer/wallet-backend/)

---

## First-Time Setup Walkthrough

### Prerequisites

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32-unknown-unknown

# Install Stellar CLI
cargo install --locked stellar-cli

# Install Node.js 22+ (via nvm recommended)
nvm install 22 && nvm use 22

# Install Docker
# https://docs.docker.com/get-docker/
```

### Step 1: Start Stellar Local Network

```bash
stellar network start local
```

This starts:
- Soroban RPC at `http://localhost:8000/soroban/rpc`
- Horizon API at `http://localhost:8000`
- Friendbot at `http://localhost:8000/friendbot`

### Step 2: Build & Start Backend Services

```bash
cd soroscan-mvp/stellar-explorer/wallet-backend

# Build the wallet-backend Docker image
docker build -t wallet-backend:latest .

# Start PostgreSQL, Redis, Ingest, and API services
docker-compose -f docker-compose.local.yaml up -d

# Verify services are running
docker-compose -f docker-compose.local.yaml ps
```

Wait for health checks to pass (~30 seconds):
```bash
curl http://localhost:8002/health  # API
curl http://localhost:8004/health  # Ingest
```

### Step 3: Deploy Ticketing Contracts & Start DApp

```bash
cd connectiondemo

# Install dependencies
npm install

# Deploy smart contracts to local network
npm run install:contracts

# Start development server
npm run dev
```

Open http://localhost:5173

**First interaction:**
1. Click "Fund" to get test XLM from friendbot
2. Connect with Freighter wallet
3. Browse events and purchase tickets

### Step 4: Start Block Explorer

```bash
cd soroscan-mvp/stellar-explorer

# Install dependencies
npm install

# Start Next.js dev server
npm run dev
```

Open http://localhost:3000

View your transactions, contracts, and network stats in real-time.

---

## Tech Stack

| Layer | Technologies |
|-------|-------------|
| **Smart Contracts** | Rust, Soroban SDK, OpenZeppelin Stellar |
| **Frontend** | React 19, Next.js 16, TypeScript, Vite, Tailwind |
| **Backend** | Go 1.24, GraphQL (gqlgen), PostgreSQL, Redis |
| **Blockchain** | Stellar, Soroban RPC |
| **DevOps** | Docker, Docker Compose |

---

## Useful Commands

```bash
# Ticketing DApp
cd connectiondemo
npm run dev              # Start with hot reload
npm run build            # Production build
npm run install:contracts # Deploy contracts

# Block Explorer
cd soroscan-mvp/stellar-explorer
npm run dev              # Start dev server
npm run build            # Production build

# Wallet Backend
cd soroscan-mvp/stellar-explorer/wallet-backend
docker-compose -f docker-compose.local.yaml up -d    # Start all
docker-compose -f docker-compose.local.yaml down     # Stop all
docker-compose -f docker-compose.local.yaml logs -f  # View logs

# Stellar Network
stellar network start local   # Start network
stellar network stop local    # Stop network
```

---

## Contract Addresses (Local)

| Contract | Address |
|----------|---------|
| Ticketing | `CDX7PH4GBUU446BURBNJCDIAR4J24P57L3562ESRBXYS2XGPZVW3FWNM` |
| Guess the Number | `CCKFJT2MK45KVLMKF4OXAXVXMFXNVCNDQWWVILVP7OB5TFJZ6BQQJT4Q` |
| NFT Enumerable | `CDS4QTWGTCDD64O4RWGOYUTSLTJI5FTMF2DPK7NV7XRUI4GKREOGTSLM` |
| Fungible Allowlist | `CDJGDDPJR6T37U3JPYYSABRODMCR5LG6D3HVACTAMWZA5DCQNSZIKGWT` |

---

## Troubleshooting

**"Connection refused" on port 8002**
→ Wait for wallet-backend to finish starting: `docker-compose logs -f soroscan-api-local`

**Contracts not deploying**
→ Ensure Stellar network is running: `curl http://localhost:8000/`

**Wallet not connecting**
→ Check Freighter is set to "Standalone Network" with passphrase: `Standalone Network ; February 2017`

---

## License

MIT
