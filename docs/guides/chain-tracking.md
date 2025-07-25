# Setting Up Chain Tracking

Learn how to configure and use chain trackers to interact with the Bitcoin SV blockchain for transaction data, UTXO queries, and SPV verification.

## Prerequisites

- Completed "Your First BSV Transaction" tutorial
- Understanding of Bitcoin transaction structure
- Basic knowledge of SPV concepts
- Understanding of `WalletClient` usage

> **📚 Related Concepts**: Review [Chain Tracking](../concepts/chain-tracking.md), [SPV Verification](../concepts/spv-verification.md), and [Transaction Verification](../concepts/verification.md) for foundational understanding.

## What is Chain Tracking?

Chain tracking provides access to Bitcoin blockchain data without running a full node. The BSV TypeScript SDK supports multiple chain tracker implementations for different data sources and use cases.

## Step 1: Basic Chain Tracker Setup

### Using WhatsOnChain

WhatsOnChain provides a chain tracker implementation for SPV verification:

```typescript
import { WhatsOnChain } from '@bsv/sdk'

// Create a chain tracker for mainnet
const chainTracker = new WhatsOnChain('main')

// For testnet
const testnetTracker = new WhatsOnChain('test')

// For STN (Scaling Test Network)
const stnTracker = new WhatsOnChain('stn')
```

### Basic Operations

```typescript
async function basicChainTracking() {
  const chainTracker = new WhatsOnChain('test')
  
  try {
    // Get current blockchain height
    const currentHeight = await chainTracker.currentHeight()
    console.log('Current blockchain height:', currentHeight)
    
    // Verify if a merkle root is valid for a specific block height
    const merkleRoot = 'your-merkle-root-here'
    const blockHeight = 800000
    const isValid = await chainTracker.isValidRootForHeight(merkleRoot, blockHeight)
    console.log(`Merkle root is ${isValid ? 'valid' : 'invalid'} for height ${blockHeight}`)
    
  } catch (error) {
    console.error('Chain tracking error:', error.message)
  }
}
```

## Step 2: Block Height Verification

Chain trackers are primarily used for SPV verification by checking merkle roots:

```typescript
async function blockHeightVerification() {
  const chainTracker = new WhatsOnChain('test')
  
  try {
    // Get the current blockchain height
    const currentHeight = await chainTracker.currentHeight()
    console.log(`Current blockchain height: ${currentHeight}`)
    
    // Verify merkle roots for specific block heights
    const testCases = [
      {
        merkleRoot: '4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b',
        height: 0 // Genesis block
      },
      {
        merkleRoot: 'invalid-root-for-testing',
        height: 1
      }
    ]
    
    for (const testCase of testCases) {
      const isValid = await chainTracker.isValidRootForHeight(
        testCase.merkleRoot, 
        testCase.height
      )
      console.log(`Block ${testCase.height}: ${isValid ? 'Valid' : 'Invalid'} merkle root`)
    }
    
  } catch (error) {
    console.error('Block verification error:', error.message)
  }
}
```

## Step 3: SPV Verification with Chain Trackers

Chain trackers provide merkle root verification for SPV (Simplified Payment Verification):

```typescript
import { MerklePath } from '@bsv/sdk'

async function spvVerification() {
  const chainTracker = new WhatsOnChain('test')
  
  try {
    // Example: Verify a merkle path against the blockchain
    // In practice, you would get the merkle path from BEEF data or other sources
    const merklePath = new MerklePath({
      blockHeight: 800000,
      path: [
        [{
          offset: 0,
          hash: 'transaction-hash-to-verify'
        }]
      ]
    })
    
    // Compute the merkle root from the path
    const merkleRoot = merklePath.computeRoot('transaction-hash-to-verify')
    
    // Verify the merkle root is valid for the block height
    const isValid = await chainTracker.isValidRootForHeight(
      merkleRoot, 
      merklePath.blockHeight
    )
    
    console.log(`Merkle path verification: ${isValid ? 'valid' : 'invalid'}`)
    
    // You can also use the MerklePath's built-in verify method
    const pathIsValid = await merklePath.verify('transaction-hash-to-verify', chainTracker)
    console.log(`MerklePath.verify result: ${pathIsValid ? 'valid' : 'invalid'}`)
    
  } catch (error) {
    console.error('SPV verification error:', error.message)
  }
}
```

