# FRAME Runtime Verification

This document describes the stabilized runtime pipeline, execution graph behavior, UI planning, capability resolution, automated flow tests, and fallback behavior for verification and debugging.

## 1. Runtime Pipeline Diagram

```
User intent (text or structured)
  → [conv] resolveIntent (ai.parse_intent or kernel normalize)
  → intent resolved { action, payload, raw, timestamp }
  → [planner] planExecution(intent, capabilities, context)
  → [graph] execution graph (steps with id, canonical, dependsOn, args)
  → [execution] executeGraph(engine, plan, options)
  → capability resolution (registry.getProviders, optional retries + fallback)
  → engine.handleIntent(canonical, args) per step
  → [router] resolve → execute (dApp in sandbox)
  → receipt build + append → state root update
  → [ui] UI plan (planUI or generateFallbackUIPlan)
  → uiPlanExecutor.executeUIPlan(plan)
  → widget rendering
```

All plans presented to the executor use the **standard ui.plan format** only: `{ type: "ui.plan", layout: string, widgets: [{ widget, props, position? }] }`.

## 2. Execution Graph Behavior

- **Topological sort**: Steps run in dependency order; cycles and missing deps throw before execution.
- **Level-based parallel execution**: If `options.parallel === true`, steps with the same dependency depth run concurrently via `Promise.all`; otherwise steps run sequentially.
- **Options**:
  - `parallel`: boolean — run same-level steps in parallel.
  - `retries`: number — retry a step up to this many times on failure.
  - `retryDelay`: number (ms) — delay between retries.
- **Provider fallback**: For each step, the registry’s `getProviders(canonical)` is used. If the best provider fails after retries, the next provider is tried; `[execution] fallback provider` is logged.
- **Result map**: Step results are stored by step `id`; later steps receive `args.$deps` and `args.$prev` from completed dependencies.

## 3. UI Planning Pipeline

- **Intent-driven**: `planUI(intent, context, capabilities, widgets)` (AI dApp) returns a standard `ui.plan` or null.
- **Fallback**: If the dynamic import of the UI planner fails, `generateFallbackUIPlan(intent)` is used. It maps actions to default widgets (e.g. `wallet.balance` → `wallet_activity`, `system.overview` → `system_status`, `notes.list` → `notes_recent`).
- **Executor**: `executeUIPlan(plan, intentForFallback)` accepts only the standard format; legacy-shaped plans (e.g. `region` + `widgets`) are normalized to `{ type: "ui.plan", layout, widgets }` before execution.

## 4. Capability Resolution

- **Registry**: Scans installed dApps and builds indexes; `getCapabilityGraph()` returns nodes and edges for composition.
- **Composition**: The planner can build multi-step graphs when the capability graph shows that output of capability A satisfies input of B (e.g. `notes.read` → `ai.summarize` → `messages.send`). When used, `[planner] composed workflow` is logged.
- **Best provider**: `getBestCapabilityByCanonical(canonical)`; execution can pass `__framePreferredProvider` to prefer a given provider; fallback to next provider on failure.

## 5. Flow Test Description

**File:** `ui/dev/runtimeFlowTests.js`

**Entry:** Run from command bar: type `frame test` or `frame.test`. Results appear in the activity feed.

**Function:** `runFrameFlowTests()`

**Flows covered:**

1. **show wallet** — intent `wallet.balance`, graph, execution, UI plan, receipt.
2. **send 5 FC to Alice** — intent `wallet.send` with target/amount; receipt optional (may need vault).
3. **create note "hello world"** — intent `notes.create` with text.
4. **list contacts** — intent `contacts.read`.
5. **system overview** — intent `system.overview`; empty graph allowed; UI plan and optional receipt.

**Per-flow checks:**

- Intent resolved (action set).
- Execution graph built (`planExecution` returns a graph; may be empty for overview).
- Execution completed (`executeGraph` run; errors logged as FAIL).
- UI plan generated (`generateFallbackUIPlan` or planUI returns standard plan).
- Receipt produced when `handleIntent` is run and returns a receipt (optional for some flows).

**Logging:** Each test logs `[flow-test] PASS <name>` or `[flow-test] FAIL <name> (<reason>)`; final summary: `[flow-test] done: N passed, M failed`.

## 6. Fallback Behavior

| Component        | Fallback behavior |
|-----------------|-------------------|
| UI planner load | If dynamic import of `ui/dapps/ai/uiPlanner.js` fails, use `FRAME_UI_PLANNER.generateFallbackUIPlan(prompt)` and execute that plan. |
| UI plan shape   | Non-standard plans are normalized to `{ type: "ui.plan", layout, widgets }`; if no valid widgets, fallback plan from intent is used. |
| Execution step  | If the best capability provider fails after retries, try next provider from `getProviders(canonical)`; log `[execution] fallback provider`. |
| Intent resolution | Unknown or empty intents map to `ai.chat` for a conversational reply. |

## 7. Debug Log Prefixes

Use these consistently for filtering and tracing:

| Prefix      | Subsystem           |
|------------|---------------------|
| `[conv]`   | conversationEngine  |
| `[planner]`| agentPlanner (planning + composition) |
| `[graph]`  | agentPlanner (graph output) |
| `[execution]` | agentPlanner / engine (step run, fallback) |
| `[router]` | router (resolve, execute) |
| `[ui]`     | uiPlanExecutor, fallback UI plan |
| `[flow-test]` | runtimeFlowTests.js |
| `[runtime]`| runtimeSelfCheck.js |

## 8. Runtime Self-Check

**File:** `ui/system/runtimeSelfCheck.js`

Runs after load. Verifies:

- `capabilityRegistry` scanned (getCapabilities)
- `uiPlanExecutor` (executeUIPlan)
- `agentPlanner` (planExecution)
- `widgetManager` (spawnWidget/createWidget)
- `layoutManager` (applyLayout)
- engine/router (handleIntent)

On failure: `[runtime] subsystem missing: <name>`.

Optional: `window.FRAME_RUNTIME_SELF_CHECK()` to re-run the check.
