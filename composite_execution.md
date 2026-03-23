# FRAME Composite Execution

FRAME supports composite execution: executing multiple intents atomically as a single transaction with rollback on failure.

**File:** `ui/runtime/engine.js` — `executeCompositeIntent(steps)`

## Composite Execution Overview

Composite execution enables:

- **Atomic transactions** — All steps succeed or all fail
- **Rollback on failure** — State restored if any step fails
- **Dependency ordering** — Steps execute in dependency order
- **Multi-step operations** — Complex workflows as single transaction

**Example:** `wallet.send` might require:
1. Check balance
2. Validate recipient
3. Send payment
4. Update ledger

All steps execute atomically.

## Execution Graph

**File:** `ui/system/agentPlanner.js` — `planExecution(intent, capabilities, context)`

The agent planner builds execution graphs:

```javascript
{
  executionGraph: [
    { id: 'balance', canonical: 'finance.balance', args: {} },
    { id: 'tx', canonical: 'finance.transactions', dependsOn: ['balance'], args: { target, amount } }
  ]
}
```

### Graph Structure

**Step fields:**

- `id` — Unique step identifier
- `canonical` — Canonical capability name
- `dependsOn` — Array of step IDs this step depends on
- `args` — Arguments for this step
- `parallel` — Optional: run in parallel with same-level steps

### Dependency Resolution

**File:** `ui/system/agentPlanner.js` — `topoSort(graph)`

**Process:**

1. Build dependency graph
2. Detect cycles
3. Topological sort
4. Return execution order

**Example:**

```javascript
[
  { id: 'a', dependsOn: [] },
  { id: 'b', dependsOn: ['a'] },
  { id: 'c', dependsOn: ['a'] },
  { id: 'd', dependsOn: ['b', 'c'] }
]
// Execution order: ['a', 'b', 'c', 'd']
```

## Composite Execution Process

**File:** `ui/runtime/engine.js` — `executeCompositeIntent(steps)`

### Step 1: Validation

```javascript
if (!Array.isArray(steps) || steps.length < 2) {
  return { type: 'error', message: 'Composite execution requires at least 2 steps.' };
}
if (_mode === 'safe') {
  return { type: 'error', message: 'Composite execution is not allowed in safe mode.' };
}
if (_mode === 'replay') {
  return { type: 'error', message: 'Composite execution is not allowed in replay mode.' };
}
```

### Step 2: Snapshot State

```javascript
var snapshotStorage = await gatherStorageSnapshot();
var snapshotStateRoot = await computeStateRoot(identity);
var snapshotChain = await FRAME_STORAGE.read('chain:' + identity);
```

**Snapshots capture:**
- Storage state
- State root
- Receipt chain

### Step 3: Execute Steps

```javascript
var results = [];
var failed = false;
var failureResult = null;

for (var i = 0; i < steps.length; i++) {
  var step = steps[i];
  
  // Build step intent
  var stepIntent = {
    action: step.canonical,
    payload: step.args,
    timestamp: normalizeTimestamp(Date.now())
  };
  
  // Inject dependency results
  if (step.dependsOn && step.dependsOn.length > 0) {
    var deps = {};
    for (var d = 0; d < step.dependsOn.length; d++) {
      deps[step.dependsOn[d]] = results[step.dependsOn[d]];
    }
    stepIntent.payload.$deps = deps;
    if (step.dependsOn.length === 1) {
      stepIntent.payload.$prev = results[step.dependsOn[0]];
    }
  }
  
  // Execute step
  var result = await handleStructuredIntent(stepIntent);
  
  if (result.type === 'error') {
    failed = true;
    failureResult = result;
    break;
  }
  
  results[step.id] = result;
}
```

### Step 4: Rollback on Failure

```javascript
if (failed) {
  // Restore storage
  await restoreStorageSnapshot(snapshotStorage);
  
  // Restore receipt chain
  await FRAME_STORAGE.write('chain:' + identity, snapshotChain);
  
  return failureResult;
}
```

### Step 5: Generate Receipt

If all steps succeed:

```javascript
var receipt = await buildReceipt({
  intent: { action: 'composite', payload: { steps } },
  identity: identity,
  dappId: 'system',
  capabilitiesDeclared: allCapabilities,
  capabilitiesUsed: allCapabilitiesUsed,
  previousStateRoot: snapshotStateRoot,
  nextStateRoot: await computeStateRoot(identity),
  previousReceiptHash: lastReceiptHash,
  result: { success: true, results: results }
});

await appendReceipt(identity, receipt);
```

