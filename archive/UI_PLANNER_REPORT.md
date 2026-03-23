# FRAME Intent-Driven UI Planning Layer — Report

## 1. Planner Architecture

### Overview

An intent-driven UI planning stage sits between **intent resolution** and **execution**. When the user sends a message, the pipeline:

1. **Resolves intent** (e.g. `view.open`, `view.show`, `system.overview`) via `ai.parse_intent` or fallback.
2. **Calls the UI planner** with the resolved intent, context, capabilities, and available widget types.
3. **If the planner returns a `ui.plan`** — the executor applies layout, spawns/updates/destroys widgets, then renders.
4. **Otherwise** — the previous flow continues: `executeIntent` → dApp execution → display message.

### Components

| Component | Role |
|-----------|------|
| **ui/dapps/ai/uiPlanner.js** | Exports `planUI(intent, context, capabilities, widgets)`. Returns a standardized `{ type: "ui.plan", layout, widgets }` or `null`. Uses relevance scoring and intent patterns (e.g. `view.open` + target) to choose layout and widget list. |
| **ui/system/uiPlanExecutor.js** | Consumes both the **standardized** plan format (`type: "ui.plan"`) and the **legacy** format (loadingWidget, region, widgets). Applies layout via `FRAME_LAYOUT.applyLayout(layoutName)`, optionally destroys ids in `plan.destroyIds`, then spawns/updates widgets via `FRAME_WIDGETS.spawnWidget` / `updateWidget`. |
| **ui/system/conversationEngine.js** | After `resolveIntent`, dynamically imports the planner, gathers context/capabilities/widget types, calls `planUI()`. If result `type === 'ui.plan'`, calls `FRAME_UI_PLANNER.executeUIPlan(plan)` and updates the assistant message; otherwise continues with `executeIntent`. |
| **ui/system/widgetManager.js** | New API: `spawnWidget(type, props)`, `destroyWidget(id)`, `updateWidget(id, props)`. Widgets attach to layout regions; layout manager persists region/widget lists. |
| **ui/system/layoutManager.js** | New: `applyLayout(layoutName)` loads `layouts/<name>.json` and calls `setLayout(parsed)`. `refreshLayout()` triggers change listeners. |
| **ui/system/capabilityRegistry.js** | New: `getWidgetTypes()` returns widget type ids collected from dApp manifests’ `widgets` array. Manifests can advertise both `capabilities` and `widgets`. |

### Standardized UI Plan Shape

```json
{
  "type": "ui.plan",
  "layout": "dashboard",
  "widgets": [
    { "widget": "default_wallet", "props": {}, "position": "workspace" },
    { "widget": "system_status", "props": { "title": "Status" }, "position": "workspace" }
  ],
  "destroyIds": []
}
```

- **type**: Always `"ui.plan"` for the new path.
- **layout**: Named layout (e.g. `dashboard`, `minimal`). Loaded from `layouts/<layout>.json`.
- **widgets**: Array of `{ widget, props?, position? }`. `widget` is the widget type id; `props` are passed to `spawnWidget`; `position` maps to region (e.g. `workspace`, `sidebar`).
- **destroyIds**: Optional array of widget ids to remove before applying the plan.

---

## 2. Pipeline Changes

### Before

```
User input → resolveIntent → [if ai.chat] getChatReply
                          → [else] executeIntent(action, payload) → getWidgetsForIntent → showWidgets (by id)
```

### After

```
User input → resolveIntent
          → [if ai.chat] getChatReply
          → [else] import(uiPlanner) → planUI(intent, context, capabilities, widgets)
                  → [if result.type === 'ui.plan'] executeUIPlan(plan) → refreshLayout / renderAll
                  → [else] executeIntent(action, payload) → getWidgetsForIntent → showWidgets
```

So:

- **resolveIntent** is unchanged.
- **New step**: After resolving intent (and before executing the dApp), we call `planUI()`. If it returns a `ui.plan`, we run **uiPlanExecutor** and skip the direct `executeIntent` for that turn.
- **Legacy behavior** is preserved: if `planUI` returns `null` or a non–`ui.plan` value, the pipeline falls back to `executeIntent` and the existing widget-by-id logic.

---

## 3. Files Modified / Created

