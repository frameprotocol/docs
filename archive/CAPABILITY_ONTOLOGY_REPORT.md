# FRAME Capability Ontology & Identity-Scoped Permissions

## 1. Capability ontology design

FRAME moves from ad-hoc capability strings (e.g. `wallet.balance`) to a **canonical capability ontology** so that:

- Many dApps can provide the same semantic capability (e.g. `finance.balance`) without name collision.
- The planner and runtime reason about **what** a capability does, not which dApp implements it.
- Provider selection is by ranking (reliability, cost, latency) instead of by name.

**Canonical capability names** are semantic identifiers. Examples:

| Canonical            | Meaning                         |
|----------------------|----------------------------------|
| `finance.balance`   | Returns current wallet/account balance |
| `finance.transactions` | Send or list transactions       |
| `navigation.route`   | Route or open a view            |
| `communication.message.send` | Send a message           |
| `notes.create`       | Create a note                   |
| `notes.search`       | Search notes                    |
| `ai.summarize`       | Summarize text                  |
| `system.status`      | System/context status           |
| `network.peers`      | Peer/connection info            |

Each **provider** (dApp) declares how it maps to the ontology: e.g. `wallet.balance` → `finance.balance`.

---

## 2. Canonical capability system

### 2.1 Capability metadata (manifest / schema)

A capability can be declared with full ontology fields:

```json
{
  "name": "wallet.balance",
  "canonical": "finance.balance",
  "category": "finance.account",
  "inputs": [],
  "outputs": ["balance"],
  "semantics": "Returns the current balance of a wallet or account",
  "cost": 0,
  "latency": 10,
  "reliability": 1.0
}
```

- **name** / **id**: Provider-specific name (e.g. `wallet.balance`).
- **canonical**: Ontology name (e.g. `finance.balance`). If omitted, defaults to `name` (backward compatibility).
- **category**: Optional taxonomy (e.g. `finance.account`).
- **inputs** / **outputs**: Optional list of input/output names.
- **semantics**: Human-readable description of what the capability does.
- **cost**, **latency**, **reliability**: Used for provider ranking.

### 2.2 Registry storage

- **providersByCanonical**: `{ "finance.balance": [provider1, provider2], ... }`  
  Each provider is the same descriptor object (id, dapp, canonical, cost, latency, reliability, etc.).
- **Indexing**: During `scan()`, capabilities from `capabilitySchemas` and from `manifest.capabilities` (when present) are normalized and indexed by **canonical**. If `canonical` is missing, `canonical = name` (migration).

### 2.3 API

- **getProviders(canonicalName)** → list of providers for that canonical capability.
- **getBestCapabilityByCanonical(canonicalName)** → single best provider (by ranking).
- **getBestCapability(intent)** / **getBestCapabilityByName(name)** remain for intent/id lookup.

---

## 3. Provider ranking

**Formula:** `score = reliability / (cost + latency)`

- **reliability**: 0–1 (or from `trust_score` if `reliability` absent).
- **cost**: Non-negative (e.g. FC).
- **latency**: Positive (e.g. ms). Avoid division by zero: use `reliability` only when `cost + latency <= 0`.

Higher score = preferred provider. The registry’s **scoreByOntology(cap)** implements this; **getBestCapabilityByCanonical** returns the provider with the highest score.

---

## 4. Identity-scoped permissions

Capabilities are **not** globally callable. They must be **granted per identity**.

### 4.1 Module: capabilityPermissions.js

- **grantCapability(identityId, canonicalCapability)**  
  Grants one canonical capability to an identity.
- **revokeCapability(identityId, canonicalCapability)**  
  Revokes it.
- **hasCapability(identityId, canonicalCapability)**  
  Returns whether that identity has the grant.

Storage: in-memory object `_grants[identityId][canonical] = true`; optionally persisted to `FRAME_STORAGE` under key `frame_capability_grants`.

### 4.2 Behavior

- **Default (no strict mode):** If an identity has no grant set at all, it is allowed all capabilities (backward compatible).
- **Strict mode (`window.FRAME_STRICT_CAPABILITY_GRANTS === true`):** Every call requires an explicit grant; otherwise `hasCapability` is false.

### 4.3 Execution check

Before running a resolved dApp, the engine:

1. Resolves the request to a **canonical** capability (from `match.canonical` or `intent.action`).
2. Gets current **identity** (e.g. from `get_current_identity`).
3. Calls **hasCapability(identity, canonical)**.
4. If false → returns error: `Capability not permitted for this identity: <canonical>`.
5. If true → continues with existing dApp permission and execution.

So: **identity → has grants → engine allows use of those canonical capabilities → agents/dApps use only what is granted.**

---

## 5. Planner integration

### 5.1 planExecution(intent, capabilities, context)

- **intent**: `{ action, payload }` (action may be raw or canonical).
- **capabilities**: List from registry (for future use).
- **context**: Optional `{ recentIntents, activeWidgets, deviceState, userIdentity }`.

The planner maps raw actions to **canonical** names (e.g. `wallet.balance` → `finance.balance`) and outputs a graph of **canonical** steps.

### 5.2 Execution graph shape

