# Transaction Verification

Understanding how to verify Bitcoin transactions using the BSV TypeScript SDK.

## What is Transaction Verification?

Transaction verification ensures that Bitcoin transactions are valid and can be trusted:

```typescript
import { Transaction } from '@bsv/sdk'

// Verify a transaction
const isValid = await transaction.verify(chainTracker, {
  merkleProof: proof,
  blockHeader: header
})
```

## Verification Levels

### Basic Validation
Check transaction structure and format:
- Valid input/output formats
- Correct serialization
- Proper script syntax
- Signature format validation

### Script Execution
Verify that unlocking scripts satisfy locking scripts:
- Execute Bitcoin script opcodes
- Validate signature operations
- Check script conditions
- Ensure proper stack state

### SPV Verification
Confirm transaction inclusion in the blockchain:
- Verify merkle proofs
- Validate block headers
- Check proof of work
- Confirm transaction position

## SDK Verification Methods

### Transaction.verify()
Complete transaction verification:
```typescript
const result = await transaction.verify(chainTracker, {
  merkleProof: merkleProof,
  blockHeader: blockHeader,
  maxMemoryLimit: 100000000
})
```

### Script Verification
Verify individual scripts:
```typescript
const isValid = unlockingScript.verify(
  lockingScript,
  transaction,
  inputIndex
)
```

### Signature Verification
Check digital signatures:
```typescript
const publicKey = PrivateKey.fromWif(wif).toPublicKey()
const isValid = publicKey.verify(messageHash, signature)
```

## Verification Options

### Memory Limits
Control script execution memory usage:
```typescript
const options = {
  maxMemoryLimit: 50000000 // 50MB limit
}
```

### Scripts-Only Mode
Skip SPV verification for performance:
```typescript
const options = {
  scriptsOnly: true
}
```

### Custom Chain Tracker
Use specific data sources:
```typescript
const customTracker = new WhatsOnChain('testnet')
const isValid = await transaction.verify(customTracker)
```

## BEEF Verification

BEEF format includes verification data:
```typescript
// BEEF transactions include proofs
const beefTx = Transaction.fromHexBEEF(beefData)

// Verify using included proofs
const isValid = await beefTx.verify(chainTracker)
```

## Error Handling

Common verification failures:
- Invalid signatures
- Script execution errors
- Missing merkle proofs
- Network connectivity issues

```typescript
try {
  const isValid = await transaction.verify(chainTracker)
  if (!isValid) {
    console.log('Transaction verification failed')
  }
} catch (error) {
  console.error('Verification error:', error.message)
}
```

## Performance Considerations

### Batch Verification
Verify multiple transactions efficiently:
```typescript
const results = await Promise.all(
  transactions.map(tx => tx.verify(chainTracker))
)
```

### Caching
Cache verification results:
```typescript
const verificationCache = new Map()

if (!verificationCache.has(txid)) {
  const result = await transaction.verify(chainTracker)
  verificationCache.set(txid, result)
}
```

## Security Best Practices

### Always Verify
- Verify transactions before trusting them
- Check both script execution and SPV proofs
- Validate input data before verification

### Multiple Sources
- Use multiple chain trackers for redundancy
- Cross-check verification results
- Implement fallback mechanisms

### Resource Limits
- Set appropriate memory limits
- Timeout long-running verifications
- Monitor verification performance

## Common Use Cases

### Payment Verification
```typescript
// Verify received payment
const payment = Transaction.fromHex(paymentHex)
const isValid = await payment.verify(chainTracker)

if (isValid) {
  // Process confirmed payment
}
```

### Historical Transaction Audit
```typescript
// Verify old transactions
for (const txHex of historicalTransactions) {
  const tx = Transaction.fromHex(txHex)
  const result = await tx.verify(chainTracker)
  console.log(`Transaction ${tx.id()}: ${result ? 'Valid' : 'Invalid'}`)
}
```

## Integration Patterns

### Wallet Integration
```typescript
// Wallets typically handle verification
const wallet = new WalletClient()
const action = await wallet.createAction({
  outputs: [/* outputs */]
})
// Wallet verifies before broadcasting
```

### Application Verification
```typescript
// Applications verify received transactions
async function processIncomingTransaction(txHex: string) {
  const tx = Transaction.fromHex(txHex)
  
  if (await tx.verify(chainTracker)) {
    // Process verified transaction
    await handleValidTransaction(tx)
  } else {
    // Reject invalid transaction
    throw new Error('Invalid transaction received')
  }
}
```

## Next Steps

- Learn about [SPV Verification](./spv-verification.md) concepts
- Understand [Digital Signatures](./signatures.md) validation
- Explore [BEEF Format](./beef.md) for efficient verification