| File | Change |
|------|--------|
| **ui/dapps/ai/uiPlanner.js** | Added `planUI(intent, context, capabilities, widgets)`, export `UI_PLAN_TYPE`. Intent patterns for `view.open` / `view.show` / `system.overview`; relevance-based widget list; debug logs when `FRAME_DEV_MODE`. Kept `generateUIPlan` for legacy/overview. |
| **ui/system/uiPlanExecutor.js** | Added handling for `type === 'ui.plan'`: `executeStandardPlan(plan)` — apply layout, destroy `destroyIds`, spawn/update from `plan.widgets`. Exposed `UI_PLAN_TYPE`. Legacy plan shape still supported. |
| **ui/system/conversationEngine.js** | In `processUserMessage`, after resolve (and before execute): dynamic `import(uiPlanner)`, gather context/capabilities/widgets, call `planUI`. If `ui.plan`, call `executeUIPlan`, refresh layout/widgets, return. |
| **ui/system/widgetManager.js** | Added `spawnWidget(type, props)`, `destroyWidget(id)`, `updateWidget(id, props)` and exported them on `FRAME_WIDGETS`. |
| **ui/system/layoutManager.js** | Added `applyLayout(layoutName)` (fetch `layouts/<name>.json`, parse, `setLayout`), `refreshLayout()` (emitChange), and exported both. |
| **ui/system/capabilityRegistry.js** | Added `_widgetTypes`, populated in `scan()` from `manifest.widgets`. Added `getWidgetTypes()` and exported it. |
| **ui/dapps/wallet/manifest.json** | Example: added `"widgets": ["wallet.balance", "wallet.transactions"]`. |

---

## 4. Example UI Plans

### View wallet

Intent: `view.open` with payload `{ target: "wallet" }`.

```json
{
  "type": "ui.plan",
  "layout": "dashboard",
  "widgets": [
    { "widget": "default_wallet", "props": {}, "position": "workspace" }
  ]
}
```

### View system overview

Intent: `view.open` with payload `{ target: "system overview" }` (or similar). Planner uses relevance scoring (from `relevance.js`).

```json
{
  "type": "ui.plan",
  "layout": "dashboard",
  "widgets": [
    { "widget": "system_status", "props": {}, "position": "workspace" },
    { "widget": "network_activity", "props": {}, "position": "workspace" },
    { "widget": "recent_tasks", "props": {}, "position": "workspace" },
    { "widget": "wallet_activity", "props": {}, "position": "workspace" },
    { "widget": "notes_recent", "props": {}, "position": "workspace" }
  ]
}
```

### View notes

Intent: `view.open` with payload `{ target: "notes" }`.

```json
{
  "type": "ui.plan",
  "layout": "dashboard",
  "widgets": [
    { "widget": "default_notes", "props": {}, "position": "workspace" }
  ]
}
```

### dApp manifest advertising widgets

Example (wallet):

```json
{
  "name": "Wallet",
  "id": "wallet",
  "capabilities": ["wallet.read", "wallet.send", "storage.read", "storage.write"],
  "widgets": ["wallet.balance", "wallet.transactions"]
}
```

The capability registry scans `manifest.widgets` and exposes them via `getWidgetTypes()` for the planner and other UI logic.

---

## 5. Debug Logging (FRAME_DEV_MODE)

When `window.FRAME_DEV_MODE === true`:

- **uiPlanner.js**: `[planner] intent <action> <payload>`, `[planner] plan generated <layout> <count> <widgets>`.
- **uiPlanExecutor.js**: `[planner] layout applied <layoutName> <boolean>`, `[planner] widgets rendered <count>`.

These allow tracing intent → plan → layout application → widget render without changing production behavior.

---

## 6. Summary

- **Planner**: `planUI(intent, context, capabilities, widgets)` in `ui/dapps/ai/uiPlanner.js` returns a standardized `ui.plan` or `null`.
- **Pipeline**: resolveIntent → planUI → if `ui.plan` then uiPlanExecutor (apply layout, spawn/destroy/update widgets) and refresh; else executeIntent as before.
- **Executor**: Supports both the new plan format and the legacy overview format; uses `applyLayout`, `spawnWidget`, `destroyWidget`, `updateWidget`.
- **Widget/layout**: widgetManager exposes spawn/destroy/update; layoutManager exposes applyLayout and refreshLayout.
- **Registry**: Capability registry collects and exposes `getWidgetTypes()` from manifest `widgets` arrays.
- **Manifest**: dApps can declare `widgets` (e.g. wallet manifest) for discovery and planning.
