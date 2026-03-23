# FRAME Storage

This document explains FRAME's storage architecture, including receipt logs, state storage, transaction journals, and crash safety mechanisms.

## Storage Overview

FRAME uses a multi-layer storage architecture:

1. **Receipt Logs**: Append-only receipt storage (Rust backend)
2. **State Storage**: Canonical JSON key-value storage (Tauri backend)
3. **Transaction Journal**: In-flight transaction tracking (per-identity)
4. **Event Logs**: System event logging with rotation

All storage is **local-first** and **encrypted** at rest.

## Receipt Log Format

Receipts are stored in an append-only log file with frame-based format:

**File**: `storage/chain_<identity>.log`

**Frame Structure**:
```
[u32 length (big-endian)]
[u32 checksum (big-endian)]
[JSON receipt payload (UTF-8)]
```

**Example**:
```
[00 00 01 2C]  // Length: 300 bytes
[12 34 56 78]  // Checksum
{ "version": 2, "timestamp": 1234567890, ... }  // JSON receipt
```

### Receipt Log Operations

**Append Receipt** (`receipt_append_internal`):
1. Serialize receipt to canonical JSON
2. Compute checksum (hash of JSON bytes)
3. Build frame: length + checksum + JSON
4. Write to temporary file
5. `fsync()` to ensure durability
6. Atomic rename to log file

**Read Receipts** (`receipt_read_all`):
1. Read log file
2. Parse frames sequentially
3. Validate checksums
4. Return array of receipts

**Crash Safety**:
- Writes to temporary file first
- `fsync()` ensures data is on disk
- Atomic rename prevents partial writes
- Checksums detect corruption

## State Storage

State storage provides canonical JSON key-value storage:

**Backend**: Tauri `storage_read` / `storage_write` commands

**Location**: `storage/<key>.enc` (encrypted files)

**Validation**: All writes validated for canonical JSON

### Storage Validation

**File**: `ui/runtime/storage.js`

**Rules**:
- Only canonical JSON allowed:
  - Strings, booleans, safe integers, null
  - Arrays, plain objects
- Rejected types:
  - Floating point numbers (only safe integers)
  - NaN, Infinity
  - undefined
  - Functions
  - Date, RegExp, Map, Set, etc.
  - Circular references

**Validation Function**:
```javascript
function validateCanonicalJson(value, path, seen) {
  // Recursively validate value
  // Throw error if non-canonical
}
```

### Storage Operations

**Read**:
```javascript
await window.FRAME_STORAGE.read(key)
// Returns: value (deep-frozen) or null
```

**Write**:
```javascript
await window.FRAME_STORAGE.write(key, value)
// Validates canonical JSON
// Writes to encrypted storage
// Deep-freezes value
```

**Storage Keys**:
- `chain:<identity>`: Receipt chain (legacy format, migrated to log)
- `permissions:<identity>`: Capability grants
- `storage:<key>`: Application state
- `frame_*`: System state (excluded from state root)

## Transaction Journal

The transaction journal tracks in-flight transactions for atomic commits:

**File**: `storage/journal:<identity>.log`

**Purpose**: Ensure atomic state updates with receipt appends

**Format**: Lightweight log of pending writes

**Process**:
1. Before state write: Record in journal
2. Write state: Update storage
3. Append receipt: Write to receipt log
4. Commit: Remove journal entry
5. On crash: Replay journal entries

**Journal Entry**:
```javascript
{
  key: "storage:wallet:balance",
  value: 100,
  timestamp: 1234567890
}
```

## Event Log Rotation

Event logs are rotated to prevent unbounded growth:

**File**: `storage/event.log`

**Rotation**:
- When log exceeds size limit (e.g., 10MB)
- Rotate: `event.log` → `event.log.1`
- Previous: `event.log.1` → `event.log.2`
- Keep last N rotated logs

**Format**: JSON lines (one event per line)

**Events**:
- Intent executions
- Receipt creations
- State updates
- Errors

## Storage Architecture

### Rust Backend (Tauri)

**Files**: `src-tauri/src/lib.rs`, `src-tauri/src/receipt_log.rs`

**Responsibilities**:
- Encrypted file storage
- Receipt log management
- Identity vault management
- Cryptographic operations

**Storage Location** (platform-specific):
- **Linux**: `~/.local/share/frame/`
- **macOS**: `~/Library/Application Support/frame/`
- **Windows**: `%APPDATA%/frame/`

**Directory Structure**:
```
frame/
├── identities/
│   └── <identity_id>/
│       ├── keys/
│       │   └── ed25519_keypair.enc
│       └── storage/
│           ├── chain_<identity>.log
│           ├── journal_<identity>.log
│           ├── event.log
│           └── <key>.enc
└── .storage_key
```

### JavaScript Frontend

**File**: `ui/runtime/storage.js`

**Responsibilities**:
- Canonical JSON validation
- Storage abstraction
- Deep-freezing values
- Wrapping Tauri backend

