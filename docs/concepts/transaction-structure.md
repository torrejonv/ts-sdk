# Transaction Structure

Understanding how Bitcoin transactions work and their representation in the BSV TypeScript SDK.

## Basic Transaction Components

A Bitcoin transaction consists of:

- **Inputs**: References to previous transaction outputs being spent
- **Outputs**: New transaction outputs being created
- **Metadata**: Version, lock time, and other transaction properties

## Transaction in the SDK

```typescript
import { Transaction } from '@bsv/sdk'

// Create a new transaction
const tx = new Transaction()

// Add inputs and outputs
tx.addInput({
  sourceTransaction: previousTx,
  sourceOutputIndex: 0,
  unlockingScript: unlockingScript
})

tx.addOutput({
  satoshis: 1000,
  lockingScript: lockingScript
})
```

## Key Concepts

### Inputs
- Reference previous transaction outputs (UTXOs)
- Include unlocking scripts to prove ownership
- Must be fully consumed (no partial spending)

### Outputs
- Create new UTXOs with specific values
- Include locking scripts that define spending conditions
- Can be spent by future transactions

### Transaction ID
- Unique identifier calculated from transaction data
- Used to reference the transaction in inputs

## Working with Transactions

The SDK provides methods to:
- Serialize transactions to hex format
- Calculate transaction fees
- Verify transaction validity
- Sign transaction inputs

## Next Steps

- Learn about [BEEF Format](./beef.md) for efficient transaction representation with validation data
- Learn about [Script Templates](./script-templates.md) for locking/unlocking scripts
- Understand [Digital Signatures](./signatures.md) for transaction authorization
- Explore [Transaction Encoding](./transaction-encoding.md) for serialization formats
