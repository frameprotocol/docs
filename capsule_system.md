# FRAME Capsule System

Capsules are portable execution units that allow data, compute tasks, and capabilities to be securely transported and executed across FRAME nodes. They integrate with the engine, router, peer router, p2p transport, capability registry, intent market, wallet/credit ledger, and context memory without breaking deterministic execution or the receipt chain.

---

## 1. Capsule Structure

**File:** `ui/system/capsule.js`

Canonical capsule format (canonical JSON only, signed by origin identity, hashable, deterministic):

```js
{
  id: string,
  version: number,
  timestamp: number,
  originIdentity: string,
  requestedCapability: string,
  payload: object,
  paymentRule: object | null,
  privacyRule: string,
  executionPolicy: object | null,
  next: array | null,       // optional: [{ requestedCapability, payloadTemplate }] for chaining
  chainDepth: number | null,  // optional: 0..5, incremented each spawn
  parentCapsule: string | null, // optional: id of parent in workflow
  signature: string | null
}
```

**Exposed:** `window.FRAME_CAPSULE`

| Function | Description |
|----------|-------------|
| `createCapsule(opts)` | Build unsigned capsule; caller then calls `signCapsule`. |
| `signCapsule(capsule)` | Sets `capsule.id` (hash) and `capsule.signature` via `__FRAME_INVOKE__('sign_data', { data: id })`. |
| `hashCapsule(capsule)` | Returns Promise of SHA-256 hex of canonical signable payload. |
| `verifyCapsule(capsule)` | Returns Promise of `{ valid, reason?, signatureVerified? }`. Id must match hash; optional Tauri `verify_capsule_signature`. |
| `validateCapsule(capsule)` | Sync validation: requestedCapability, privacyRule, paymentRule shape, executionPolicy shape. |

Signable payload (for hashing/signing) excludes `id` and `signature`: version, timestamp, originIdentity, requestedCapability, payload, paymentRule, privacyRule, executionPolicy.

---

## 2. Capsule Storage

**File:** `ui/system/capsuleStore.js`  
**Storage:** localStorage key `frame_capsules`. Max 500 capsules; prune via `pruneCapsules`.

**Exposed:** `window.FRAME_CAPSULE_STORE`

| Function | Description |
|----------|-------------|
| `addCapsule(capsule)` | Append record { id, timestamp, originIdentity, requestedCapability, status, capsule }; returns boolean. |
| `getCapsule(id)` | Return full capsule or null. |
| `getRecord(id)` | Return store record (includes status). |
| `setStatus(id, status)` | Set status; valid: pending, announced, executing, completed, failed. |
| `listCapsules(filter)` | Filter by status, originIdentity, requestedCapability; return index entries. |
| `removeCapsule(id)` | Remove by id. |
| `pruneCapsules(options)` | Remove old; options: maxAgeMs, keepStatuses. |

**Lifecycle states:** pending → announced → (executing) → completed | failed.

---

## 3. Capsule Network

**File:** `ui/system/capsuleNetwork.js`

Uses existing p2pTransport message types: `capsule_announce`, `capsule_request`, `capsule_data`.

**Exposed:** `window.FRAME_CAPSULE_NETWORK`

| Function | Description |
|----------|-------------|
| `announceCapsule(capsule)` | Store, set status announced, send `capsule_announce` with capsuleIds to all peers. |
| `requestCapsule(peerId, capsuleId)` | Send `capsule_request` to peer. |
| `requestCapsules(peerId, capsuleIds)` | Same for multiple ids. |
| `sendCapsule(peerId, capsule)` | Send `capsule_data` with full capsule (origin responds to request). |
| `receiveCapsule(capsule)` | Verify, validate, store, optionally execute if `canExecuteCapsule`; returns `{ imported, capsuleId? }`. |

**Exposed for p2pTransport:** `window.FRAME_CAPSULES`