**Interface**:
```javascript
window.FRAME_STORAGE = {
  read: async (key) => { /* ... */ },
  write: async (key, value) => { /* ... */ }
}
```

## Receipt Chain Storage

Receipt chains are stored in two formats:

### Legacy Format (Array)

**Key**: `chain:<identity>`

**Format**: Array of receipt objects

**Location**: `storage/chain_<identity>.enc`

**Migration**: Automatically migrated to log format on first write

### Log Format (Append-Only)

**File**: `storage/chain_<identity>.log`

**Format**: Frame-based append-only log

**Advantages**:
- Crash-safe appends
- Efficient sequential reads
- Checksum validation
- No full-chain rewrites

**Migration Process**:
1. Read legacy array format
2. Write each receipt to log file
3. Verify checksums
4. Remove legacy file

## State Root Computation

State root includes storage state:

**Components**:
- Identity public key
- Installed dApps (with code hashes)
- Storage keys/values (canonicalized)
- Receipt chain commitment

**Excluded Prefixes**:
- `chain:` (included via commitment)
- `frame_integrity`
- `frame_context_`
- `frame_layout`
- `frame_widgets`
- `frame_capsules`
- `frame_logs`
- `frame_ui`

**Computation**:
```javascript
var stateRoot = {
  version: STATE_ROOT_VERSION,
  capabilityVersion: CAPABILITY_VERSION,
  identityPublicKey: publicKey,
  installedDApps: dapps,  // Sorted by id
  storage: storage,  // Sorted keys
  receiptChainCommitment: commitment
};
var hash = sha256(JSON.stringify(canonicalize(stateRoot)));
```

## Crash Safety

FRAME ensures crash safety through:

### Atomic Writes

**Receipt Log**:
1. Write to temporary file
2. `fsync()` to disk
3. Atomic rename

**State Storage**:
1. Write to temporary file
2. `fsync()` to disk
3. Atomic rename

### Transaction Journal

Tracks in-flight transactions:
- Records pending writes
- Replays on recovery
- Ensures atomicity

### Checksums

Receipt log frames include checksums:
- Detects corruption
- Validates integrity
- Enables recovery

## Storage Migration

FRAME supports migration from legacy formats:

### Receipt Chain Migration

**From**: Array format (`chain:<identity>`)

**To**: Log format (`chain_<identity>.log`)

**Process**:
1. Detect legacy format
2. Read all receipts
3. Write to log file
4. Verify integrity
5. Remove legacy file

### Storage Key Migration

**From**: Unencrypted storage

**To**: Encrypted storage

**Process**:
1. Read unencrypted keys
2. Encrypt with storage key
3. Write to encrypted files
4. Remove unencrypted files

## Storage Security

### Encryption

All storage is encrypted at rest:

**Storage Key**: Generated per-installation, stored in `.storage_key`

**Encryption**: AES-GCM with 256-bit keys

**Key Derivation**: From master key + identity ID

### Access Control

Storage access controlled by:
- Identity isolation (per-identity storage)
- Capability grants (dApp permissions)
- Runtime validation (canonical JSON only)

### Integrity

Storage integrity ensured by:
- Checksums (receipt log)
- State root verification
- Code hash verification
- Receipt chain linking

## Storage Performance

### Optimization Strategies

1. **Receipt Chain Commitment**: Rolling hash instead of full chain
2. **State Root Caching**: Cache computed state roots
3. **Lazy Loading**: Load receipts on demand
4. **Batch Writes**: Group storage writes

### Storage Limits

- **Receipt Log**: Unbounded (append-only)
- **State Storage**: Unbounded (per-key)
- **Event Log**: Rotated at size limit
- **Transaction Journal**: Cleared after commit

## Storage API

### JavaScript Interface

```javascript
// Read
var value = await window.FRAME_STORAGE.read(key);

// Write
await window.FRAME_STORAGE.write(key, value);

// Receipt operations (via Tauri)
await window.__FRAME_INVOKE__('receipt_append', { identity, receipt });
var receipts = await window.__FRAME_INVOKE__('receipt_read_all', { identity });
```

### Rust Interface

```rust
// Append receipt
pub fn receipt_append_internal(
    app: &AppHandle,
    identity_id: &str,
    receipt: &serde_json::Value,
) -> Result<(), String>

// Read receipts
pub fn receipt_read_all(
    app: &AppHandle,
    identity_id: &str,
) -> Result<Vec<serde_json::Value>, String>
```

## Storage Guarantees

FRAME storage provides:

1. **Crash Safety**: Atomic writes, checksums, journal recovery
2. **Integrity**: Canonical JSON validation, state root verification
3. **Isolation**: Per-identity storage, capability-scoped access
4. **Replay Safety**: Receipt chain enables full state reconstruction
5. **Encryption**: All data encrypted at rest

These guarantees ensure that FRAME can reconstruct entire application sessions from receipt chains, even after crashes or corruption.
