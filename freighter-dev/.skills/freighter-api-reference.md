# Freighter API Reference

> **Skill**: Load this when you need to know what endpoints exist, what they accept, what they return, and which backend owns them.
>
> **When to use**: Designing new API endpoints, understanding client→backend data flow, writing integration specs, planning v1→v2 migrations.

---

## Freighter Backend v1 (Node.js/Fastify)

Base URL: configured via `FREIGHTER_BACKEND_V1_URL`
All routes prefixed: `/api/v1/`

### Health & Configuration

| Method | Path | Purpose | Response |
|--------|------|---------|----------|
| GET | `/ping` | Health check | `"Alive!"` |
| GET | `/price-worker-health` | Price cache health | `{status}` |
| GET | `/rpc-health?network=` | Stellar RPC health | `{status}` |
| GET | `/horizon-health?network=` | Horizon health | `{database_connected, core_up, core_synced}` |
| GET | `/feature-flags` | Simple feature flags | `{useSorobanPublic}` |
| GET | `/user-notification` | In-app notifications | `{enabled, message}` |

### Account Data

| Method | Path | Params | Response |
|--------|------|--------|----------|
| GET | `/account-balances/:pubKey` | `?network=&contract_ids[]=` | `{balances: BalanceMap, isFunded, subentryCount, error?}` |
| GET | `/account-history/:pubKey` | `?network=&is_failed_included=` | `OperationRecord[]` |

### Token Operations

| Method | Path | Params | Response |
|--------|------|--------|----------|
| GET | `/token-details/:contractId` | `?pub_key=&network=&should_fetch_balance=` | `{symbol, name, decimals, ...}` |
| GET | `/token-spec/:contractId` | `?network=` | `{data: boolean, error}` |
| GET | `/contract-spec/:contractId` | `?network=` | `{data: ContractSpec, error}` |
| GET | `/is-sac-contract/:contractId` | `?network=` | `{isSacContract: boolean}` |
| POST | `/token-prices` | `{tokens: string[]}` | `{data: {[token]: {currentPrice, percentagePriceChange24h}}}` |

### Security Scanning (Blockaid)

| Method | Path | Params | Response |
|--------|------|--------|----------|
| GET | `/scan-dapp` | `?url=` | `{data: {status}, error}` |
| GET | `/scan-tx` | `?tx_xdr=&url=&network=` | `{data: ScanResult, error}` |
| POST | `/scan-tx` | `{tx_xdr, url, network}` | `{data: ScanResult, error}` |
| GET | `/scan-asset` | `?address=` | `{data: AssetScanResponse, error}` |
| GET | `/scan-asset-bulk` | `?asset_ids[]=` | `{data: {results: {[addr]: ScanResponse}}, error}` |
| GET | `/report-asset-warning` | `?details=&address=` | `{data, error}` |
| GET | `/report-transaction-warning` | `?details=&request_id=&event=` | `{data, error}` |

### Transaction Operations

| Method | Path | Body | Response |
|--------|------|------|----------|
| POST | `/submit-tx` | `{signed_xdr, network_url, network_passphrase}` | Horizon submit response |
| POST | `/simulate-tx` | `{xdr, network_url, network_passphrase}` | `{simulationResponse, preparedTransaction}` |
| POST | `/simulate-token-transfer` | `{address, pub_key, memo, fee?, params, network_url, network_passphrase}` | `{simulationResponse, preparedTransaction}` |

### Subscriptions (Mercury)

| Method | Path | Body | Response |
|--------|------|------|----------|
| POST | `/subscription/token` | `{contract_id, pub_key, network}` | Subscription result |
| POST | `/subscription/account` | `{pub_key, network}` | Subscription result |
| POST | `/subscription/token-balance` | `{contract_id, pub_key, network}` | Subscription result |

### Onramp

| Method | Path | Body | Response |
|--------|------|------|----------|
| POST | `/onramp/token` | `{address}` | `{data: {token}}` |

---

## Freighter Backend v2 (Go/net-http)

Base URL: configured via `FREIGHTER_BACKEND_V2_URL`
All routes prefixed: `/api/v1/`

| Method | Path | Request | Response | Upstream |
|--------|------|---------|----------|----------|
| GET | `/ping` | — | `{status: "ok"}` | — |
| GET | `/protocols` | — | `{data: {protocols: Protocol[]}}` | JSON config file |
| GET | `/feature-flags` | `?platform=&version=` | `{swap_enabled, discover_enabled, onramp_enabled}` | Hardcoded logic |
| POST | `/account-balances` | `?network=` + `{addresses: string[]}` | `{data: AccountBalances}` | Wallet Backend (wbclient) |
| POST | `/ledger-key/accounts` | `{public_keys: string[], network: string}` | `{data: LedgerEntryMap[]}` | Stellar RPC |
| POST | `/collectibles` | `{owner, contracts: [{id, token_ids}], network}` | `{data: {collections: Collection[]}}` | Stellar RPC |