| Function | Used by p2pTransport |
|----------|----------------------|
| `receiveCapsuleAnnounce(msg)` | On `capsule_announce`; returns `{ capsuleIds }` to request (filter by capability if msg.capsuleSummaries). |
| `handleCapsuleRequest(capsuleIds)` | On `capsule_request`; returns array of full capsules from store. |
| `importCapsule(capsule)` | On `capsule_data`; same as receiveCapsule. |
| `announceCapsules()` | For broadcast; returns `{ capsuleIds }` (pending/announced). |

**Flow:** Node A create → store → announce → peers receive announce → peers send request → A sends capsule_data → peer importCapsule → verify → store → optionally execute.

---

## 4. Capsule Execution

**File:** `ui/system/capsuleExecutor.js`

Execution **only** via `FRAME.handleIntent(intent)`. No bypass of router or permissions; receipts remain consistent.

**Exposed:** `window.FRAME_CAPSULE_EXECUTOR`

| Function | Description |
|----------|-------------|
| `canExecuteCapsule(capsule)` | Async: validate, verify signature, capability exists (`getBestCapability(requestedCapability)`), privacy (restricted ⇒ current identity must match origin), payment rule check. Returns `{ canExecute, reason? }`. |
| `executeCapsule(capsule)` | Set status executing; canExecute check; build intent from `capsule.payload` (action, payload); `FRAME.handleIntent(intent)`; on success set completed, apply payment (mintCredits for executor); on error set failed. |
| `capsuleToIntent(capsule)` | `payload.action` or requestedCapability; `payload.payload` or `payload.text`. |

**Intent shape:** `{ action, payload, raw: 'capsule:' + capsule.id, timestamp }`.

---

## 5. Payment Rule

Example: `{ type: "credit", amount: 5 }`.

- **Executor:** After successful execution, if `paymentRule.type === 'credit'` and `amount > 0`, executor calls `FRAME_CREDIT_LEDGER.mintCredits(amount, 'capsule:' + capsule.id)`. Origin’s spend is out-of-band (e.g. pre-lock or separate settlement).
- **canExecuteCapsule:** If payment rule has `requirePrepay`, checks ledger balance ≥ amount.

---

## 6. Privacy Rule

- **public:** Any node can execute (subject to capability match).
- **restricted:** Only origin identity (current identity matches `capsule.originIdentity`) or future permission grant can execute.
- **encrypted:** Reserved; not implemented in this version.

---

## 7. Execution Policy

Example: `{ maxRetries: 2, timeout: 10000, preferredLocality: "ring1" }`.

- **peerRouter** can use `preferredLocality` when selecting peers for routing (future: route capsule announce to ring1 first).
- **timeout / maxRetries:** Can be used by executor or network layer for retries and timeouts (application-specific).

---

## 8. Capability Matching

On receive: `capsule.requestedCapability` is resolved with `FRAME_CAPABILITY_REGISTRY.getBestCapability(capsule.requestedCapability)`. If a capability exists locally, the node may accept and execute the capsule (after verify, payment, privacy checks).

---

## 9. Engine Integration

- **No change to core deterministic logic.** Capsule executor calls `FRAME.handleIntent(intent)` only. All receipts and state changes go through the existing pipeline.
- Optional future: engine could expose a “capsule intent” entry point that builds intent from a verified capsule and runs the same path.

---

## 10. UI Widget

**Capsule Activity** widget:

- **Data source:** `capsule.activity` (in `widgetManager.js` DATA_SOURCES).
- **Shows:** pending, executing, completed counts; list of capsules (requestedCapability / id slice, status).
- **Default widget:** `default_capsule_activity`, data source `capsule.activity`, region sidebar, priority 55.

---

## 11. Capsule Chaining and Workflow Execution

A capsule may declare optional **next** steps. After successful execution, the executor spawns follow-up capsules for each step, creating distributed workflows (e.g. AI analyze → report → store → notify).

### Workflow chains

