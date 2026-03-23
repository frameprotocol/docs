# FRAME Capability Execution Planner

## Overview

The **agent planner** composes multiple dApp capabilities into an **execution graph**. The flow is:

1. **resolveIntent** → intent `{ action, payload }`
2. **planner.planExecution(intent, capabilities)** → `{ executionGraph: [{ capability, args }] }`
3. **executeGraph(engine, plan)** → runs each step via `engine.handleIntent(capability, args)` (sequentially)

The planner selects the best capability provider per step (the engine’s router uses the registry for `ai.*` and matches intents for other capabilities).

## 1. agentPlanner.js

- **planExecution(intent, capabilities)**  
  - Input: `intent` = `{ action: string, payload?: object }`, `capabilities` = list from registry.  
  - Output: `{ executionGraph: Array<{ capability: string, args: object, parallel?: boolean }> }`.  
  - Single-step: `[{ capability: intent.action, args: intent.payload || {} }]`.  
  - Composite example: `wallet.send` with target/amount → `[{ capability: 'wallet.balance', args: {} }, { capability: 'wallet.send', args: { target, amount } }]`.

- **executeGraph(engine, plan, options)**  
  - `engine` must expose `handleIntent(action, payload)`.  
  - Runs each step in order; can pass previous result into next step as `$prev`.  
  - Returns the last step result or an error object `{ type: 'error', message, step? }`.

- **getBestCapabilityForStep(capabilityName)**  
  - Uses `FRAME_CAPABILITY_REGISTRY.getCapability(name)` or `getBestCapability(name)`.

- **Debug (when `FRAME_DEV_MODE`):**  
  - `[planner] planExecution`, `[graph] executionGraph`, `[execution] executeGraph steps`, `[execution] step N`, `[execution] complete`.

## 2. Integration

- **conversationEngine:** After UI plan handling, calls `FRAME_AGENT_PLANNER.planExecution(intent, caps)`. If `plan.executionGraph.length > 0`, runs `executeGraph(FRAME, plan)` and uses the result for the assistant message; otherwise falls back to `executeIntent(action, payload)`.
- **Router:** Unchanged for resolution; the engine’s `handleIntent` still uses the router to resolve each capability to a dApp and execute.

## 3. Capability registry

- **getBestCapability(intent, options)** — already existed; returns best provider for an intent.
- **getBestCapabilityByName(name)** — returns `getCapability(name)` if present, else `getBestCapability(name)` for intent lookup.

## 4. Manifest capability metadata

Manifests can declare capabilities in two ways:

**String array (unchanged):**

```json
"capabilities": ["wallet.read", "wallet.send", "storage.read", "storage.write"]
```

**Object array (name + optional cost/latency):**

```json
"capabilities": [
  { "name": "wallet.balance", "cost": 1, "latency": 50 },
  { "name": "wallet.send", "cost": 2, "latency": 100 }
]
```

The router normalizes both to a list of capability names for validation and resolution. Detailed cost/latency for scoring remain in **capabilitySchemas** (e.g. `cost`, `latency` on each schema).

## 5. Example execution graph

Intent: `{ action: 'wallet.send', payload: { target: 'Alice', amount: 5 } }`

Plan:

```json
{
  "executionGraph": [
    { "capability": "wallet.balance", "args": {} },
    { "capability": "wallet.send", "args": { "target": "Alice", "amount": 5 } }
  ]
}
```

executeGraph runs `handleIntent('wallet.balance', {})` then `handleIntent('wallet.send', { target: 'Alice', amount: 5 })`.

## 6. Script load order

`agentPlanner.js` is loaded after `capabilityRegistry.js` and before `conversationEngine.js` so `FRAME_AGENT_PLANNER` is defined when the conversation engine runs.