### Protocol Type (from JSON config)

```json
{
    "name": "Blend",
    "tags": ["DeFi", "Lending"],
    "website_url": "https://blend.capital",
    "icon_url": "https://...",
    "description": "Decentralized lending protocol",
    "is_blacklisted": false,
    "is_wc_not_supported": false
}
```

### Feature Flags Logic

```
Default: all true (swap, discover, onramp)
Exception: iOS v1.6.23 → all false
```

### Account Balances Request/Response

```
POST /api/v1/account-balances?network=PUBLIC
{
    "addresses": ["GABC...", "GDEF..."]
}

→ Proxied to wallet-backend via wbclient.GetBalancesByAccountAddresses()
→ Returns GraphQL balancesByAccountAddresses response shape
```

---

## Wallet Backend (GraphQL)

Base URL: configured via `WalletBackendConfig.PubnetUrl` / `TestnetUrl`
Authentication: JWT signed with Stellar keypair

### Queries

```graphql
# Single account balances
query {
    balancesByAccountAddress(address: "GABC...") {
        ... on NativeBalance { balance minimumBalance buyingLiabilities sellingLiabilities }
        ... on TrustlineBalance { balance code issuer limit isAuthorized }
        ... on SACBalance { balance code issuer decimals isAuthorized isClawbackEnabled }
        ... on SEP41Balance { balance name symbol decimals }
    }
}

# Multi-account balances (used by Freighter's wallet view)
query {
    balancesByAccountAddresses(addresses: ["GABC...", "GDEF..."]) {
        address
        balances { balance tokenId tokenType }
        error
    }
}

# Transaction lookup
query { transactionByHash(hash: "abc...") { hash envelopeXdr feeCharged resultCode ledgerNumber } }

# Account with relationships
query {
    accountByAddress(address: "GABC...") {
        address
        transactions(first: 10) { edges { node { hash ledgerCreatedAt } } }
        operations(first: 10) { edges { node { id operationType successful } } }
        stateChanges(filter: {category: "BALANCE"}, first: 10) {
            edges { node { ... on StandardBalanceChange { tokenId amount } } }
        }
    }
}
```

### Mutations

```graphql
# Register account for tracking
mutation { registerAccount(input: {address: "GABC..."}) { success account { address } } }

# Build transaction (wraps with fee bump)
mutation {
    buildTransaction(input: {
        transactionXdr: "..."
        simulationResult: { transactionData: "...", minResourceFee: "100" }
    }) { success transactionXdr }
}
```

---

## Client → Backend Routing

### Extension API Calls

```typescript
// @shared/api/internal.ts (via background message bridge)
getDiscoverData()          → background → freighterBackendV2.GET("/protocols")
getAccountBalances(pubKey) → background → freighterBackendV1.GET("/account-balances/:pubKey")
getAccountHistory(pubKey)  → background → freighterBackendV1.GET("/account-history/:pubKey")
scanDapp(url)              → background → freighterBackendV1.GET("/scan-dapp")
scanTx(xdr, url, network) → background → freighterBackendV1.POST("/scan-tx")
```

### Mobile API Calls

```typescript
// services/backend.ts
fetchBalances({publicKey, network})       → freighterBackendV1.GET("/account-balances/:pubKey")
fetchTokenPrices({tokens})                → freighterBackendV1.POST("/token-prices")
getAccountHistory({publicKey})            → freighterBackendV1.GET("/account-history/:pubKey")
fetchProtocols()                          → freighterBackendV2.GET("/protocols")
fetchCollectibles({owner, contracts})     → freighterBackendV2.POST("/collectibles")
getTokenDetails({contractId, publicKey})  → freighterBackendV1.GET("/token-details/:contractId")
simulateTokenTransfer(params)             → freighterBackendV1.POST("/simulate-token-transfer")
submitTransaction(body)                   → freighterBackendV1.POST("/submit-tx")
```

---

## Authentication

| System | Auth Method | Details |
|--------|------------|---------|
| v1 endpoints | None | Rate-limited (100 req/min via Redis) |
| v2 endpoints | None | Body size limited (1MB) |
| Wallet Backend | JWT | Signed with Stellar keypair; per-request token via `wbclient.auth.NewHTTPRequestSigner` |
| Admin (future) | TBD | Not yet implemented |