- **next** — Optional array of `{ requestedCapability, payloadTemplate }`. Each step becomes a new capsule after the current one completes.
- **payloadTemplate** — Object that may contain `$result.*` placeholders; see Template substitution.
- **chainDepth** — Optional number 0..5. Root capsules have 0 (or omit). Each spawned capsule has `chainDepth = parent.chainDepth + 1`. Spawn is skipped if depth would exceed 5.
- **parentCapsule** — Optional id of the capsule that spawned this one. Set on each child for workflow tracing.

### Template substitution

**File:** `ui/system/capsuleTemplates.js`

- **applyTemplate(template, result)** — Recursively substitutes values in `template`. Any string equal to `"$result"` or starting with `"$result."` is replaced: the remainder after `$result.` is a path into `result` (e.g. `$result.report` → `result.report`, `$result.summary` → `result.summary`). Nested objects and arrays are processed recursively. Used when building child payloads from the execution result.

### Safety limits

- **Max chain depth = 5.** If `(capsule.chainDepth || 0) >= 5`, no follow-up capsules are spawned. `validateCapsule` rejects `chainDepth` outside 0..5.
- Prevents unbounded or infinite chains.

### Parent capsule tracking

- Every spawned capsule has **parentCapsule** set to the executing capsule’s id. The store’s `listCapsules` includes `parentCapsule` and `chainDepth` when present on the stored capsule. The **Workflow Activity** widget uses this to show parent → child and status.

### Spawn logic (capsuleExecutor.js)

After `FRAME.handleIntent(intent)` succeeds:

1. If `capsule.next` exists and `(capsule.chainDepth || 0) < 5`:
2. For each step in `capsule.next`: build payload with `applyTemplate(step.payloadTemplate, result)`; create capsule with same `originIdentity`, `paymentRule`, `privacyRule`, `executionPolicy`, `chainDepth: depth + 1`, `parentCapsule: capsule.id`; sign and announce.

Agents may include **next** when creating capsules (e.g. context analyzer → ai.summarize with `payloadTemplate: { action: "ai.summarize", analysis: "$result.analysis" }`).

### Widget: Workflow Activity

- **Data source:** `capsule.workflows`. Uses `FRAME_CAPSULE_STORE.listCapsules()` (with parentCapsule/chainDepth); each row: parent id → child id (or “(root)”), status. Sorted by timestamp descending.
- **Default widget:** `default_workflow_activity`, title “Workflow Activity”, data source `capsule.workflows`, sidebar, priority 54.

---

## 12. Capsule Scheduling and Capability Market

Nodes advertise capabilities (id, price, latency, locality) and maintain a table of remote providers. The capsule scheduler chooses the best executor (local or peer) before any broadcast.

### Capability advertisement

**File:** `ui/system/capabilityAdvertiser.js`

- **announceCapabilities()** — Async. Scans `FRAME_CAPABILITY_REGISTRY.getCapabilities()`, builds message `{ type: 'capability_advertisement', nodeId, capabilities: [{ capabilityId, price, estimatedLatency, locality }], timestamp }`, sends to all peers via P2P. Locality from registry: `local` or `ring2` (for remote).
- **receiveCapabilityAdvertisement(msg)** — Called by p2pTransport when `capability_advertisement` is received; updates `FRAME_CAPABILITY_TABLE` per capabilityId.
- **getCapabilityProviders(capabilityId)** — Returns providers from table (optional id for one capability).
- **startAnnounceLoop()** / **stopAnnounceLoop()** — Fire announce once, then every 60s.

**File:** `ui/system/capabilityTable.js`

- **Storage:** localStorage key `frame_capability_table`. Structure: capabilityId → array of `{ nodeId, price, latency, locality, lastSeen }`.
- **updateProviders(capabilityId, providers)** — Merge providers for that capability; refresh lastSeen.
- **getProviders(capabilityId)** — Return list for capability (entries older than 5 minutes excluded).
- **getAllProviders()** — Return all capabilities with valid provider lists.
- **pruneProviders()** — Remove entries older than 5 minutes.

**P2P:** New message type `capability_advertisement` in `p2pTransport.js`; handler calls `FRAME_CAPABILITY_ADVERTISER.receiveCapabilityAdvertisement(msg)`.

