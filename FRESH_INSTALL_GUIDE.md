# FRAME Fresh Install Guide

This guide explains how to run FRAME on a new computer as if it's the first time, by clearing all persisted state, cache, and storage.

## Overview

FRAME stores data in two main locations:
1. **Tauri Backend Storage** (Rust) - Encrypted filesystem storage
2. **Browser Storage** (JavaScript) - localStorage, IndexedDB, Service Worker caches

## Tauri App Data Directory

### Location by Platform

**Linux:**
```
~/.local/share/app.frame.local/
```

**macOS:**
```
~/Library/Application Support/app.frame.local/
```

**Windows:**
```
%APPDATA%\app.frame.local\
```

### Directory Structure

```
app.frame.local/
├── identities/              # All identities and their data
│   └── <identity_id>/       # Per-identity directory
│       ├── keys.enc         # Encrypted Ed25519 keypair
│       ├── .key             # Storage encryption key
│       ├── .storage_key     # Storage encryption key (alternative)
│       ├── meta.json        # Identity metadata (capabilities)
│       ├── storage/          # Encrypted storage files
│       │   ├── chain:<identity>.enc      # Receipt chain
│       │   ├── permissions:<identity>.enc  # Permission grants
│       │   └── <other_keys>.enc          # Other storage data
│       └── event.log        # Event log (if enabled)
├── current_identity         # File containing current identity ID
└── (other app files)
```

### What to Delete for Fresh Install

**Option 1: Complete Fresh Start (Recommended)**
Delete the entire app data directory:
```bash
# Linux
rm -rf ~/.local/share/app.frame.local/

# macOS
rm -rf ~/Library/Application\ Support/app.frame.local/

# Windows
rmdir /s "%APPDATA%\app.frame.local"
```

**Option 2: Keep App Config, Clear Data**
Delete only the `identities/` directory:
```bash
# Linux
rm -rf ~/.local/share/app.frame.local/identities/

# macOS
rm -rf ~/Library/Application\ Support/app.frame.local/identities/

# Windows
rmdir /s "%APPDATA%\app.frame.local\identities"
```

**Option 3: Clear Specific Identity**
Delete a specific identity's directory:
```bash
# Linux
rm -rf ~/.local/share/app.frame.local/identities/<identity_id>/

# macOS
rm -rf ~/Library/Application\ Support/app.frame.local/identities/<identity_id>/

# Windows
rmdir /s "%APPDATA%\app.frame.local\identities\<identity_id>"
```

## Browser Storage

### localStorage Keys

FRAME uses the following localStorage keys (all prefixed with `frame_`):

| Key | Purpose |
|-----|---------|
| `frame_widgets` | Widget state and configuration |
| `frame_layout` | UI layout (regions, grid columns) |
| `frame_context_rolling` | Context memory (ephemeral) |
| `frame_context_archive` | Archive pointer (ephemeral) |
| `frame_theme` | Theme settings |
| `frame_installed_dapps` | List of installed dApps |
| `frame_peer_router` | Peer router table |
| `frame_route_history` | Route history |
| `frame_intent_market` | Intent market data |
| `frame_credit_ledger` | Credit ledger |
| `frame_peer_reputation` | Peer reputation scores |

### How to Clear Browser Storage

**Option 1: Clear All localStorage (Recommended)**
```javascript
// In browser console (F12)
localStorage.clear();
sessionStorage.clear();
```

**Option 2: Clear Only FRAME Keys**
```javascript
// In browser console (F12)
const frameKeys = [
  'frame_widgets',
  'frame_layout',
  'frame_context_rolling',
  'frame_context_archive',
  'frame_theme',
  'frame_installed_dapps',
  'frame_peer_router',
  'frame_route_history',
  'frame_intent_market',
  'frame_credit_ledger',
  'frame_peer_reputation'
];
frameKeys.forEach(key => localStorage.removeItem(key));
```

### IndexedDB

FRAME may use IndexedDB for caching. Clear it:

