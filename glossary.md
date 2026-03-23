# FRAME Glossary

## A

**Attestation** — Cryptographic proof of state root and receipt chain commitment, signed by identity. Used for cross-instance verification.

## C

**Canonical JSON** — JSON format with sorted keys, safe integers only, no floats/NaN/Infinity, no functions or prototypes. Ensures deterministic serialization.

**Capability** — Privileged operation that dApps must declare and be granted permission to use. Examples: `wallet.send`, `storage.read`.

**Capability Guard** — Function that verifies dApp has declared a capability before allowing its use.

**Capability Registry** — System that scans dApp manifests, builds capability indexes, and scores providers.

**Composite Execution** — Atomic execution of multiple intents as a single transaction with rollback on failure.

## D

**dApp** — Self-contained application that executes within FRAME's capability-constrained runtime. Declares intents and capabilities in `manifest.json`.

**Deterministic Execution** — Execution that produces identical outputs for identical inputs. Required for replay and verification.

**Deterministic Sandbox** — Execution environment with frozen time, seeded randomness, and blocked async APIs.

## E

**Ed25519** — Elliptic curve signature scheme used for identity signatures and receipt signing.

**Escrow** — System for locking frame tokens until contract conditions are met.

**Execution Graph** — Dependency-ordered graph of intents for composite execution.

## F

**frame (FRAME Credit)** — Single deterministic asset in FRAME. Total supply tracked globally.

**Federation** — Peer-to-peer synchronization via receipt chain exchange.

## I

**Identity** — Ed25519 keypair with associated metadata. Each identity has isolated storage and receipt chain.

**Intent** — Structured request to perform an action. Contains `action`, `payload`, `raw`, `timestamp`.

**Integrity Lock** — Boot-time state root and code hash capture. Used to detect runtime drift.

## K

**Kernel** — Core runtime engine (`engine.js`) that processes intents, enforces capabilities, and generates receipts.

## M

**Manifest** — dApp declaration file (`manifest.json`) containing metadata, intents, and capabilities.

## N

**Network Request** — Deterministic network operation via `network.request` capability. Responses sealed in receipts.

## P

**Protocol Version** — Version constants (`PROTOCOL_VERSION`, `RECEIPT_VERSION`, `CAPABILITY_VERSION`, `STATE_ROOT_VERSION`) that control compatibility.

## R

**Receipt** — Cryptographic certificate of state transition. Contains input hash, result hash, state roots, capabilities used, and Ed25519 signature.

**Receipt Chain** — Append-only chain of receipts linked via `previousReceiptHash`. Forms cryptographic execution log.

**Receipt Chain Commitment** — Hash of latest receipt in chain. Included in state root.

**Reconstruction Mode** — Execution mode for replay verification. No receipts appended, no state mutations persisted.

**Replay** — Deterministic re-execution of receipt chain for verification and state reconstruction.

**Router** — System (`router.js`) that resolves intents to dApps and executes them with scoped APIs.

## S

**Scoped API** — Restricted API object containing only declared capabilities. Built by kernel for each dApp execution.

**Snapshot** — Complete export of FRAME state including receipt chain, storage, and protocol versions.

**State Root** — Deterministic SHA-256 hash representing entire system state. Computed from identity, dApps, storage, and receipt chain commitment.

**Storage** — Canonical JSON storage layer with validation and deep freeze.

## V

**Vault** — Per-identity storage system. Each identity has isolated, encrypted storage.

**Verification** — Process of verifying receipt chain integrity, signatures, and state roots.

## W

**Widget** — UI component rendered in layout regions. Types: `data`, `status`, `list`, `chart`, `control`, `theme`.

## Related Documentation

- [Core Principles](core_principles.md) - Foundational concepts
- [Protocol Versions](protocol_versions.md) - Version constants
- [FRAME Protocol Spec](FRAME_PROTOCOL_SPEC.md) - Protocol specification
