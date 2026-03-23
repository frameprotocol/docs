# FRAME Wallet and Bridge

FRAME includes a deterministic wallet system with frame (FRAME Credit) tokens and bridge operations.

**File:** `ui/dapps/wallet/index.js`

## frame (FRAME Credit)

frame is the single deterministic asset in FRAME.

**Properties:**

- **Total supply** — Tracked globally in storage (`wallet:totalSupply`)
- **Deterministic** — All operations are deterministic
- **Single asset** — No multi-token support

## Wallet Operations

### Balance

**Intent:** `wallet.balance`

**Returns:** Current frame balance

**Process:**

1. Read balance from storage (`wallet:balance:<identity>`)
2. Subtract locked escrow reservations
3. Return available balance

### Send

**Intent:** `wallet.send`

**Parameters:**

- `recipient` — Recipient address/identity
- `amount` — Amount to send

**Process:**

1. Check balance
2. Validate recipient
3. Deduct from sender balance
4. Add to recipient balance
5. Generate receipt

### Mint frame

**Intent:** `wallet.mintFrame` (also accepts `wallet.mintFC` for compatibility)

**Capability:** `bridge.mint`

**Parameters:**

- `amount` — Amount to mint
- `target` — Target address

**Process:**

1. Verify `bridge.mint` capability
2. Increase total supply
3. Add to target balance
4. Generate receipt

### Burn frame

**Intent:** `wallet.burnFrame` (also accepts `wallet.burnFC` for compatibility)

**Capability:** `bridge.burn`

**Parameters:**

- `amount` — Amount to burn
- `target` — Target address

**Process:**

1. Verify `bridge.burn` capability
2. Decrease total supply
3. Deduct from target balance
4. Generate receipt

## Escrow System

**Intents:** `escrow.lock`, `escrow.release`, `escrow.refund`

**Storage:** Escrow reservations stored in `escrow:<reservationId>`

**Process:**

1. **Lock** — Reserve frame in escrow
2. **Release** — Transfer escrowed frame to recipient
3. **Refund** — Return escrowed frame to sender

**Available balance:** `wallet balance - sum(locked reservations)`

## Bridge Operations

Bridge operations change total supply:

- **Mint** — Increase supply (requires `bridge.mint`)
- **Burn** — Decrease supply (requires `bridge.burn`)

**Use cases:**

- Cross-chain deposits (mint)
- Cross-chain withdrawals (burn)

## Storage Keys

- `wallet:totalSupply` — Total frame supply
- `wallet:balance:<identity>` — Balance per identity
- `escrow:<reservationId>` — Escrow reservations
- `escrow_index` — Escrow index

## Related Documentation

- [dApp Model](dapp_model.md) - Wallet dApp structure
- [Capability System](capability_system.md) - Bridge capabilities