```javascript
// In browser console (F12)
if (window.indexedDB && indexedDB.databases) {
  const dbs = await indexedDB.databases();
  for (const db of dbs) {
    indexedDB.deleteDatabase(db.name);
  }
} else {
  // Fallback: delete known database
  indexedDB.deleteDatabase('frame');
}
```

### Service Worker Caches

Clear service worker caches:

```javascript
// In browser console (F12)
if (window.caches) {
  const keys = await caches.keys();
  for (const key of keys) {
    await caches.delete(key);
  }
}
```

## Automated Reset (Development Mode)

FRAME includes a built-in reset function for development:

**Method 1: Via Browser Console**
```javascript
// Only works in dev mode (localhost or FRAME_DEV_MODE=true)
if (window.FRAME_DEV_RESET) {
  await window.FRAME_DEV_RESET.resetRuntime();
}
```

**Method 2: Via UI Button**
- In development mode, a reset button appears in the UI
- Click it to clear all runtime state and reload

**What it clears:**
- All FRAME_STORAGE data (via `FRAME_STORAGE.clear()`)
- All localStorage
- All sessionStorage
- All IndexedDB databases
- All Service Worker caches
- Then reloads the page

## Complete Fresh Install Steps

### Step 1: Close FRAME Application
Make sure FRAME is completely closed before deleting files.

### Step 2: Delete Tauri App Data
```bash
# Linux
rm -rf ~/.local/share/app.frame.local/

# macOS  
rm -rf ~/Library/Application\ Support/app.frame.local/

# Windows
rmdir /s "%APPDATA%\app.frame.local"
```

### Step 3: Clear Browser Storage (if running in browser)
If you're running FRAME in a browser (not as Tauri app):
1. Open browser DevTools (F12)
2. Go to Application/Storage tab
3. Clear:
   - Local Storage
   - Session Storage
   - IndexedDB
   - Service Workers
   - Cache Storage

### Step 4: Launch FRAME
Launch FRAME. It will:
- Create a new app data directory
- Prompt you to create your first identity
- Start with empty storage and no widgets

## What Gets Reset

After a fresh install, FRAME will have:

✅ **Empty State:**
- No identities (you'll need to create one)
- No receipt chains
- No permissions granted
- No widgets
- No layout preferences
- No theme settings
- No installed dApps (beyond built-in ones)
- No peer connections
- No route history

✅ **Default State:**
- Built-in dApps available (wallet, notes, contacts, etc.)
- Default UI layout
- Default theme
- Empty capability registry

## Verification

After fresh install, verify:

1. **No identities exist:**
   ```javascript
   // In browser console
   await window.__FRAME_INVOKE__('list_identities');
   // Should return: []
   ```

2. **No storage keys:**
   ```javascript
   // In browser console
   await window.__FRAME_INVOKE__('storage_list_keys');
   // Should return: []
   ```

3. **No localStorage:**
   ```javascript
   // In browser console
   Object.keys(localStorage).filter(k => k.startsWith('frame_'));
   // Should return: []
   ```

## Troubleshooting

### "App data directory not found"
This is normal for a fresh install. FRAME will create it on first launch.

### "Identity already exists"
If you see this error, the app data directory wasn't fully cleared. Delete it again and restart.

### "Storage keys still present"
Make sure you cleared both:
1. Tauri app data directory (`identities/` folder)
2. Browser localStorage (if running in browser)

### "Widgets still appear"
Widgets are stored in localStorage. Make sure you cleared `frame_widgets` key.

## Notes

- **Backup First**: If you want to preserve data, backup the app data directory before deleting
- **Cross-Platform**: App data directory location differs by OS, but structure is the same
- **Encryption**: All Tauri storage files are encrypted with AES-GCM
- **Identity Isolation**: Each identity has completely isolated storage
- **Ephemeral Data**: Some localStorage keys (`frame_context_*`) are ephemeral and don't affect state root

## Related Documentation

- [Storage Model](storage_model.md) - How FRAME stores data
- [Identity and Vaults](identity_and_vaults.md) - Identity system details
- [Boot Sequence](boot_sequence.md) - What happens on first launch
