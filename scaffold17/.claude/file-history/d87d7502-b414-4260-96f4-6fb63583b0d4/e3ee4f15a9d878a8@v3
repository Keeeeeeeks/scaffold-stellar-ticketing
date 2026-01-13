# Local Block Explorer Integration Plan

Add a local Soroscan block explorer to view transactions from the local Stellar network, with deeplinks from the StellarTix app.

---

## Overview

| Component | Location | Port |
|-----------|----------|------|
| Local Stellar Network | Already running | 8000 |
| Soroscan Frontend | `/Users/george/scaffold17/soroscan-mvp/stellar-explorer` | 3000 |
| Wallet-Backend (Go) | `/Users/george/scaffold17/soroscan-mvp/stellar-explorer/wallet-backend` | 8002 |
| StellarTix App | `/Users/george/scaffold17/connectiondemo` | 5173 |

**Transaction URL Pattern**: `http://localhost:3000/tx/{hash}`

---

## Phase 1: Configure and Run Soroscan

### 1.1 Configure Frontend Environment

**File**: `/Users/george/scaffold17/soroscan-mvp/stellar-explorer/.env.local`

```env
NEXT_PUBLIC_GRAPHQL_URL=http://localhost:8002/graphql
NEXT_PUBLIC_GRAPHQL_WS_URL=ws://localhost:8002/graphql
NEXT_PUBLIC_SOROBAN_RPC_URL=http://localhost:8000/soroban/rpc
NEXT_PUBLIC_HORIZON_URL=http://localhost:8000
NEXT_PUBLIC_NETWORK_PASSPHRASE="Standalone Network ; February 2017"
NEXT_PUBLIC_NETWORK=local
```

### 1.2 Configure Wallet-Backend for Local Network

**File**: `/Users/george/scaffold17/soroscan-mvp/stellar-explorer/wallet-backend/.env`

Key settings needed:
- `DATABASE_URL` - PostgreSQL connection
- `NETWORK=local`
- `STELLAR_ENVIRONMENT=development`
- Point RPC to `http://localhost:8000/soroban/rpc`
- Point Horizon to `http://localhost:8000`

### 1.3 Start Services

```bash
# 1. Start wallet-backend (with PostgreSQL, Redis)
cd /Users/george/scaffold17/soroscan-mvp/stellar-explorer/wallet-backend
docker compose up -d

# 2. Wait for services to be healthy
# 3. Run migrations
go run main.go migrate up

# 4. Start ingestion (indexes transactions from local network)
go run main.go ingest &

# 5. Start API server
go run main.go serve &

# 6. Start frontend
cd /Users/george/scaffold17/soroscan-mvp/stellar-explorer
npm install
npm run dev
```

---

## Phase 2: Add Soroscan Deeplinks to StellarTix

### 2.1 Create Explorer URL Utility

**New File**: `src/util/explorer.ts`

```typescript
const SOROSCAN_BASE_URL = "http://localhost:3000";

export const getTransactionUrl = (txHash: string): string => {
  return `${SOROSCAN_BASE_URL}/tx/${txHash}`;
};

export const getAccountUrl = (address: string): string => {
  return `${SOROSCAN_BASE_URL}/account/${address}`;
};

export const getContractUrl = (contractId: string): string => {
  return `${SOROSCAN_BASE_URL}/contract/${contractId}`;
};
```

### 2.2 Create TransactionLink Component

**New File**: `src/components/common/TransactionLink.tsx`

A reusable component that:
- Displays truncated tx hash
- Links to Soroscan
- Shows external link icon
- Opens in new tab

### 2.3 Update EventDetailPage

**File**: `src/pages/EventDetailPage.tsx` (lines 120-125)

Replace plain text hash display with `<TransactionLink>` component.

### 2.4 Update MyTicketsPage (Optional)

**File**: `src/pages/MyTicketsPage.tsx`

Add transaction link to each ticket card if we store the purchase tx hash.

**Note**: Currently `UserTicket` type doesn't include `txHash`. May need to:
1. Add `purchaseTxHash` field to ticket metadata
2. Or query from contract events

---

## Phase 3: Handle Wallet-Backend Availability

### 3.1 Health Check Utility

Create a utility that checks if wallet-backend is available:

```typescript
export const checkExplorerHealth = async (): Promise<boolean> => {
  try {
    const response = await fetch("http://localhost:8002/graphql", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ query: "{ __typename }" }),
    });
    return response.ok;
  } catch {
    return false;
  }
};
```

### 3.2 Graceful Degradation

The `<TransactionLink>` component should:
1. Check if explorer is available
2. If available: show clickable link
3. If unavailable: show plain text hash with tooltip "Explorer not available"

---

## Critical Files to Modify/Create

### Soroscan Configuration
- `soroscan-mvp/stellar-explorer/.env.local` - UPDATE
- `soroscan-mvp/stellar-explorer/wallet-backend/.env` - UPDATE
- `soroscan-mvp/stellar-explorer/wallet-backend/docker-compose.yml` - May need local network config

### StellarTix App
- `src/util/explorer.ts` - NEW
- `src/components/common/TransactionLink.tsx` - NEW
- `src/pages/EventDetailPage.tsx` - UPDATE (add deeplink)
- `src/pages/MyTicketsPage.tsx` - UPDATE (optional)
- `src/components/tickets/TicketCard.tsx` - UPDATE (optional)

---

## Verification Steps

1. **Soroscan Running**: Visit `http://localhost:3000` - should show block explorer
2. **GraphQL API**: Query `http://localhost:8002/graphql` - should respond
3. **Ingestion Working**: Recent transactions from local network appear in Soroscan
4. **Deeplinks Work**:
   - Buy a ticket in StellarTix
   - Click on transaction hash link
   - Should open Soroscan with transaction details

---

## Dependencies & Order

```
1. Local Stellar Network (already running at :8000)
       ↓
2. wallet-backend services (PostgreSQL, Redis, API, Ingest)
       ↓
3. Soroscan frontend (:3000)
       ↓
4. StellarTix deeplinks (point to :3000/tx/{hash})
```

---

## Questions/Considerations

1. **Wallet-backend complexity**: The Go backend requires PostgreSQL, Redis, and proper configuration. May need to simplify or check if there's a simpler setup.

2. **Ingestion**: wallet-backend needs to ingest transactions from the local network. Need to verify it can connect to localhost:8000 for RPC.

3. **Demo mode**: Soroscan has a "demo mode" when backend is unavailable - may want to disable this for local dev.