## Dependency Injection

Steps can access results from dependencies:

**`$deps`** — Map of all dependency results:

```javascript
{
  balance: { balance: 100 },
  validation: { valid: true }
}
```

**`$prev`** — Single dependency result (convenience):

```javascript
{ balance: 100 }
```

**Example:**

```javascript
{
  id: 'tx',
  canonical: 'finance.transactions',
  dependsOn: ['balance', 'validation'],
  args: {
    target: 'Alice',
    amount: 10,
    $deps: {
      balance: { balance: 100 },
      validation: { valid: true }
    },
    $prev: { balance: 100 }  // First dependency
  }
}
```

## Parallel Execution

**File:** `ui/system/agentPlanner.js` — `executeGraph(engine, plan, options)`

**Options:**

```javascript
{
  parallel: true  // Run same-level steps in parallel
}
```

**Process:**

1. Compute dependency depth for each step
2. Group steps by depth level
3. Execute same-level steps in parallel
4. Wait for all steps to complete before next level

**Example:**

```javascript
Level 0: ['a']
Level 1: ['b', 'c']  // Parallel
Level 2: ['d']
```

## Error Handling

**On step failure:**

1. **Stop execution** — No further steps execute
2. **Rollback state** — Restore storage snapshot
3. **Restore chain** — Restore receipt chain
4. **Return error** — Return failure result

**Error result:**

```javascript
{
  type: 'error',
  message: 'Step failed',
  step: 'tx',
  error: { ... }
}
```

## Retry Logic

**File:** `ui/system/agentPlanner.js` — `executeGraph(engine, plan, options)`

**Options:**

```javascript
{
  retries: 3,        // Retry up to 3 times
  retryDelay: 1000   // 1 second delay between retries
}
```

**Process:**

1. Execute step
2. If failure and retries remaining:
   - Wait `retryDelay`
   - Retry step
3. If all retries exhausted:
   - Try next provider (if available)
   - Or fail

## Provider Fallback

If a step fails after retries:

1. **Get providers** — `getProviders(canonical)`
2. **Try next provider** — Execute with different dApp
3. **Continue** — If successful, continue execution

**Example:**

```javascript
// Step fails with wallet dApp
// Try payments dApp
// If successful, continue
```

## Execution Graph Examples

### Simple Composite

```javascript
[
  { id: 'balance', canonical: 'finance.balance', args: {} },
  { id: 'send', canonical: 'finance.transactions', dependsOn: ['balance'], args: { target: 'Alice', amount: 10 } }
]
```

### Complex Composite

```javascript
[
  { id: 'validate', canonical: 'contacts.validate', args: { recipient: 'Alice' } },
  { id: 'balance', canonical: 'finance.balance', args: {} },
  { id: 'send', canonical: 'finance.transactions', dependsOn: ['validate', 'balance'], args: { target: 'Alice', amount: 10 } },
  { id: 'notify', canonical: 'messages.send', dependsOn: ['send'], args: { recipient: 'Alice', message: 'Payment sent' } }
]
```

## Integration with Agent Planner

**File:** `ui/system/conversationEngine.js`

**Process:**

1. **Resolve intent** — `resolveIntent(userText)`
2. **Plan execution** — `agentPlanner.planExecution(intent, capabilities)`
3. **Execute graph** — `agentPlanner.executeGraph(FRAME, plan)`
4. **Return result** — Use result for response

**Example:**

```javascript
var intent = await resolveIntent("Send 10 frame to Alice");
var plan = await agentPlanner.planExecution(intent, capabilities);
if (plan.executionGraph.length > 1) {
  var result = await agentPlanner.executeGraph(FRAME, plan);
  return result;
} else {
  var result = await executeIntent(intent.action, intent.payload);
  return result;
}
```

## Summary

FRAME composite execution enables:

- **Atomic transactions** — All steps succeed or all fail
- **Rollback on failure** — State restored if any step fails
- **Dependency ordering** — Steps execute in dependency order
- **Multi-step operations** — Complex workflows as single transaction

**Composite execution guarantees:**

- Atomicity (all or nothing)
- Consistency (state always valid)
- Isolation (no partial state)
- Durability (receipt generated on success)

## Related Documentation

- [Runtime Pipeline](runtime_pipeline.md) - Execution flow
- [Agent System](agent_system.md) - Agent planner
- [Kernel Runtime](kernel_runtime.md) - Kernel coordination
- [Receipt Chain](receipt_chain.md) - Receipt generation