## Step 4: Multiple Chain Tracker Providers

For production applications, consider using multiple providers for redundancy:

```typescript
class MultiChainTracker {
  private providers: WhatsOnChain[]
  
  constructor() {
    this.providers = [
      new WhatsOnChain('main'),
      new WhatsOnChain('test'), // Fallback to test network if needed
      // Add other ChainTracker implementations here
    ]
  }
  
  async getCurrentHeight(): Promise<number> {
    for (const provider of this.providers) {
      try {
        return await provider.currentHeight()
      } catch (error: any) {
        console.warn(`Provider failed: ${error.message}`)
        continue
      }
    }
    throw new Error('All providers failed')
  }
  
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    for (const provider of this.providers) {
      try {
        return await provider.isValidRootForHeight(root, height)
      } catch (error: any) {
        console.warn(`Provider failed: ${error.message}`)
        continue
      }
    }
    throw new Error('All providers failed')
  }
}

// Usage
async function redundantChainTracking() {
  const chainTracker = new MultiChainTracker()
  
  try {
    const height = await chainTracker.getCurrentHeight()
    console.log('Current height retrieved successfully:', height)
    
    const isValid = await chainTracker.isValidRootForHeight('some-merkle-root', height - 1)
    console.log('Root validation result:', isValid)
  } catch (error: any) {
    console.error('All providers failed:', error.message)
  }
}
```

## Step 5: Error Handling and Retry Logic

Implement robust error handling for network operations:

```typescript
async function robustChainTracking() {
  const chainTracker = new WhatsOnChain('main')
  const maxRetries = 3
  const retryDelay = 1000 // 1 second
  
  async function withRetry<T>(operation: () => Promise<T>): Promise<T> {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await operation()
      } catch (error: any) {
        if (attempt === maxRetries) {
          throw error
        }
        
        console.warn(`Attempt ${attempt} failed, retrying in ${retryDelay}ms...`)
        await new Promise(resolve => setTimeout(resolve, retryDelay))
      }
    }
    throw new Error('Max retries exceeded')
  }
  
  try {
    // Use retry logic for chain operations
    const currentHeight = await withRetry(() => 
      chainTracker.currentHeight()
    )
    
    const isValidRoot = await withRetry(() =>
      chainTracker.isValidRootForHeight('some-merkle-root', currentHeight - 1)
    )
    
    console.log('Operations completed successfully')
    console.log(`Current height: ${currentHeight}`)
    console.log(`Root validation: ${isValidRoot}`)
    
  } catch (error: any) {
    console.error('Chain tracking failed after retries:', error.message)
  }
}
```

## Step 6: Configuration and Performance

### Configuration Options

```typescript
// Configure with API key and custom HTTP client
const chainTracker = new WhatsOnChain('main', {
  apiKey: 'your-whatsonchain-api-key', // Optional API key for higher rate limits
  // httpClient: customHttpClient // Optional custom HTTP client
})
```

### Caching for Performance

```typescript
class CachedChainTracker {
  private cache = new Map<string, any>()
  private chainTracker: WhatsOnChain
  
  constructor(network: 'main' | 'test' | 'stn') {
    this.chainTracker = new WhatsOnChain(network)
  }
  
  async getCurrentHeight(): Promise<number> {
    const cacheKey = 'currentHeight'
    const cached = this.cache.get(cacheKey)
    
    // Cache height for 30 seconds
    if (cached && Date.now() - cached.timestamp < 30000) {
      return cached.value
    }
    
    const height = await this.chainTracker.currentHeight()
    this.cache.set(cacheKey, {
      value: height,
      timestamp: Date.now()
    })
    return height
  }
  
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    const cacheKey = `${root}-${height}`
    const cached = this.cache.get(cacheKey)
    
    if (cached) {
      return cached
    }
    
    const isValid = await this.chainTracker.isValidRootForHeight(root, height)
    this.cache.set(cacheKey, isValid)
    return isValid
  }
  
  clearCache() {
    this.cache.clear()
  }
}
```