### Capsule scheduler

**File:** `ui/system/capsuleScheduler.js`

- **chooseExecutor(capsule)** — 1) Check local capability via `getBestCapabilitySync(capsule.requestedCapability)`. 2) Get remote providers from table. 3) Score local: `price_weight * price + latency_weight * latency + locality_weight * locality_distance` (weights 1, 0.5, 0.3; locality: local=0, ring1=1, ring2=2, global=3). 4) If local capable and local score ≤ best remote score → `{ nodeId: 'local' }`. 5) Else if best remote → `{ nodeId: remoteNodeId }`. 6) Else if local capable → `{ nodeId: 'local' }`. 7) Else `{ nodeId: null }`.
- **scheduleCapsule(capsule)** — Call chooseExecutor. If local: `FRAME_CAPSULE_EXECUTOR.executeCapsule(capsule)`. If remote: `FRAME_CAPSULE_NETWORK.sendCapsule(nodeId, capsule)`. Return `{ scheduled: true, executor }` or `{ scheduled: false }`.

### Integration with announce

**capsuleNetwork.js:** `announceCapsule(capsule)` now calls `FRAME_CAPSULE_SCHEDULER.scheduleCapsule(capsule)` first. If `scheduled`, return (no broadcast). Only if not scheduled does it broadcast `capsule_announce` to all peers.

### Boot

On node boot (`app.js`): `FRAME_CAPABILITY_ADVERTISER.announceCapabilities()` then `startAnnounceLoop()` (repeat every 60s).

### Widget: Capability Market

- **Data source:** `capability.providers`. Uses `FRAME_CAPABILITY_TABLE.getAllProviders()`; each row: capabilityId, provider count, min price, min latency.
- **Default widget:** `default_capability_market`, title "Capability Market", sidebar, priority 58.

---

## 13. Security

- **Verify signature** before store/execute (`verifyCapsule`).
- **Verify capability** available locally before execute (`getBestCapability`).
- **Enforce payment rule** (mint on executor after success; optional requirePrepay check).
- **Enforce privacy** (restricted ⇒ origin match).
- **Never bypass permission system or router** — execution only via `handleIntent`.

---

## 14. Script Load Order

In `index.html`, after `peerRouter.js`, before `networkSimulator.js`:

1. `capsule.js`
2. `capsuleStore.js`
3. `capabilityTable.js`
4. `capabilityAdvertiser.js`
5. `capsuleScheduler.js`
6. `capsuleTemplates.js`
7. `capsuleExecutor.js`
8. `capsuleNetwork.js`

Then `p2pTransport.js` (which uses `window.FRAME_CAPSULES` and handles `capability_advertisement`).

---

## 15. File Summary

| File | Purpose |
|------|---------|
| `ui/system/capsule.js` | Canonical format, create, hash, sign, verify, validate. |
| `ui/system/capsuleStore.js` | Local storage, index, status, prune. |
| `ui/system/capabilityTable.js` | Remote capability providers per capabilityId; 5min TTL. |
| `ui/system/capabilityAdvertiser.js` | Scan registry, send capability_advertisement, receive and update table. |
| `ui/system/capsuleScheduler.js` | chooseExecutor (score: price, latency, locality), scheduleCapsule (local execute or send to peer). |
| `ui/system/capsuleTemplates.js` | applyTemplate(template, result) for $result.* substitution in chained payloads. |
| `ui/system/capsuleNetwork.js` | Announce via scheduler first; broadcast only when no capable node. |
| `ui/system/capsuleExecutor.js` | canExecuteCapsule, executeCapsule via handleIntent; spawn next-step capsules when capsule.next and depth < 5. |

Integration points: **p2pTransport** (capsule_*, capability_advertisement), **peerRouter** (getPeers for send), **capabilityRegistry** (getCapabilities, getBestCapabilitySync), **creditLedger** (mintCredits), **engine** (handleIntent only), **widgetManager** (capsule.activity, capability.providers, capsule.workflows).
