# FRAME Developer Guide

This guide explains how to build on FRAME, including creating dApps, defining capabilities, writing manifests, handling intents, generating widgets, and testing deterministic behavior.

## Creating a dApp

### Step 1: Create Directory Structure

Create a directory for your dApp:

```
ui/dapps/your-dapp/
├── manifest.json
└── index.js
```

### Step 2: Write Manifest

Create `manifest.json`:

```json
{
  "name": "Your dApp",
  "id": "your.dapp",
  "version": "1.0.0",
  "intents": ["your.intent"],
  "capabilities": ["storage.read", "storage.write"],
  "capabilitySchemas": [
    {
      "id": "your.intent",
      "type": "data",
      "description": "Your intent description",
      "input": {},
      "output": { "result": "string" }
    }
  ]
}
```

**Required Fields**:
- `name`: Human-readable name
- `intents`: Array of intent strings
- `capabilities`: Array of capability strings

**Optional Fields**:
- `id`: dApp identifier (set by router if missing)
- `version`: Version string
- `capabilitySchemas`: Detailed capability descriptions
- `widgets`: Array of widget IDs

### Step 3: Implement dApp

Create `index.js`:

```javascript
export async function run(intent, ctx) {
  // ctx.storage.read/write available
  // ctx.capabilities available
  // ctx.identity available
  
  var action = intent.action;
  var payload = intent.payload || {};
  
  if (action === 'your.intent') {
    // Read from storage
    var data = await ctx.storage.read('your:key');
    
    // Write to storage
    await ctx.storage.write('your:key', { value: 'data' });
    
    // Return result
    return {
      type: 'dapp',
      result: 'Success',
      state: { 'your:key': { value: 'data' } }  // Optional state update
    };
  }
  
  return { type: 'error', message: 'Unknown intent' };
}
```

## Defining Capabilities

### Capability Declaration

Capabilities must be declared in the manifest:

```json
{
  "capabilities": ["storage.read", "storage.write", "wallet.read"]
}
```

### Capability Usage

Capabilities are accessed through the scoped API:

```javascript
// Storage capability
await ctx.storage.read('key');
await ctx.storage.write('key', value);

// Wallet capability
var balance = await ctx.wallet.getBalance();

// Identity capability
var identity = await ctx.identity.getCurrent();
```

### Capability Guard

All capability operations go through a guard:

```javascript
// Guard checks:
// 1. Capability declared in manifest
// 2. Capability granted by user
// 3. Capability logged for receipt
```

## Writing Manifests

### Manifest Structure

```json
{
  "name": "dApp Name",
  "id": "dapp.id",
  "version": "1.0.0",
  "intents": ["intent.action"],
  "capabilities": ["capability.name"],
  "capabilitySchemas": [
    {
      "id": "intent.action",
      "type": "data" | "payment" | "bridge",
      "description": "Intent description",
      "input": { "param": "type" },
      "output": { "result": "type" },
      "examples": ["Example usage"]
    }
  ],
  "widgets": ["widget.id"]
}
```

### Manifest Validation

Manifests are validated by the router:

**Rules**:
- `name` must be a string
- `intents` must be an array of strings
- `capabilities` must be an array
- Each capability must exist in `CAPABILITY_SCHEMA`

## Handling Intents

### Intent Structure

Intents have this structure:

```javascript
{
  action: "your.intent",
  payload: { param: "value" },
  timestamp: 1234567890,
  raw: "original user input"
}
```

### Intent Handling

Handle intents in the `run()` function:

```javascript
export async function run(intent, ctx) {
  var action = intent.action;
  var payload = intent.payload || {};
  
  switch (action) {
    case 'your.intent':
      return handleYourIntent(payload, ctx);
    default:
      return { type: 'error', message: 'Unknown intent' };
  }
}
```

### Return Values

Return values must be canonical JSON:

```javascript
// Success
return {
  type: 'dapp',
  result: 'Success',
  state: { 'key': 'value' }  // Optional state update
};

// Error
return {
  type: 'error',
  message: 'Error message'
};
```

## Generating Widgets

### Widget Schema

The UI planner generates widget schemas automatically. To influence widget generation, use `capabilitySchemas`:

```json
{
  "capabilitySchemas": [
    {
      "id": "your.intent",
      "type": "data",
      "output": { "timeseries": "array" }  // Suggests chart widget
    }
  ]
}
```

### Widget Types

Supported widget types:
- `card`: Simple data display
- `table`: Tabular data
- `chart`: Data visualization
- `grid`: Grid layout
- `tabs`: Tabbed interface
- `map`: Geographic map
- `timeline`: Timeline visualization
- `kanban`: Kanban board
- `editor`: Text/code editor
- `panel`: Panel container
- `list`: List display
- `form`: Form input

### Widget Parameters

Widget parameters can be specified in `capabilitySchemas`:

