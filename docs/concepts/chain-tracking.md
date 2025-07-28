# Chain Tracking

Understanding how the BSV TypeScript SDK verifies blockchain integrity through merkle root validation.

## What is a Chain Tracker?

A Chain Tracker in the BSV SDK serves a specific and crucial purpose: **verifying that merkle roots belong to legitimate blocks at specific heights**. This is the foundation of Simplified Payment Verification (SPV).

The Chain Tracker interface is intentionally minimal, providing only two essential functions:

```typescript
interface ChainTracker {
  // Verify a merkle root is valid for a specific block height
  isValidRootForHeight(root: string, height: number): Promise<boolean>
  
  // Get the current blockchain height
  currentHeight(): Promise<number>
}
```

## Why These Two Functions?

### Merkle Root Verification

The merkle root is a cryptographic fingerprint of all transactions in a block. By verifying that a merkle root belongs to a specific block height, we can:

- **Prove Transaction Inclusion**: Verify that a transaction exists in the blockchain without downloading full blocks
- **Prevent Fraud**: Ensure merkle proofs point to legitimate blocks, not fabricated ones
- **Enable SPV**: Allow lightweight clients to verify transactions trustlessly

### Current Height

Knowing the current blockchain height allows applications to:

- **Assess Confirmation Depth**: Calculate how many blocks have been mined after a transaction
- **Verify Recency**: Ensure proofs reference recent, not outdated, blocks
- **Track Progress**: Monitor blockchain growth and synchronization status

## The SPV Connection

Chain Trackers are the cornerstone of SPV (Simplified Payment Verification). When verifying a transaction:

1. **Transaction provides a Merkle path** linking it to a merkle root
2. **Chain Tracker verifies the merkle root** belongs to a legitimate block
3. **Height information confirms** the block's position in the chain

This creates a chain of trust from transaction to blockchain without downloading full blocks.

## Implementation Example

```typescript
import { WhatsOnChain, MerklePath } from '@bsv/sdk'

// Create a chain tracker
const chainTracker = new WhatsOnChain('main')

// Verify a merkle proof
async function verifyTransaction(txid: string, merklePath: MerklePath) {
  // Extract the merkle root from the proof
  const merkleRoot = merklePath.computeRoot(txid)
  
  // Verify this root exists at the claimed height
  const isValid = await chainTracker.isValidRootForHeight(
    merkleRoot,
    merklePath.blockHeight
  )
  
  if (isValid) {
    // Transaction is provably in the blockchain
    const currentHeight = await chainTracker.currentHeight()
    const confirmations = currentHeight - merklePath.blockHeight + 1
    console.log(`Transaction confirmed with ${confirmations} blocks`)
  }
}
```

## Trust Model

Chain Trackers implement different trust models:

### Service-Based (WhatsOnChain)

Trusts a specific service provider to accurately report merkle roots and heights:

```typescript
const tracker = new WhatsOnChain('main')
// Trusts WhatsOnChain API for merkle root verification
```

### Headers-Based

Maintains and verifies a chain of block headers locally:

```typescript
// Theoretical implementation that verifies headers
class HeadersChainTracker implements ChainTracker {
  private headers: Map<number, BlockHeader>
  
  async isValidRootForHeight(root: string, height: number) {
    const header = this.headers.get(height)
    return header?.merkleRoot === root
  }
}
```

### Consensus-Based

Could query multiple sources and require agreement:

```typescript
// Theoretical implementation using multiple sources
class ConsensusChainTracker implements ChainTracker {
  async isValidRootForHeight(root: string, height: number) {
    const results = await Promise.all([
      tracker1.isValidRootForHeight(root, height),
      tracker2.isValidRootForHeight(root, height),
      tracker3.isValidRootForHeight(root, height)
    ])
    // Require majority agreement
    return results.filter(r => r).length >= 2
  }
}
```

## Common Misconceptions

### What Chain Trackers DON'T Do

Chain Trackers in the BSV SDK do **not**:

- Retrieve transaction data
- Query UTXOs (unspent transaction outputs)
- Submit transactions to the network
- Provide fee estimates
- Store blockchain data

These functions may be provided by other services or implementations, but they are **not** part of the Chain Tracker interface.

### Separation of Concerns

The minimal Chain Tracker interface follows the principle of separation of concerns:

- **Chain Tracker**: Verifies merkle roots and tracks height
- **Transaction Service**: Retrieves and submits transactions
- **UTXO Service**: Manages unspent outputs
- **Fee Service**: Estimates appropriate fees

This separation allows for flexible, composable architectures.

## Security Implications

### Critical for SPV Security

Merkle root verification is the linchpin of SPV security. Without it:

- Anyone could create fake merkle proofs
- Transactions could be falsely claimed as confirmed
- The trustless nature of SPV would be compromised

### Trust Requirements

Using a Chain Tracker requires trusting that it:

1. **Accurately reports merkle roots** from the legitimate chain
2. **Follows the chain with most proof-of-work**
3. **Provides current height** information reliably

Different implementations offer different trust trade-offs.

## Best Practices

### Use Multiple Sources

For critical applications, consider verifying against multiple chain trackers:

```typescript
async function verifyWithMultipleSources(root: string, height: number) {
  const trackers = [
    new WhatsOnChain('main'),
    // Add other implementations
  ]
  
  const results = await Promise.all(
    trackers.map(t => t.isValidRootForHeight(root, height))
  )
  
  // All must agree
  return results.every(r => r === true)
}
```

### Handle Failures Gracefully

Chain tracker queries can fail due to network issues:

```typescript
try {
  const isValid = await chainTracker.isValidRootForHeight(root, height)
} catch (error) {
  // Handle network errors, retry logic, fallback to other trackers
  console.error('Chain tracker unavailable:', error)
}
```

### Cache Results

Merkle roots for historical blocks don't change:

```typescript
class CachedChainTracker implements ChainTracker {
  private cache = new Map<string, boolean>()
  
  async isValidRootForHeight(root: string, height: number) {
    const key = `${height}:${root}`
    
    if (this.cache.has(key)) {
      return this.cache.get(key)!
    }
    
    const result = await this.baseTracker.isValidRootForHeight(root, height)
    this.cache.set(key, result)
    return result
  }
}
```

## Conclusion

Chain Trackers provide the essential link between merkle proofs and the blockchain. By verifying merkle roots and tracking blockchain height, they enable SPV - allowing lightweight clients to verify transactions without downloading the entire blockchain.

The intentionally minimal interface (just two methods) reflects the focused purpose: enabling trustless transaction verification through merkle root validation. This is the foundation upon which all SPV-based applications are built.

## Related Concepts

- [SPV and Merkle Proofs](./spv.md) - How merkle proof verification enables lightweight validation
- [Verification](./verification.md) - The broader concept of trustless validation
- [Transaction Structure](./transactions.md) - How transactions connect to merkle trees
