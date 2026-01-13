# Summary of Changes - January 8, 2026

## 1. Block Explorer (soroscan-mvp/stellar-explorer)

### Hydration Fix - `src/components/layout/SearchBar.tsx`
Fixed React hydration mismatch error caused by server/client rendering different keyboard shortcuts (Ctrl+K vs ⌘K).

**Changes:**
- Added `isMac` state variable initialized to `false`
- Added `useEffect` to detect Mac after hydration using `navigator.userAgent`
- Changed inline `typeof navigator` check to use the state variable

```tsx
const [isMac, setIsMac] = useState(false);

useEffect(() => {
  setIsMac(/Mac/i.test(navigator.userAgent));
}, []);

// In JSX:
<kbd>{isMac ? '⌘K' : 'Ctrl+K'}</kbd>
```

### Network Selector Note
The `NetworkSelector` component (`src/components/layout/NetworkSelector.tsx`) is cosmetic only - it uses local React state but doesn't actually change the API endpoints. The explorer always uses URLs from `.env.local`.

---

## 2. Wallet Backend (soroscan-mvp/stellar-explorer/wallet-backend)

### Added `--skip-account-tokens-cache` Flag
Added support for local networks without history archives by skipping the initial account tokens cache population.

**Files Modified:**

#### `cmd/ingest.go`
Added new CLI flag:
```go
{
    Name:        "skip-account-tokens-cache",
    Usage:       "Skip initial account tokens cache population from history archives...",
    OptType:     types.Bool,
    ConfigKey:   &cfg.SkipAccountTokensCache,
    FlagDefault: false,
    Required:    false,
}
```

#### `internal/ingest/ingest.go`
- Added `SkipAccountTokensCache` field to `Configs` struct
- Modified `setupDeps()` to skip archive connection when flag is set
- Passed flag to `NewIngestService()`

#### `internal/services/ingest.go`
- Added `skipAccountTokensCache` field to `ingestService` struct
- Modified `NewIngestService()` to accept the new parameter
- Modified `Run()` to skip `PopulateAccountTokens()` when flag is set
- Gets start ledger from RPC's `OldestLedger` instead of using checkpoint

#### `docker-compose.local.yaml`
Updated ingest command:
```yaml
./wallet-backend ingest --skip-account-tokens-cache --checkpoint-frequency 8
```

Added environment variable:
```yaml
SKIP_ACCOUNT_TOKENS_CACHE: "true"
```

---

## 3. Ticketing DApp (connectiondemo)

### Contract ID Display - `src/pages/EventDetailPage.tsx`
Added display of the deployed contract ID under the Ticket Price component.

**Changes:**
```tsx
import { networks } from "ticketing";

const contractId = networks.standalone.contractId;

// In JSX (under price-options):
<div className="contract-info">
  <span className="contract-label">Contract:</span>
  <code className="contract-id">{contractId}</code>
</div>
```

### Contract ID Styling - `src/pages/EventDetailPage.css`
Added CSS for the contract info display:
```css
.contract-info {
  margin-top: 1rem;
  padding-top: 0.75rem;
  border-top: 1px solid var(--sds-clr-gray-04);
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.75rem;
}

.contract-label {
  color: var(--sds-clr-gray-09);
}

.contract-id {
  background: var(--sds-clr-gray-03);
  padding: 0.25rem 0.5rem;
  border-radius: 0.25rem;
  font-family: monospace;
  font-size: 0.7rem;
  color: var(--sds-clr-gray-11);
  word-break: break-all;
}
```

---

## 4. Deployed Contracts (after rebuild)

| Contract | Contract ID |
|----------|-------------|
| ticketing | `CDX7PH4GBUU446BURBNJCDIAR4J24P57L3562ESRBXYS2XGPZVW3FWNM` |
| guess_the_number | `CCKFJT2MK45KVLMKF4OXAXVXMFXNVCNDQWWVILVP7OB5TFJZ6BQQJT4Q` |
| nft_enumerable_example | `CDS4QTWGTCDD64O4RWGOYUTSLTJI5FTMF2DPK7NV7XRUI4GKREOGTSLM` |
| fungible_allowlist_example | `CDJGDDPJR6T37U3JPYYSABRODMCR5LG6D3HVACTAMWZA5DCQNSZIKGWT` |

### Events Created on Ticketing Contract
- Event 1: 500 capacity
- Event 2: 100 capacity
- Event 3: 50 capacity
- Event 4: 1000 capacity
- Event 5: 200 capacity

---

## 5. Running Services

| Service | URL | Status |
|---------|-----|--------|
| Ticketing DApp | http://localhost:5173 | Running |
| Block Explorer | http://localhost:3000 | Running |
| Wallet Backend API | http://localhost:8002 | Running (healthy) |
| Stellar Local Network | http://localhost:8000 | Running |
| Ingest Service | Port 8004 | Running (healthy) |

---

## Root Cause Analysis

### "Unauthorized" Error
The wallet-backend `.env` file had `CLIENT_AUTH_PUBLIC_KEYS` configured, requiring JWT authentication. The `docker-compose.local.yaml` removes this requirement, so running via docker-compose disables auth.

### Contract ID Mismatch
Old production builds in `connectiondemo/dist/` contained stale contract IDs from previous deployments. Cleaning the dist folder and vite cache resolves this.
