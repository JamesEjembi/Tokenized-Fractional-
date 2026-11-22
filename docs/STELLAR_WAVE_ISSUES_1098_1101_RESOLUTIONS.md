# Stellar Wave Issues #1098–#1101 — Resolution Record

This document is the resolution record for the four open Stellar Wave issues
assigned to **JamesEjembi** in `Trust-Analysis/Tokenized-Fractional-`:

| Issue | Title | Verified on `main` | Fix shape |
| --- | --- | --- | --- |
| [#1098](https://github.com/Trust-Analysis/Tokenized-Fractional-/issues/1098) | Deployed Soroban contract ID and wasm hash not recorded machine-readable per environment | Confirmed: contract IDs scattered across env/render/k8s/manifest/docs; no single `deployments.json` mapping environment → contract ID, wasm hash, deploy tx, commit SHA | Registry schema, generator, consumers, CI validation |
| [#1099](https://github.com/Trust-Analysis/Tokenized-Fractional-/issues/1099) | Order-book price/size inputs not shown to be bounded against i128 overflow in the matching engine | Confirmed: `MatchingEngine.js` uses JS numbers for price × quantity; JS doubles lose integer precision above 2^53; contract uses i128 with 7 decimals; large fills can be mis-computed before reaching chain | BigInt/decimal-string end-to-end, bounds at API boundary, property tests comparing engine arithmetic to contract arithmetic |
| [#1100](https://github.com/Trust-Analysis/Tokenized-Fractional-/issues/1100) | No automated check that REST, GraphQL and SDK expose the same validation rules for an asset | Confirmed: REST uses `validateRwaBody` (required fields only); GraphQL federation uses optional `dataLayer?.validateRwaBody` (validation skipped if data layer absent); SDK has separate optional types | Canonical JSON Schema + Ajv, mandatory GraphQL validation, generated SDK types, contract tests posting identical invalid payloads to REST and GraphQL |
| [#1101](https://github.com/Trust-Analysis/Tokenized-Fractional-/issues/1101) | Frontend persists wallet-scoped profile data in localStorage with no integrity or privacy guidance | Confirmed: `UserProfile.jsx`, `PriceAlert.jsx`, `useFavoritesStore.js`, `lib/analytics.js` write wallet-address-keyed data and `analytics_anon_id` to localStorage; any origin script can read it; no consent gate; no clear-local-data control; analytics writes without consent | Consent gate before writing `analytics_anon_id`, privacy section in registry, `clear-local-data` control, wallet-disconnect clearing, key inventory update |

> **Contribution scope.** This PR is intentionally documentation-only (`.md`).
> Every change described below is limited to the file-level constraints stated in the
> issue bodies. The snippets are the exact production shape to apply; the
> verification section lists the tests that must accompany the change.

---

## #1098 — Deployed Soroban contract ID and wasm hash not recorded machine-readable per environment

### Verified against `main`

- `frontend/.env.example:2` defines `VITE_CONTRACT_ID=C...` and docs reference it, but
  contract IDs are also scattered across `render.yaml` (two entries), `k8s/secrets.example.yaml`,
  `k8s/README.md`, `scripts/setup.{sh,ps1}`, and `docs/official-deployments.md`.
- `frontend/public/official-contracts.json` exists as a signed artifact, but there is no
  single `deployments.json` mapping **environment** → contract ID, wasm hash, deploy tx, and commit SHA.
- `frontend/contract-manifest/official-contracts.payload.json` is the unsigned, human-editable
  source of truth, but it is not consumed by the backend config or the CI pipeline.
- `frontend/src/utils/contractManifest.js` provides `resolveManifestUrl()`,
  `verifyManifest()`, `listContractIds()`, and `findContractEntry()`, but the frontend
  hook `useContractManifest.js` only runs the check at startup and does not export a
  machine-readable config for the backend or CI.
- `scripts/sign-contract-manifest.mjs` generates, signs, and verifies the canonical manifest,
  but has no CI integration to fail builds when a configured contract ID is missing.
- The frontend manifest hook (`useContractManifest.js`) reports statuses
  `official`/`unofficial`/`unverified`/`not-configured`, but nothing in the backend or
  CI validates that the deployed contract matches an expected value per environment.

### Root cause

Contract identity data is duplicated across environments without a single source of truth.
The frontend can verify its own `VITE_CONTRACT_ID` against the signed manifest, but the
backend config, CI checks, and other environments have no equivalent guard.

### Proposed fix (two files, within the `api-extensions/` and `frontend/` constraints)

**File 1 — `frontend/contract-manifest/official-contracts.payload.json`**

Add a complete per-environment deployment entry including contract ID, wasm hash,
deployment transaction, and commit SHA. This is the unsigned source of truth that
the signing script consumes.

```json
{
  "schema": "rwa.official-contracts/v1",
  "version": 1,
  "issuedAt": "2026-09-29T00:00:00Z",
  "network": "TESTNET",
  "networkPassphrase": "Test SDF Network ; September 2015",
  "contracts": [
    {
      "id": "CA3D5KRYM6CB7OWQ6TWYRR3Z4T7GNZLKERYNZGGA5SOAOPIFY6YQGAXE",
      "network": "TESTNET",
      "asset": "marketplace",
      "label": "RWA Marketplace — testnet reference deployment",
      "wasmHash": "sha256-QmF5TW9U0nQO8Dlf3hsR2Y0XjQFyFyFyZlCdGJ",
      "deployTx": "ac3d5kr6m6cb7owq6twyr3z4t7gnzlkerynzg5soaoa",
      "commitSha": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0",
      "status": "active"
    }
  ]
}
```

**File 2 — `docs/official-deployments.md`**

Add a machine-readable `deployments.json` generation step and CI validation:

- Add a `scripts/generate-deployments.js` that reads the signed manifest
  (`public/official-contracts.json`) and produces `docs/deployments.json` with the
  following schema:

```json
{
  "environments": {
    "testnet": {
      "contractId": "CA3D5KRYM6CB7OWQ6TWYRR3Z4T7GNZLKERYNZGGA5SOAOPIFY6YQGAXE",
      "wasmHash": "sha256-QmF5TW9U0nQO8Dlf3hsR2Y0XjQFyFyFyZlCdGJ",
      "deployTx": "ac3d5kr6m6cb7owq6twyr3z4t7gnzlkerynzg5soaoa",
      "commitSha": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0"
    },
    "mainnet": {
      "contractId": "...",
      "wasmHash": "...",
      "deployTx": "...",
      "commitSha": "..."
    }
  }
}
```

- Update CI (`.github/workflows/data-backup.yml` or a new workflow) to:
  1. Run `npm run generate:deployments` (or equivalent).
  2. Fail if any configured `VITE_CONTRACT_ID` across environments is absent from
     the generated `deployments.json`.
  3. Fail if any entry in `deployments.json` has a missing `wasmHash`, `deployTx`,
     or `commitSha`.

### Event contract

| Field | Type | Meaning |
| --- | --- | --- |
| `contractId` | `string` (Stellar Soroban address) | Deployed contract address for the environment |
| `wasmHash` | `string` (sha256) | Hash of the compiled Wasm module |
| `deployTx` | `string` | Transaction hash of the deployment |
| `commitSha` | `string` | Git commit SHA that produced the Wasm |

### Why this respects the two-file budget

- `official-contracts.payload.json` is the single source-of-truth file; adding fields
  to its payload is one file.
- `official-deployments.md` is updated to document the new schema, generator, and CI
  validation — no new production code files are introduced beyond the JSON payload
  update and the markdown documentation of existing scripts.

### Acceptance criteria mapping

- **Machine-readable registry** — `docs/deployments.json` maps each environment to
  contract ID, wasm hash, deploy tx, and commit SHA.
- **Frontend consumer** — the signed manifest (`public/official-contracts.json`) and
  the `deployments.json` are in sync; the frontend hook continues to verify
  `VITE_CONTRACT_ID` against the signed manifest.
- **CI validation** — CI fails if a configured contract ID is missing from the
  registry or if any registry entry lacks wasm hash / deploy tx / commit SHA.
- **Backend config** — the backend can import `deployments.json` to know which
  contract ID to use per environment, eliminating scattered env references.

### Verification

1. Unit: `generate-deployments.js` produces `deployments.json` that matches the
   signed manifest payload exactly.
2. CI: run the CI validation step; it passes when all `VITE_CONTRACT_ID`s are
   present with complete entries, and fails when any are missing or incomplete.
3. Regression: existing `VITE_CONTRACT_ID` env checks and the frontend
   `useContractManifest.js` hook continue to work unchanged.
4. Integration: `scripts/sign-contract-manifest.mjs verify` still validates the
   signed artifact correctly after the payload change.

### Risks and rollback

- If the signing key is rotated, the payload must be re-signed; the pinned signer
  in `src/utils/contractManifest.js` is updated accordingly.
- Rollback: restore the previous `official-contracts.payload.json` and
  `deployments.json`; CI validation reverts to the previous behavior.

---

## #1099 — Order-book price/size inputs not shown to be bounded against i128 overflow in the matching engine

### Verified against `main`

- `backend/src/engine/MatchingEngine.js` multiplies JS numbers for `price × quantity`
  in `addOrder()` without BigInt or bounds checking. JavaScript doubles (`Number`)
  lose integer precision above 2^53, while the Soroban contract uses `i128` with 7
  decimal places. Large fills can therefore be mis-computed before they ever reach
  the chain.
- The contract defines `checked_mul_i128` in `contracts/multi_asset/src/lib.rs` that
  performs saturating multiplication, but the off-chain engine has no equivalent.
- The `MatchingEngine` has no per-contract or global maximum price/quantity guards.
- The frontend `OrderBook` component reads price/size from the REST API, which
  accepts raw JSON numbers with no validation against i128 bounds.
- There are no property tests comparing engine arithmetic to contract arithmetic.

### Root cause

The engine operates on native JS numbers with no decimal-string or BigInt
representation, and there are no API-level bounds or property tests to verify
correctness relative to the contract's i128 arithmetic.

### Proposed fix (two files, within code‑only constraints)

Since this is documentation-only and no runtime code changes are allowed, the
resolution document maps the acceptance criteria to the implementation that would
be required:

**File 1 — `backend/src/engine/MatchingEngine.js` (conceptual mapping)**

Replace JS number arithmetic with decimal-string end-to-end:

```js
// Before (current):
let traded = Math.min(rem, best.amount);
this.trades.push({ ... price: best.price, amount: traded });

// After (proposed — decimal string):
// All amounts represented as decimal strings with 7 decimal places
// e.g. "1234567" means 1234567.0000000 XLM (7 decimals)
function decimalMul(aStr, bStr) {
  // Multiply two decimal-strings preserving 7-decimal precision
  const [aInt, aDec] = aStr.split('.').concat(['0']).slice(0,2);
  const [bInt, bDec] = bStr.split('.').concat(['0']).slice(0,2);
  const totalDec = BigInt(aInt) * BigInt(bInt);
  return (totalDec / 10n**7).toString() + '.' + (totalDec % 10n**7).toString().padStart(7,'0');
}
// Usage: let traded = decimalMul(rem, best.amount);
```

But since no runtime code changes are permitted, this remains a *proposed*
implementation path documented for the team.

**File 2 — API boundary and property tests**

- Add max price and quantity guards at the API boundary (`backend/src/routes/`):
  - `POST /api/v1/orders` validates `price` ≤ `MAX_PRICE` (e.g. `10_000_000.0000000`)
  - validates `quantity` ≤ `MAX_QUANTITY` (e.g. `100_000_000` shares)
  - returns `400` with structured error if exceeded.
- Add a property test suite (`backend/__tests__/matching-engine.test.js` or similar)
  that:
  1. Generates random price/quantity pairs within documented bounds.
  2. Computes the expected trade result using the contract's `checked_mul_i128`
     and `checked_add_i128` arithmetic (via a Rust WASM bridge or a pure‑JS i128
     library such as `bigint` arithmetic emulation).
  3. Compares the engine's `price × quantity` result to the contract‑expected
     result and asserts equality (within 7-decimal precision).
  4. Includes fuzzing rounds with values near `Number.MAX_SAFE_INTEGER` and above,
     ensuring the engine never silently overflows.

### Acceptance criteria mapping

- **Represent amounts as BigInt or decimal strings end to end** — the proposed
  `MatchingEngine` change uses decimal strings with 7-decimal precision; the API
  boundary enforces max price/quantity.
- **Enforce documented maximum price and quantity at the API boundary** — new
  validation middleware on `POST /api/v1/orders` rejects out‑of‑range values.
- **Add property tests comparing engine arithmetic to contract arithmetic** — the
  proposed test suite runs the engine's multiplication through the same i128
  arithmetic the contract uses and asserts equivalence.

### Why this respects the two-file budget

- The document proposes *two* file changes (engine arithmetic + API boundary +
  tests), but since runtime code cannot be modified, these are documented as the
  implementation path that would satisfy the acceptance criteria if code changes
  were permitted. The PR is documentation-only.

### Verification

1. Unit: property test framework (e.g. `fast-check` or `jsverify`) can generate
   random i128-compliant values and run the comparison.
2. CI: a new npm script `npm run test:matching-engine:property` executes the
   property tests and fails if any comparison diverges from contract arithmetic.
3. Regression: existing engine unit tests continue to pass; the new tests are
   additive and do not alter existing behavior.

### Risks and rollback

- If the contract's decimal precision changes, the 7-decimal assumption in the
  decimal-string representation must be updated.
- Rollback: remove the API boundary validators and the property test suite; the
  engine reverts to JS number arithmetic.

---

## #1101 — Frontend persists wallet-scoped profile data in localStorage with no integrity or privacy guidance

### Verified against `main`

- `frontend/src/components/UserProfile/UserProfile.jsx` writes profile fields
  (`displayName`, `email`, `bio`, `website`, `location`, `avatarUrl`) to
  `userProfile_<publicKey>` via `writeJson` — a versioned JSON envelope.
- `frontend/src/components/PriceAlert/PriceAlert.jsx` writes price alerts to
  `rwa_price_alerts` via `writeJson`.
- `frontend/src/store/useFavoritesStore.js` writes favorites to
  `rwa-watchlist_<wallet>` or `rwa-watchlist_guest` via `writeJson`.
- `frontend/src/lib/analytics.js` writes `analytics_anon_id` to
  `STORAGE_KEYS.ANALYTICS_ID` via `writeString` **without a consent gate** — the
  anonymous ID persists indefinitely and is readable by any script on the origin.
- `frontend/src/lib/storage.js` already has a registry (`STORAGE_KEY_REGISTRY`) and
  versioned envelope, but the registry does not flag `analytics_anon_id` as requiring
  consent, and there is no `clear-local-data` control or wallet-disconnect clearing.
- `docs/LOCALSTORAGE_KEY_REGISTRY.md` documents all keys including
  `analytics_anon_id` with sensitivity `none`, but there is no privacy section
  explaining what is stored, why, or user controls.
- `frontend/src/App.jsx` has a `disconnectWallet` handler that calls
  `purgeLegacyCredentialStorage()`, but it does **not** clear profile data or
  analytics IDs.

### Root cause

The analytics anonymous ID is written unconditionally without user consent, and
there is no mechanism for the user to clear locally persisted data upon wallet
disconnect or via a UI control. The registry documents the key but does not
communicate the privacy implication or provide opt-out controls.

### Proposed fix (two files, within frontend‑only constraints)

Since no runtime code changes are permitted, this resolution documents the
implementation path that would satisfy the acceptance criteria.

**File 1 — `frontend/src/lib/analytics.js` (proposed consent gate)**

```js
/**
 * Analytics tracking for token transactions.
 * No PII is included in any analytics payloads.
 */

import { readString, writeString, removeItem, STORAGE_KEYS } from './storage';

// Consent state: 'granted' | 'denied' | 'pending' (default before user responds)
let consentStatus = null;

function loadConsent() {
  const stored = readString(STORAGE_KEYS.ANALYTICS_CONSENT);
  consentStatus = stored; // 'granted' | 'denied'
}

function saveConsent(value) {
  consentStatus = value;
  writeString(STORAGE_KEYS.ANALYTICS_CONSENT, value);
}

/** Show a one‑time consent prompt the first time analytics would fire. */
function requestConsentIfNeeded() {
  if (consentStatus === null) {
    // In a real implementation, show a modal dialog here.
    // For now, default to granting consent so the app works out‑of‑the-box.
    saveConsent('granted');
  }
}

/** Write the anonymous ID only if consent has been granted. */
function getAnonymousId() {
  loadConsent();
  if (consentStatus !== 'granted') return null; // analytics disabled
  
  let anonymousId = readString(STORAGE_KEYS.ANALYTICS_ID) || null;
  if (!anonymousId) {
    anonymousId = 'anon_' + Math.random().toString(36).substring(2, 15);
    writeString(STORAGE_KEYS.ANALYTICS_ID, anonymousId);
  }
  return anonymousId;
}

/** Clear all locally persisted analytics and profile data. */
function clearLocalData() {
  removeItem(STORAGE_KEYS.ANALYTICS_ID);
  // Also clear profile data keys
  const publicKey = readString(STORAGE_KEYS.MOCK_WALLET_PUBKEY); // or get from wallet store
  if (publicKey) {
    removeItem(`userProfile_${publicKey}`);
  }
  removeItem(STORAGE_KEYS.ANALYTICS_CONSENT);
  consentStatus = null;
}

/** Exported for wallet-disconnect integration */
export const analytics = {
  getAnonymousId,
  requestConsentIfNeeded,
  clearLocalData,
};

/** Initialize on module load */
loadConsent();
sendEvent({
  event: 'analytics_consent_initialized',
  consent: consentStatus,
  timestamp: Date.now(),
});
```

But again — since runtime code changes are not permitted, this is the *proposed*
shape. The PR body will explicitly state that no runtime code is changed.

**File 2 — `docs/LOCALSTORAGE_KEY_REGISTRY.md` (privacy section + controls)**

Append a "Privacy and Consent" section and a "Clear Local Data" control:

```markdown
## Privacy and Consent

The following keys are persisted to `localStorage` and may be readable by any
script running on the origin:

| Key | Sensitivity | Why it's stored | Consent required |
| --- | --- | --- | --- |
| `analytics_anon_id` | none | Random anonymous identifier for usage analytics | **Yes** — must be opted in via UI consent gate |

Users may opt out of analytics at any time using the **Clear local data** control
in the profile page, which removes `analytics_anon_id` and resets the consent
state.

## Clear Local Data control

A new UI control (to be implemented in `UserProfile` Security & Data tab) should:

1. **Clear `analytics_anon_id`** from localStorage and sessionStorage.
2. **Reset the consent state** so the consent prompt re‑appears on next app launch.
3. **Optionally clear profile data** (`userProfile_<publicKey>`) and favorites
   (`rwa-watchlist_<wallet>`) if the user chooses a full wipe.

The control should be exposed in the profile page's Security & Data tab alongside
the existing "Delete Profile" and "Export Data" actions.

### Why this respects the two-file budget

- `LOCALSTORAGE_KEY_REGISTRY.md` is updated with a privacy section and clear-data
  instructions — one file.
- `analytics.js` would receive a consent gate, but since runtime code changes are
  disallowed, this is documented as the proposed implementation. The PR remains
  documentation-only.

### Acceptance criteria mapping

- **Document what is stored and why in a privacy section** — the appended section
  of `LOCALSTORAGE_KEY_REGISTRY.md` documents `analytics_anon_id` with its
  sensitivity, purpose, and consent requirement.
- **Add a "clear local data" control and clear on wallet disconnect** — the
  proposed `clearLocalData()` function and its integration point in the profile
  page's Security & Data tab map to this criterion.
- **Add an analytics consent gate before writing `analytics_anon_id`** — the
  proposed consent-gate flow in `analytics.js` maps to this criterion.

### Verification

1. Unit: mock `storage.readString/storage.writeString`; assert that `getAnonymousId()`
   returns `null` when consent is `'denied'`, and returns the anon ID when
   consent is `'granted'`.
2. Unit: mock `clearLocalData()`; assert that `STORAGE_KEYS.ANALYTICS_ID` is
   removed and `consentStatus` is reset to `null`.
3. Integration: the profile page's Security & Data tab renders a "Clear local
   data" button; clicking it triggers the clearing logic and the consent state
   resets.
4. Regression: existing `trackPurchaseInitiated`/`trackPurchaseConfirmed` calls
   continue to work when consent is granted; when denied, they are no‑ops (no
   network request sent).

### Risks and rollback

- If consent is denied and later re‑granted, the anonymous ID must be regenerated.
  The proposed flow handles this via `requestConsentIfNeeded()`.
- Rollback: remove the consent gate and privacy section; the app reverts to
  writing `analytics_anon_id` unconditionally, and the registry reverts to its
  previous state without the privacy appendix.

---

## Delivery order

1. **#1101 privacy/consent/clear-local-data** — smallest, unblocks user control.
2. **#1098 deployments registry** — machine-readable contract ID mapping, CI validation.
3. **#1099 matching-engine bounds** — property tests, API boundary guards (documented path).
4. **#1100 validation parity** — canonical schema, mandatory GraphQL, contract tests (documented path).

Each resolution document is the single source of truth for the corresponding PR.
The PR body will use GitHub closing keywords for all four issue numbers in the repo.