```json
{
  "executionGraph": [
    {
      "id": "balance",
      "canonical": "finance.balance",
      "args": {}
    },
    {
      "id": "tx",
      "canonical": "finance.transactions",
      "dependsOn": ["balance"],
      "args": { "limit": 10 }
    }
  ]
}
```

- **id**: Step id (for dependencies and debugging).
- **canonical**: Canonical capability name.
- **dependsOn**: Optional list of step ids that must run before this one (sequential execution respects order).
- **args**: Arguments for this step.

### 5.3 executeGraph(engine, plan)

- For each step, calls **engine.handleIntent(step.canonical, step.args)**.
- The engine (and router) resolve **canonical** → best provider → dApp, then check **hasCapability(identity, canonical)**, then execute with **providerIntent** (e.g. `wallet.balance`) passed to the dApp.

So the planner operates only on canonical names; resolution to a concrete provider and permission check happen at execution time.

---

## 6. Execution verification flow

1. User or agent triggers an intent (e.g. “send 5 FC to Alice”).
2. **Planner** produces a graph with canonical steps (e.g. `finance.balance`, then `finance.transactions`).
3. **executeGraph** runs each step:
   - **engine.handleIntent(canonical, args)**:
     - **Router** resolves `canonical` via **getBestCapabilityByCanonical** → match with `id`, `providerIntent`, `canonical`.
     - **Engine** gets current identity.
     - **Engine** checks **hasCapability(identity, canonical)** → if false, return error.
     - **Engine** runs existing dApp permission check (hasPermission(identity, dappId, cap)).
     - **Router** executes dApp with intent `action = providerIntent` (e.g. `wallet.balance`).
4. Result is returned to the caller; next step may use it (e.g. `$prev`).

So: **canonical → best provider → identity permission check → dApp permission check → execute with provider intent.**

---

## 7. Example manifests

### 7.1 capabilitySchemas with canonical (wallet)

```json
{
  "id": "wallet.balance",
  "canonical": "finance.balance",
  "category": "finance.account",
  "type": "data",
  "semantics": "Returns the current balance of a wallet or account",
  "inputs": [],
  "outputs": ["balance"],
  "cost": 0,
  "latency": 10,
  "reliability": 1.0
}
```

### 7.2 manifest.capabilities (object form)

```json
"capabilities": [
  {
    "name": "wallet.balance",
    "canonical": "finance.balance",
    "category": "finance.account",
    "inputs": [],
    "outputs": ["balance"],
    "semantics": "Returns the current balance of a wallet or account",
    "cost": 0,
    "latency": 50,
    "reliability": 1.0
  }
]
```

If only `name` is present, `canonical` defaults to `name` (migration).

---

## 8. Example execution graphs

### 8.1 Single step (balance)

Intent: `{ action: "wallet.balance", payload: {} }`  
Graph:

```json
{
  "executionGraph": [
    { "id": "balance", "canonical": "finance.balance", "args": {} }
  ]
}
```

Execution: resolve `finance.balance` → e.g. wallet dApp → check hasCapability(identity, "finance.balance") → execute wallet with `wallet.balance`.

### 8.2 Two steps (balance then send)

Intent: `{ action: "wallet.send", payload: { target: "Alice", amount: 5 } }`  
Graph:

```json
{
  "executionGraph": [
    { "id": "balance", "canonical": "finance.balance", "args": {} },
    {
      "id": "tx",
      "canonical": "finance.transactions",
      "dependsOn": ["balance"],
      "args": { "target": "Alice", "amount": 5 }
    }
  ]
}
```

Execution: run `finance.balance`, then run `finance.transactions` with target and amount (and optionally `$prev` from balance).

---

## 9. Debug logging (FRAME_DEV_MODE)

When `window.FRAME_DEV_MODE === true`:

- **[planner]**  
  - `planExecution` (intent, context keys)  
  - `canonical capability requested: <canonical>`  
  - `providers found: <n> [dapp1, dapp2, ...]`  
  - `selected provider: <dapp>`
- **[graph]**  
  - `executionGraph <length> [canonical1, canonical2, ...]`
- **[execution]**  
  - `executeGraph steps <n>`  
  - `identity verified; capability allowed: <canonical>`  
  - or `capability not permitted for identity: <canonical>`  
  - `step <i> <canonical> <args keys>`  
  - `complete <n>`

---

## 10. Summary

- **Ontology**: Canonical names (e.g. `finance.balance`) with optional category, inputs, outputs, semantics, cost, latency, reliability.
- **Registry**: `providersByCanonical`, getProviders, getBestCapabilityByCanonical, scoreByOntology; scan from capabilitySchemas and manifest.capabilities; default canonical = name.
- **Permissions**: capabilityPermissions.js with grant/revoke/hasCapability; engine checks hasCapability(identity, canonical) before execution.
- **Planner**: planExecution(intent, capabilities, context) returns graph with id, canonical, dependsOn, args; executeGraph runs handleIntent(canonical, args); router resolves canonical to best provider and passes providerIntent to the dApp.
- **Result**: Identity-scoped, ontology-based capability model that scales to many dApps and keeps execution under user-controlled grants.
