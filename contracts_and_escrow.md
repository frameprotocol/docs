# FRAME Contracts and Escrow

FRAME supports deterministic escrow for contract execution.

**File:** `ui/dapps/contracts/index.js`

## Escrow Model

Escrow locks frame tokens until contract conditions are met.

**Intents:**

- `escrow.lock` — Lock frame in escrow
- `escrow.release` — Release escrowed frame to recipient
- `escrow.refund` — Refund escrowed frame to sender

## Escrow Process

### Lock

**Intent:** `escrow.lock`

**Parameters:**

- `reservation` — Reservation object with:
  - `amount` — Amount to lock
  - `recipient` — Recipient address
  - `contractId` — Contract identifier
  - `expiry` — Expiration timestamp

**Process:**

1. Check available balance
2. Create reservation
3. Lock frame in escrow
4. Store reservation in storage
5. Return reservation ID

### Release

**Intent:** `escrow.release`

**Parameters:**

- `reservationId` — Reservation identifier

**Process:**

1. Load reservation
2. Verify contract conditions
3. Transfer frame to recipient
4. Remove reservation
5. Generate receipt

### Refund

**Intent:** `escrow.refund`

**Parameters:**

- `reservationId` — Reservation identifier

**Process:**

1. Load reservation
2. Verify expiry or cancellation
3. Return frame to sender
4. Remove reservation
5. Generate receipt

## Contract Execution

Contracts use escrow for payment:

1. **Lock** — Buyer locks payment
2. **Execute** — Contract executes
3. **Release** — Payment released on success
4. **Refund** — Payment refunded on failure

## Storage

**Keys:**

- `escrow:<reservationId>` — Reservation data
- `escrow_index` — Reservation index

**Reservation structure:**

```javascript
{
  id: "reservation_id",
  amount: 100,
  recipient: "recipient_id",
  sender: "sender_id",
  contractId: "contract_id",
  expiry: 1234567890,
  status: "locked" | "released" | "refunded"
}
```

## Ride State Machine

**File:** `ui/runtime/contracts.js`

FRAME includes a deterministic ride event model for location-based contracts.

### Valid Ride Statuses

```javascript
VALID_RIDE_STATUSES = [
  'requested',
  'driver_enroute',
  'arrived',
  'in_progress',
  'completed'
]
```

### Valid Ride Transitions

```javascript
VALID_RIDE_TRANSITIONS = {
  'requested': 'driver_enroute',
  'driver_enroute': 'arrived',
  'arrived': 'in_progress',
  'in_progress': 'completed'
}
```

### Ride Operations

**Update Driver Location:**

- `updateDriverLocation(contractId, location)` — Update driver location
- Validates location (lat/lng finite numbers)
- Increments revision
- Enforces participant role (driver)
- Uses `handleStructuredIntent` for state transition

**Update Rider Location:**

- `updateRiderLocation(contractId, location)` — Update rider location
- Validates location (lat/lng finite numbers)
- Increments revision
- Enforces participant role (rider)
- Uses `handleStructuredIntent` for state transition

**Transition Ride Status:**

- `transitionRideStatus(contractId, newStatus)` — Transition ride status
- Validates transition (must be in `VALID_RIDE_TRANSITIONS`)
- Validates current state
- Enforces participant role
- Uses `handleStructuredIntent` for state transition

### Location Validation

**Location structure:**

```javascript
{
  lat: number,  // Finite number
  lng: number,  // Finite number
  timestamp?: number  // Optional timestamp
}
```

**Validation:**

- `lat` must be finite number
- `lng` must be finite number
- `timestamp` optional, must be number if present

### State Machine Properties

- **Deterministic** — Same inputs produce same transitions
- **Revisioned** — Each update increments revision
- **Role-enforced** — Only participants can update
- **Canonicalized** — All state canonicalized before storage

## Related Documentation

- [Wallet and Bridge](wallet_and_bridge.md) - Wallet operations
- [dApp Model](dapp_model.md) - Contracts dApp
