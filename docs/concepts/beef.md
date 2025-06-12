# BEEF Format

Bitcoin Extras Extension Format (BEEF) - an efficient way to package Bitcoin transactions with their verification data.

## What is BEEF?

BEEF is a standardized format that combines:
- **Transaction Data**: The actual Bitcoin transaction
- **Merkle Proofs**: SPV verification data
- **Block Headers**: Chain validation information
- **Metadata**: Additional context and references

## BEEF in the SDK

```typescript
import { Transaction } from '@bsv/sdk'

// Create transaction with BEEF data
const tx = Transaction.fromHexBEEF(beefHex)

// Serialize transaction to BEEF
const beefData = transaction.toBEEF()

// Verify transaction using included proofs
const isValid = await tx.verify(chainTracker)
```

## Key Benefits

### Efficiency
- **Compact**: Includes only necessary verification data
- **Self-Contained**: No external lookups required
- **Batch Processing**: Multiple transactions in one package

### SPV Integration
- **Merkle Proofs**: Verify transaction inclusion
- **Block Headers**: Validate proof of work
- **Chain Context**: Understand transaction position

### Interoperability
- **Standardized**: Consistent format across applications
- **Portable**: Easy to transmit and store
- **Compatible**: Works with SPV clients

## Use Cases

### Transaction Broadcasting
```typescript
// Broadcast transaction with proof
const beefTx = Transaction.fromHexBEEF(beefData)
await beefTx.broadcast(arcConfig)
```

### Data Exchange
- Share transactions between applications
- Provide verification data to SPV clients
- Archive transactions with proofs

### Wallet Integration
- Import transactions with full context
- Verify historical transactions
- Synchronize between devices

## BEEF Structure

The format includes:
1. **Version**: BEEF format version
2. **Transactions**: One or more Bitcoin transactions
3. **Proofs**: Merkle proofs for each transaction
4. **Headers**: Relevant block headers
5. **Metadata**: Additional application data

## Best Practices

- Use BEEF for transactions that need verification
- Include minimal necessary proof data
- Validate BEEF structure before processing
- Cache parsed BEEF data for performance

## Next Steps

- Learn about [SPV Verification](./spv-verification.md) concepts
- Understand [Transaction Encoding](./transaction-encoding.md) formats
- Explore [Chain Tracking](./chain-tracking.md) integration
