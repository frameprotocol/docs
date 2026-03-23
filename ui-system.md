# FRAME UI System

This document describes FRAME's UI architecture, including the UI planner dApp, widget schemas, capability introspection, and deterministic layout engine.

## UI Architecture Overview

FRAME generates UI dynamically from intents through a deterministic pipeline:

```
Intent
  ↓
Application dApp Execution
  ↓
UI Planner dApp (generates widget schema)
  ↓
Widget Schema Canonicalization
  ↓
Deterministic Layout Engine
  ↓
Widget Rendering
```

## UI Planner dApp

The UI planner (`ui.planner`) is a deterministic dApp that generates widget schemas from intents and execution results.

**Location**: `ui/dapps/ui.planner/index.js`

### Planner Responsibilities

1. **Widget Schema Generation**: Creates widget schemas from intents
2. **Capability Introspection**: Discovers capabilities from dApp manifests
3. **Deterministic Caching**: Caches schemas for instant replay
4. **Canonical JSON Output**: Returns only canonical JSON data

### Planner Execution

**Triggered**: After application dApp execution, before receipt building

**Input**:
```javascript
{
  intent: { action, payload, timestamp },
  executionResult: { /* dApp execution result */ },
  dappManifest: { /* dApp manifest */ }
}
```

**Output**:
```javascript
{
  type: "ui.plan",
  widgets: [
    {
      type: "card",
      title: "Wallet Balance",
      source: "wallet.balance",
      region: "workspace",
      params: {}
    }
  ]
}
```

### Capability Introspection

The planner introspects dApp manifests to discover available data sources:

**Process**:
1. Reads `capabilitySchemas` from manifest
2. Sorts capabilities deterministically
3. Infers widget types from capability metadata:
   - `type: "payment"` → `table` widget
   - `type: "data"` with timeseries → `chart` widget
   - `type: "data"` with list output → `list` widget
   - Default → `card` widget
4. Extracts widget parameters from capability output

**Example Manifest**:
```json
{
  "capabilitySchemas": [
    {
      "id": "wallet.balance",
      "type": "data",
      "description": "Check wallet balance",
      "output": { "balance": "number" }
    },
    {
      "id": "wallet.send",
      "type": "payment",
      "description": "Send frame tokens",
      "output": { "message": "string" }
    }
  ]
}
```

**Planner Output**:
```javascript
{
  widgets: [
    {
      type: "card",
      title: "Check wallet balance",
      source: "wallet.balance",
      region: "workspace",
      params: {}
    },
    {
      type: "table",
      title: "Send frame tokens",
      source: "wallet.send",
      region: "workspace",
      params: {}
    }
  ]
}
```

### Planner Caching

The planner caches widget schemas for instant replay:

**Cache Key**: `ui_plan_cache:<intent_hash>:<manifest_hash>`

**Cache Entry**:
```javascript
{
  widgetSchema: { /* widget schema */ },
  intentAction: "wallet.balance",
  cachedAt: intent.timestamp  // Deterministic timestamp
}
```

**Cache Behavior**:
- Cache lookup before generation
- Cache write after generation
- Uses intent timestamp (not `Date.now()`)
- Deterministic across replay

## Widget Schema Structure

Widget schemas are canonical JSON structures:

### Schema Format

```javascript
{
  type: "ui.plan",
  widgets: [
    {
      type: "card" | "table" | "chart" | "grid" | "tabs" | "map" | "timeline" | "kanban" | "editor" | "panel" | "list" | "form",
      title: "string",
      source: "string",  // Data source ID (e.g., "wallet.balance")
      region: "workspace" | "sidebar" | "dock" | "statusbar",
      params: {
        // Widget-specific parameters
        chartType: "line" | "bar" | "pie",  // For chart widgets
        // ... other params
      }
    }
  ]
}
```

### Widget Primitives

FRAME supports 12 widget primitive types:

- **card**: Simple data display card
- **table**: Tabular data display
- **chart**: Data visualization (line, bar, pie, etc.)
- **grid**: Grid layout for multiple items
- **tabs**: Tabbed interface
- **map**: Geographic map display
- **timeline**: Timeline visualization
- **kanban**: Kanban board
- **editor**: Text/code editor
- **panel**: Panel container
- **list**: List display
- **form**: Form input interface

### Widget Schema Validation

Widget schemas are validated for determinism:

**Checks**:
- No functions or executable code
- No floating point values (only safe integers)
- No undefined values
- All keys sorted (canonicalized)
- Recursive validation of nested objects

**Location**: `ui/runtime/engine.js` → `verifyWidgetSchemaDeterminism()`

## Deterministic Layout Engine

The layout engine computes widget placement deterministically:

**Location**: `ui/runtime/widgets/layout.js`

### Layout Rules

Layout is computed solely from widget count:

