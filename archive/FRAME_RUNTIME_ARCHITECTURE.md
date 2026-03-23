# FRAME Runtime Architecture

## 1. Execution Pipeline Overview

The full execution loop follows this path:

```
User text input
  → conversationEngine.processUserMessage()
    → resolveIntent() [ai.parse_intent via engine]
      → agentPlanner.planExecution() [build execution graph]
        → agentPlanner.executeGraph() [execute steps in dependency order]
          → engine.handleIntent() [per step]
            → router.resolve() [find best dApp provider]
              → router.execute() [run dApp in sandbox]
                → buildReceipt() [Ed25519 signed, chain-linked]
                  → appendReceipt() [state update]
    → UI plan (optional) → uiPlanExecutor → widget rendering
  → display response to user
```

## 2. Intent Resolution

**File:** `ui/system/conversationEngine.js` — `resolveIntent(userText)`

1. Calls `engine.handleIntent('ai.parse_intent', { text, input })` to classify user text
2. If the AI dApp returns `{ intent: { resolved: true, action, payload } }`, uses that
3. Falls back to `ai.chat` for unrecognized or empty intents
4. System intents (`system.unknown`, `system.empty`, `ai.parse_intent`) are remapped to `ai.chat`

## 3. Execution Graph System

**File:** `ui/system/agentPlanner.js`

### Planning: `planExecution(intent, capabilities, context)`

- Maps action names to canonical capability names via `ACTION_TO_CANONICAL`
- Builds a step graph: `{ id, canonical, dependsOn?, args }`
- Multi-step graphs for compound operations (e.g. `wallet.send` → balance check then transaction)
- Single-step graphs for simple operations

### Execution: `executeGraph(engine, plan)`

- **Topological sort** (`topoSort`) resolves execution order honoring `dependsOn` declarations
- Detects cycles and missing dependencies before execution starts
- Maintains a **results map** keyed by step `id` (not just a single `prev` variable)
- Injects `$deps` (map of dependency results) and `$prev` (single-dep convenience) into step args
- Stops execution on first error, returning the error result
- Returns the last step's result on success

### Canonical Capability Names

| Action | Canonical |
|--------|-----------|
| wallet.balance | finance.balance |
| wallet.send | finance.transactions |
| wallet.mintFC | finance.mint |
| wallet.burnFC | finance.burn |
| notes.create | notes.create |
| notes.search | notes.search |
| ai.summarize | ai.summarize |
| ai.chat | ai.chat |
| context.query | system.status |
| view.open | navigation.open |

## 4. Engine Kernel

**File:** `ui/runtime/engine.js` — FRAME v2.2.0 Kernel

### `handleIntent(input)` pipeline:
1. `kernelNormalize` — normalize input to `{ action, payload }`
2. `normalizeViaAiDApp` — optional AI-assisted normalization
3. `validateIntent` — check action is a valid string
4. System intent handling (built-in actions like `system.status`)
5. `router.resolve(intent)` — find best dApp provider
6. Permission checks — verify dApp has declared required capabilities
7. Integrity lock — install deterministic sandbox
8. `router.execute(match, intent, buildScopedApi)` — run dApp
9. `buildReceipt(result)` — create signed execution receipt
10. `appendReceipt(receipt)` — append to chain log

### Capability Schema

Frozen taxonomy in `CAPABILITY_SCHEMA` defines all valid capabilities:
`bridge.burn`, `bridge.mint`, `storage.read`, `storage.write`, `identity.read`,
`wallet.send`, `wallet.read`, `wallet.balance`, `contacts.read`, `contacts.import`,
`messages.send`, `messages.list`, `display.timerTick`, `network.request`

### Deterministic Sandbox

`installDeterministicSandbox()`:
- Freezes `Date.now` to execution-start timestamp
- Seeds `Math.random` for deterministic output
- Blocks `setTimeout`, `setInterval`, `fetch`, `WebSocket` during dApp execution

## 5. Router & dApp Resolution

**File:** `ui/runtime/router.js`

### `resolve(intent)`:
1. Try `capabilityRegistry.getBestCapabilityByCanonical(canonical)` — scored provider lookup
2. Try AI intent matching (if ai dApp available)
3. Try manifest intent pattern matching across installed dApps
4. Return `{ dapp, match, manifest }` or null

### `execute(match, intent, buildScopedApi)`:
1. Verify dApp manifest declares required capabilities
2. Import dApp ES module
3. Build scoped API via `buildScopedApi(dappId, declaredCapabilities)`
4. Install async blockers (sandbox)
5. Call dApp's exported handler
6. Return result

## 6. Capability Registry

**File:** `ui/system/capabilityRegistry.js`

- Scans all installed dApp manifests at boot
- Builds indexes: `_index`, `_intentIndex`, `_typeIndex`, `_providersByCanonical`, `_graph`
- Provider scoring: `trust + (1/latency) - cost + intentBonus*0.3 + capBonus*0.3 + fedBonus*0.2 + repBonus*0.2`
- AI preference: local-first for `parse_intent` always; local for `chat` if <200 tokens; local for `summarize` if <2000 chars
- Auto-inferred capability graph via `outputMatchesInput` for pipeline composition

## 7. UI Planning & Widget Rendering

### UI Planner — `ui/dapps/ai/uiPlanner.js`

`planUI(intent, context, capabilities, widgets)`:
- Pattern-matches intent action/payload to widget configurations
- Returns `{ type: "ui.plan", layout, widgets: [{ widget, props, position }] }` or null
- Uses `scoreWidgets` from `relevance.js` for overview dashboards

### UI Plan Executor — `ui/system/uiPlanExecutor.js`

`executeUIPlan(plan)`:
- Standard plans (`type === "ui.plan"`): apply layout → destroy old widgets → spawn/update new widgets
- Legacy plans: create loading widget → create region → spawn widgets → cleanup
- Returns `{ success, spawned, total, errors }` for standard plans
- Widget spawn errors are caught individually; partial success is possible

### Widget Manager — `ui/system/widgetManager.js`

Full lifecycle: create, spawn, update, destroy, render, refresh.
14+ built-in data sources. Region-aware rendering with priority sorting.

### Layout Manager — `ui/system/layoutManager.js`

Layout persistence to localStorage. `applyLayout(name)` fetches `layouts/<name>.json`.
Region management: create, remove, list, cleanup empty regions.

## 8. Receipt & State Integrity

### Execution Receipts (v2)

Each engine execution produces a receipt with:
- `id`, `timestamp`, `action`, `dapp`
- `inputHash` — SHA-256 of canonical input
- `resultHash` — SHA-256 of canonical result
- `prevReceiptId` — chain link to previous receipt
- `signature` — Ed25519 signature via Tauri identity system

### State Root (v3)

**File:** `ui/runtime/stateRoot.js`

SHA-256 computed from:
- Current identity public key
- Installed dApps list (with code hashes)
- Storage state
- Receipt chain commitment (hash of latest receipt)

### Chain Verification

Engine verifies receipt chain integrity:
- Ed25519 signature validation
- Chain link continuity (`prevReceiptId` matches)
- No gaps or forks in the receipt sequence