## Step 7: Integration with Wallet Operations

Chain trackers work with wallet operations for SPV verification:

```typescript
import { WalletClient, Transaction } from '@bsv/sdk'

async function walletWithChainTracker() {
  const wallet = new WalletClient('auto', 'localhost')
  const chainTracker = new WhatsOnChain('main')
  
  try {
    // Create a transaction
    const action = await wallet.createAction({
      outputs: [{
        satoshis: 100,
        lockingScript: '006a0c48656c6c6f20436861696e21' // OP_RETURN "Hello Chain!"
      }]
    })
    
    // Sign and process the action
    const result = await wallet.signAction(action)
    console.log(`Transaction created: ${result.txid}`)
    
    // Get current blockchain height for context
    const currentHeight = await chainTracker.currentHeight()
    console.log(`Current blockchain height: ${currentHeight}`)
    
    // If you have a transaction with merkle path (from BEEF), you can verify it
    // This is typically done when receiving transactions from other parties
    if (result.tx) {
      try {
        // Verify the transaction using the chain tracker
        const isValid = await result.tx.verify(chainTracker)
        console.log(`Transaction verification: ${isValid ? 'valid' : 'invalid'}`)
      } catch (verifyError: any) {
        console.log('Transaction verification not possible (no merkle path):', verifyError.message)
      }
    }
    
    console.log('Wallet operation completed successfully')
    
  } catch (error: any) {
    console.error('Wallet operation error:', error.message)
  }
}
```

## Common Use Cases

### SPV Verification

Verify merkle roots against blockchain state for transaction validation.

### Block Height Monitoring

Track current blockchain height for application synchronization.

### Merkle Path Validation

Validate merkle paths received in BEEF transactions.

### Multi-Network Support

Switch between mainnet, testnet, and STN for development and production.

## Troubleshooting

### Common Issues

1. **Network Timeouts**: Implement retry logic with exponential backoff
2. **Rate Limiting**: Use API keys and respect WhatsOnChain rate limits
3. **Invalid Network Names**: Use 'main', 'test', or 'stn' (not 'mainnet', 'testnet')
4. **Method Not Found**: WhatsOnChain only supports `currentHeight()` and `isValidRootForHeight()`

### API Limitations

```typescript
// ❌ These methods DO NOT exist in WhatsOnChain:
// chainTracker.getTransaction(txid)
// chainTracker.getUTXOs(address)
// chainTracker.getMerkleProof(txid)
// chainTracker.getBlockHeader(blockHash)

// ✅ Only these methods are available:
const height = await chainTracker.currentHeight()
const isValid = await chainTracker.isValidRootForHeight(merkleRoot, blockHeight)
```

### Debugging Tips

```typescript
// Configure with API key for better rate limits
const chainTracker = new WhatsOnChain('main', {
  apiKey: 'your-api-key'
})

// Add error handling and logging
try {
  const height = await chainTracker.currentHeight()
  console.log('Current height:', height)
} catch (error) {
  console.error('Chain tracker error:', error.message)
  // Check network connectivity and API status
}
```

## Next Steps

- Learn about [Transaction Verification](../concepts/verification.md) for SPV implementation
- Explore [BEEF Format](../concepts/beef.md) for efficient data exchange
- Review [Security Best Practices](./security-best-practices.md) for production deployment
- Check out the [SPV and Merkle Proofs](../tutorials/spv-merkle-proofs.md) tutorial

## Conclusion

Chain tracking is essential for building Bitcoin applications that need blockchain data access. By following the patterns in this guide, you can build robust applications that efficiently interact with the Bitcoin SV network while maintaining security and performance.

While the `WalletClient` provides high-level blockchain interaction, understanding chain tracking enables you to build sophisticated applications that can monitor and verify blockchain state.