- **1 widget** → `single` layout (1 column, workspace)
- **2-3 widgets** → `vertical` layout (1 column stack, workspace)
- **4 widgets** → `grid` layout (2x2 grid, workspace)
- **5+ widgets** → `dashboard` layout (3 column grid, workspace)

### Layout Computation

**Process**:
1. Count widgets in schema
2. Determine layout type from count
3. Assign widgets to regions (sorted by source)
4. Apply layout to FRAME layout manager

**Determinism Guarantees**:
- Same widget count → same layout type
- Same widget order → same region assignments
- Widgets sorted by source before assignment
- No system state dependencies

### Region Assignment

Widgets are assigned to regions deterministically:

**Regions**:
- `workspace`: Main content area
- `sidebar`: Sidebar area
- `dock`: Dock area
- `statusbar`: Status bar area

**Assignment**:
1. Use `widget.region` from schema if specified
2. Otherwise, use computed layout
3. Sort widgets by source before assignment
4. Assign to regions in sorted order

## Widget Rendering

Widgets are rendered by the widget manager:

**Location**: `ui/system/widgetManager.js`

### Widget Creation

**Process**:
1. Read widget schema from execution result
2. Create widgets from schema
3. Connect data sources
4. Apply layout
5. Render to UI

**Widget Creation**:
```javascript
FRAME_WIDGETS.createWidget({
  id: widgetDef.id || genId(),
  title: widgetDef.title,
  type: widgetDef.type,
  dataSource: widgetDef.source,
  region: widgetDef.region,
  config: widgetDef.params || {}
});
```

### Data Source Connection

Widgets connect to data sources:

**Data Sources**:
- dApp capabilities (e.g., `wallet.balance`)
- Storage keys (e.g., `storage:notes:recent`)
- System state (e.g., `system.status`)

**Connection**:
```javascript
widget.dataSource = function() {
  // Fetch data from source
  return data;
};
```

## UI Execution Flow

The complete UI generation flow:

### 1. Intent Execution

```javascript
// User intent
{ action: "wallet.balance", payload: {} }
  ↓
// Application dApp executes
{ balance: 100 }
  ↓
// UI planner called
{ widgets: [{ type: "card", source: "wallet.balance" }] }
  ↓
// Widget schema canonicalized
{ widgets: [...] }  // Canonicalized
  ↓
// Added to execution result
execResult.widgetSchema = widgetSchema
  ↓
// Included in receipt hash
resultHash = hash(execResult)  // Includes widget schema
```

### 2. Layout Computation

```javascript
// Widget schema
{ widgets: [{ type: "card", source: "wallet.balance" }] }
  ↓
// Layout engine computes layout
{ type: "single", regions: { workspace: { widgets: [...] } } }
  ↓
// Layout applied
FRAME_LAYOUT.setLayout(layoutConfig)
```

### 3. Widget Rendering

```javascript
// Widgets created
FRAME_WIDGETS.createWidget({ type: "card", source: "wallet.balance" })
  ↓
// Data source connected
widget.dataSource = () => fetchBalance()
  ↓
// Widgets rendered
FRAME_WIDGETS.renderAll()
```

## Replay Guarantees

FRAME guarantees identical UI during replay:

**Property**:
```
same receipt chain
  → same execution result (via resultHash verification)
  → same widget schema (included in execResult)
  → same layout (deterministic from widget count)
  → same UI
```

**Verification**:
1. Widget schema included in `execResult.widgetSchema`
2. `resultHash = hash(execResult)` includes widget schema
3. Replay verifies `resultHash` matches
4. Planner regenerates identical schema (deterministic)
5. Layout engine computes identical layout (deterministic)

## UI Planner Determinism

The planner is fully deterministic:

### Deterministic Operations

- Sorted key iteration (`Object.keys().sort()`)
- Canonical JSON hashing
- Intent timestamp (not `Date.now()`)
- Safe integer arithmetic only

### Blocked Operations

- `Date.now()` (uses intent timestamp)
- `Math.random()` (not used)
- Floating point arithmetic (only safe integers)
- Unordered object iteration (must sort keys)

### Verification

Planner determinism verified through:
- Schema canonicalization
- Cache key determinism
- Replay consistency
- `FRAME.verifyDeterminism()` utility

## Widget Schema Integration

Widget schemas are integrated into receipts:

**Process**:
1. Planner generates widget schema
2. Schema canonicalized
3. Schema added to `execResult.widgetSchema`
4. `resultHash = hash(execResult)` includes schema
5. Receipt includes `resultHash`
6. Replay verifies `resultHash` matches

**Receipt Field**:
```javascript
{
  resultHash: "sha256(canonicalize(execResult))"  // Includes widgetSchema
}
```

This ensures that widget schemas are cryptographically committed in receipts, enabling verifiable UI reconstruction during replay.
