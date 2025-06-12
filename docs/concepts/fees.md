# Transaction Fees

Understanding Bitcoin transaction fees and how they work in the BSV TypeScript SDK.

## What are Transaction Fees?

Transaction fees are payments to miners for including transactions in blocks:

```typescript
import { Transaction } from '@bsv/sdk'

// Calculate transaction fee
const tx = new Transaction()
const feeRequired = tx.getFee()

// Set custom fee rate
const customFee = tx.getFee(feePerKB)
```

## Fee Calculation

Fees are calculated based on transaction size:
- **Fee Rate**: Satoshis per kilobyte (sat/kB)
- **Transaction Size**: Total bytes in serialized transaction
- **Total Fee**: Size × Fee Rate

## SDK Fee Handling

### Automatic Fee Calculation
The SDK calculates fees automatically:
```typescript
// Wallet handles fees automatically
const wallet = new WalletClient()
const action = await wallet.createAction({
  outputs: [{
    satoshis: 1000,
    lockingScript: script
  }]
})
// Fee calculated and deducted automatically
```

### Manual Fee Control
For advanced use cases:
```typescript
// Calculate fee manually
const tx = new Transaction()
tx.addInput(/* input */)
tx.addOutput(/* output */)

const estimatedSize = tx.getSerializedSize()
const feeRequired = estimatedSize * feePerByte
```

## Fee Rates

### Network Fee Rates
BSV typically uses low, predictable fees:
- **Standard Rate**: ~0.5 sat/byte
- **Priority Rate**: ~1.0 sat/byte
- **Economy Rate**: ~0.1 sat/byte

### Dynamic Fee Estimation
```typescript
// Get current network fee rates
const feeRates = await chainTracker.getFeeRates()
const recommendedFee = feeRates.standard
```

## Fee Components

### Input Costs
Each input adds to transaction size:
- Previous transaction hash (32 bytes)
- Output index (4 bytes)
- Unlocking script (variable)
- Sequence number (4 bytes)

### Output Costs
Each output adds to transaction size:
- Value (8 bytes)
- Locking script length (1-9 bytes)
- Locking script (variable)

### Base Transaction
Fixed overhead for every transaction:
- Version (4 bytes)
- Input count (1-9 bytes)
- Output count (1-9 bytes)
- Lock time (4 bytes)

## Fee Optimization

### UTXO Selection
Choose UTXOs efficiently:
```typescript
// Prefer fewer, larger UTXOs to reduce fees
const utxos = await wallet.getUTXOs()
const selected = selectOptimalUTXOs(utxos, targetAmount)
```

### Output Consolidation
Combine multiple payments:
```typescript
// Batch multiple outputs in one transaction
const outputs = [
  { satoshis: 1000, lockingScript: script1 },
  { satoshis: 2000, lockingScript: script2 },
  { satoshis: 1500, lockingScript: script3 }
]
```

### Script Efficiency
Use efficient script templates:
```typescript
// P2PKH is more efficient than complex scripts
const p2pkh = new P2PKH()
const efficientScript = p2pkh.lock(publicKeyHash)
```

## Fee Estimation

### Size Estimation
Estimate transaction size before creation:
```typescript
// Estimate size for fee calculation
const estimatedInputs = 2
const estimatedOutputs = 3
const estimatedSize = estimateTransactionSize(estimatedInputs, estimatedOutputs)
const estimatedFee = estimatedSize * feeRate
```

### Template-Based Estimation
```typescript
// Use script templates for accurate estimation
const template = new P2PKH()
const scriptSize = template.estimateLength([publicKeyHash])
```

## Error Handling

Common fee-related issues:
- Insufficient funds for fees
- Fee rate too low for network
- Transaction size miscalculation

```typescript
try {
  const action = await wallet.createAction({
    outputs: outputs
  })
} catch (error) {
  if (error.message.includes('Insufficient funds')) {
    // Handle insufficient balance for fees
    console.log('Not enough funds to cover transaction and fees')
  }
}
```

## Best Practices

### Fee Management
- Always account for fees in balance calculations
- Use appropriate fee rates for urgency
- Monitor network conditions for fee adjustments
- Implement fee estimation before transaction creation

### Cost Optimization
- Batch transactions when possible
- Use efficient script templates
- Optimize UTXO selection
- Consider transaction timing

### User Experience
- Display estimated fees to users
- Provide fee rate options (economy, standard, priority)
- Handle insufficient fund errors gracefully
- Show fee breakdown for transparency

## Wallet Integration

Most applications rely on wallets for fee handling:
```typescript
// Wallet manages fees automatically
const wallet = new WalletClient()

// Fees are calculated and deducted automatically
const result = await wallet.createAction({
  description: 'Payment with automatic fees',
  outputs: [/* outputs */]
})
```

## Advanced Fee Strategies

### Replace-by-Fee (RBF)
Increase fees for faster confirmation:
```typescript
// Create transaction with RBF enabled
const tx = new Transaction()
tx.setRBF(true) // Enable replace-by-fee
```

### Child-Pays-for-Parent (CPFP)
Use dependent transactions to increase effective fee rate:
```typescript
// Create child transaction with higher fee
const childTx = new Transaction()
childTx.addInput(/* from parent transaction */)
// Higher fee rate pulls parent transaction along
```

## Next Steps

- Understand [Transaction Structure](./transaction-structure.md) for size calculation
- Learn about [Wallet Integration](./wallet-integration.md) for automatic fee handling
- Explore [Script Templates](./script-templates.md) for efficient fee optimization