```json
{
  "capabilitySchemas": [
    {
      "id": "your.intent",
      "type": "data",
      "output": { "timeseries": "array" },
      "widgetParams": { "chartType": "line" }
    }
  ]
}
```

## Testing Deterministic Behavior

### Test Structure

Create tests in `tests/headless/`:

```javascript
import { test, expect } from '@playwright/test';

test('dApp produces deterministic results', async ({ page }) => {
  // Setup
  await page.goto('http://localhost:3000/index.html');
  await waitForFRAME(page);
  
  // Execute intent
  var result1 = await page.evaluate(async () => {
    return await window.FRAME.handleStructuredIntent({
      dappId: 'your.dapp',
      action: 'your.intent',
      parameters: {}
    });
  });
  
  // Execute again
  var result2 = await page.evaluate(async () => {
    return await window.FRAME.handleStructuredIntent({
      dappId: 'your.dapp',
      action: 'your.intent',
      parameters: {}
    });
  });
  
  // Verify deterministic
  expect(result1.resultHash).toBe(result2.resultHash);
});
```

### Replay Testing

Test replay behavior:

```javascript
test('replay produces identical results', async ({ page }) => {
  // Execute intent
  var result = await page.evaluate(async () => {
    return await window.FRAME.handleStructuredIntent({
      dappId: 'your.dapp',
      action: 'your.intent',
      parameters: {}
    });
  });
  
  // Get receipt hash
  var receiptHash = result.receipt.receiptHash;
  
  // Replay
  var replayResult = await page.evaluate(async (hash) => {
    return await window.FRAME.replayIntent(hash);
  }, receiptHash);
  
  // Verify replay
  expect(replayResult.replayValid).toBe(true);
});
```

### Determinism Verification

Verify determinism:

```javascript
test('verify determinism', async ({ page }) => {
  var result = await page.evaluate(async () => {
    var identity = await window.__FRAME_INVOKE__('get_current_identity');
    var chainKey = 'chain:' + identity;
    var chain = await window.FRAME_STORAGE.read(chainKey);
    return await window.FRAME.verifyDeterminism(chain);
  });
  
  expect(result.verified).toBe(true);
});
```

## Best Practices

### Deterministic Code

- Use sorted key iteration: `Object.keys(obj).sort()`
- Use safe integers only (no floats)
- Use intent timestamp (not `Date.now()`)
- Avoid `Math.random()` (use seeded PRNG if needed)

### Capability Design

- Declare minimal required capabilities
- Use capability schemas for widget generation
- Document capability usage

### Error Handling

- Return structured errors: `{ type: 'error', message: '...' }`
- Validate inputs before processing
- Handle capability denials gracefully

### State Management

- Use `state` field in return value for updates
- Keep state updates atomic
- Validate state values (canonical JSON only)

## Examples

### Simple dApp

```javascript
// manifest.json
{
  "name": "Counter",
  "intents": ["counter.increment"],
  "capabilities": ["storage.read", "storage.write"]
}

// index.js
export async function run(intent, ctx) {
  if (intent.action === 'counter.increment') {
    var count = await ctx.storage.read('counter:count') || 0;
    count = (count || 0) + 1;
    await ctx.storage.write('counter:count', count);
    return {
      type: 'dapp',
      result: { count: count },
      state: { 'counter:count': count }
    };
  }
  return { type: 'error', message: 'Unknown intent' };
}
```

### Widget-Generating dApp

```javascript
// manifest.json
{
  "name": "Data Viewer",
  "intents": ["data.view"],
  "capabilities": ["storage.read"],
  "capabilitySchemas": [
    {
      "id": "data.view",
      "type": "data",
      "description": "View data",
      "output": { "items": "array" },
      "widgetParams": { "widgetType": "list" }
    }
  ]
}

// index.js
export async function run(intent, ctx) {
  if (intent.action === 'data.view') {
    var items = await ctx.storage.read('data:items') || [];
    return {
      type: 'dapp',
      result: { items: items }
    };
  }
  return { type: 'error', message: 'Unknown intent' };
}
```

## Debugging

### Runtime Debugging

Enable debug mode:

```javascript
window.FRAME_DEV_MODE = true;
```

### Receipt Inspection

Inspect receipts:

```javascript
var identity = await window.__FRAME_INVOKE__('get_current_identity');
var chainKey = 'chain:' + identity;
var chain = await window.FRAME_STORAGE.read(chainKey);
console.log(chain[chain.length - 1]); // Latest receipt
```

### State Root Inspection

Inspect state root:

```javascript
var identity = await window.__FRAME_INVOKE__('get_current_identity');
var stateRoot = await window.FRAME_STATE_ROOT.computeStateRoot(identity);
console.log(stateRoot);
```

## Resources

- [Architecture](architecture.md) - System architecture
- [Runtime](runtime.md) - Runtime internals
- [Storage](storage.md) - Storage design
- [Security](security.md) - Security model
- [UI System](ui-system.md) - UI architecture
- [Determinism and Replay](determinism-and-replay.md) - Deterministic guarantees
