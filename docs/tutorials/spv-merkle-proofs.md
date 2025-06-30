# SPV and Merkle Proof Verification

## Introduction

Simplified Payment Verification (SPV) is a method for verifying Bitcoin transactions without downloading the entire blockchain. Instead of storing all transaction data, SPV clients only need block headers and merkle proofs to verify that specific transactions are included in the blockchain.

This tutorial covers:
- Understanding SPV principles and merkle trees
- Working with merkle proofs using the `MerklePath` class
- Verifying transactions with the `Transaction.verify()` method
- Implementing custom chain trackers for block header verification
- Working with BEEF (BRC-62) structures for efficient SPV

> **📚 Related Concepts**: Review [SPV Verification](../concepts/spv-verification.md), [Transaction Verification](../concepts/verification.md), and [BEEF Format](../concepts/beef.md) for foundational understanding.

## Prerequisites

- Completed "Your First BSV Transaction" tutorial
- Basic understanding of Bitcoin transaction structure
- Familiarity with cryptographic hash functions

## Understanding SPV and Merkle Trees

### What is SPV?

SPV allows lightweight clients to verify transactions without storing the full blockchain by:

1. **Block Headers Only**: Store only block headers (80 bytes each) instead of full blocks
2. **Merkle Proofs**: Use cryptographic proofs to verify transaction inclusion
3. **Chain Validation**: Verify the proof-of-work chain of block headers
4. **Script Validation**: Validate that transaction scripts are properly formed

### Merkle Trees in Bitcoin

Bitcoin blocks organize transactions in a binary merkle tree structure:

```
        Merkle Root
       /           \
    Hash AB       Hash CD
   /      \      /      \
Hash A  Hash B Hash C  Hash D
  |       |      |       |
 Tx A    Tx B   Tx C    Tx D
```

A merkle proof provides the minimum hashes needed to compute the merkle root from a specific transaction.

## Working with MerklePath

The `MerklePath` class represents a merkle proof for a specific transaction:

```typescript
import { MerklePath, WhatsOnChain } from '@bsv/sdk'

async function runMerkleExample() {
  // Create the merkle path first (our demonstration example)
  const blockHeight = 850000
  const merklePath = new MerklePath(blockHeight, [
    [
      { offset: 0, hash: 'ffeff11c25cde7c06d407490d81ef4d0db64aad6ab3d14393530701561a465ef', txid: true },
      { offset: 1, hash: 'b9ef07a62553ef8b0898a79c291b92c60f7932260888bde0dab2dd2610d8668e' }
    ]
  ])

  // Example tx
  const txid = 'ffeff11c25cde7c06d407490d81ef4d0db64aad6ab3d14393530701561a465ef'

  // Create a chain tracker for mainnet
  const chainTracker = new WhatsOnChain('main')

  // Verify the merkle proof
  const isValid = await merklePath.verify(txid, chainTracker)
  console.log('Merkle proof valid:', isValid)
  
  // Note: This will return false because our example merkle path
  // doesn't correspond to a real block on the BSV mainnet
}

runMerkleExample().catch(console.error)
```

## Working with Real Blockchain Data

The example above demonstrates the fundamental concepts using a simplified 2-transaction block. In real BSV blockchain scenarios, blocks contain hundreds or thousands of transactions, creating much deeper merkle trees.

**Our Working Example:**
- **Block Height**: 850000 (arbitrary example height)
- **Transaction ID**: `ffeff11c25cde7c06d407490d81ef4d0db64aad6ab3d14393530701561a465ef` (from BSV Technical Standards)
- **Sibling Hash**: `b9ef07a62553ef8b0898a79c291b92c60f7932260888bde0dab2dd2610d8668e` (from BSV Technical Standards)
- **Computed Merkle Root**: `6f0a2a566d54512576b3b32eb3a8ca5273d8f35d8bfba02123bb7aad59be1e61`

**Real-World Complexity:**
In actual BSV blocks, a transaction at index 12 (like in the BSV Technical Standards example) would require a merkle path with multiple levels:
- **5 proof levels** for a block with ~32 transactions
- **10 proof levels** for a block with ~1024 transactions  
- **20 proof levels** for a block with ~1 million transactions

The logarithmic nature of merkle trees means even massive blocks require relatively few proof hashes for verification.

## Real Blockchain Merkle Paths

In practice, merkle paths from real BSV blocks are more complex, with multiple levels representing the tree structure. The example above shows the simplest case - a block with only 2 transactions.

**Real BSV Blockchain Example:**
Based on data from the [BSV Technical Standards](https://tsc.bsvblockchain.org/standards/merkle-proof-standardised-format/), a transaction at index 12 in a larger block would have a merkle path with these hash values:

- **Transaction**: `ffeff11c25cde7c06d407490d81ef4d0db64aad6ab3d14393530701561a465ef`
- **Proof nodes**: 5 hash values that form the path to the merkle root
- **Merkle root**: `75edb0a69eb195cdd81e310553aa4d25e18450e08f168532a2c2e9cf447bf169`

The SDK handles the complex offset calculations automatically when parsing from binary formats or BEEF structures.

## Serialization Formats

The BSV TypeScript SDK uses an internal object format for MerklePath construction, while the [BSV Technical Standards](https://tsc.bsvblockchain.org/standards/merkle-proof-standardised-format/) define a binary serialization format. The SDK handles the conversion between these formats internally.

**Internal Format (used above):**
- Array of levels, each containing leaf objects with `offset`, `hash`, `txid`, and `duplicate` properties
- Direct construction allows for clear understanding of the merkle tree structure

**Binary Format (from standards):**
- Compact binary representation for network transmission and storage
- Can be parsed using `MerklePath.fromHex()` when properly formatted

## Computing Merkle Roots

You can compute the merkle root for a given transaction ID:

```typescript
// Compute merkle root for a specific transaction
const txid = 'ffeff11c25cde7c06d407490d81ef4d0db64aad6ab3d14393530701561a465ef'
const merkleRoot = merklePath.computeRoot(txid)

console.log('Computed merkle root:', merkleRoot)

// How the merkle root is computed for our 2-transaction block:
// 1. Take our transaction: ffeff11c25cde7c06d407490d81ef4d0db64aad6ab3d14393530701561a465ef
// 2. Take sibling transaction: b9ef07a62553ef8b0898a79c291b92c60f7932260888bde0dab2dd2610d8668e
// 3. Hash them together: SHA256(SHA256(txid + sibling))
// 4. Result is the merkle root: 6f0a2a566d54512576b3b32eb3a8ca5273d8f35d8bfba02123bb7aad59be1e61

console.log('Expected result: 6f0a2a566d54512576b3b32eb3a8ca5273d8f35d8bfba02123bb7aad59be1e61')
console.log('Merkle root matches expected:', merkleRoot === '6f0a2a566d54512576b3b32eb3a8ca5273d8f35d8bfba02123bb7aad59be1e61')
```

## Understanding Merkle Tree Computation

The merkle root computation follows a specific mathematical process:

**For a 2-transaction block (our example):**
```
Level 0 (Leaves):    [Transaction A]  [Transaction B]
                           |              |
Level 1 (Root):           [Hash(A + B)]
```

**Step-by-step process:**
1. **Start with transaction IDs** (already hashed)
2. **Concatenate them**: `ffeff11c...465ef` + `b9ef07a6...8668e`
3. **Double SHA256**: `SHA256(SHA256(concatenated_data))`
4. **Result**: The merkle root that represents the entire block

**For larger blocks (e.g., 4 transactions):**
```
Level 0:    [Tx A]  [Tx B]  [Tx C]  [Tx D]
              |       |       |       |
Level 1:    [Hash(A+B)]     [Hash(C+D)]
                 |               |
Level 2:         [Hash(AB + CD)]
```

This tree structure allows you to prove any transaction's inclusion with only `log₂(n)` hashes, making verification extremely efficient even for blocks with millions of transactions.

## Verifying Merkle Proofs

Verify that a transaction is included in a block using a chain tracker:

```typescript
import { MerklePath, WhatsOnChain } from '@bsv/sdk'

async function runMerkleVerificationExample() {
  // Create a simple merkle path for demonstration
  // This is a 2-transaction block example from BSV Technical Standards
  const blockHeight = 850000
  const merklePath = new MerklePath(blockHeight, [
    [
      { offset: 0, hash: 'ffeff11c25cde7c06d407490d81ef4d0db64aad6ab3d14393530701561a465ef', txid: true },
      { offset: 1, hash: 'b9ef07a62553ef8b0898a79c291b92c60f7932260888bde0dab2dd2610d8668e' }
    ]
  ])

  // Example transaction ID
  const txid = 'ffeff11c25cde7c06d407490d81ef4d0db64aad6ab3d14393530701561a465ef'

  // Create a chain tracker for mainnet
  const chainTracker = new WhatsOnChain('main')

  try {
    // Compute the merkle root (this works with our demonstration data)
    const merkleRoot = merklePath.computeRoot(txid)
    console.log('Computed merkle root:', merkleRoot)
    
    // Verify the merkle proof (this will return false for our demo data)
    const isValid = await merklePath.verify(txid, chainTracker)
    console.log('Merkle proof valid:', isValid)
    
    // Note: This returns false because our example uses demonstration data
    // rather than real blockchain merkle proof data
    if (!isValid) {
      console.log('ℹ️  This is expected - our example uses synthetic data for learning purposes')
      console.log('   Real applications receive merkle paths from BEEF structures or blockchain services')
    }
  } catch (error) {
    console.error('Error verifying merkle proof:', error)
  }
}

// Run the example
runMerkleVerificationExample().catch(console.error)
```

## Chain Trackers

Chain trackers verify that merkle roots are valid for specific block heights. The SDK provides the `WhatsOnChain` implementation:

### Using WhatsOnChain Chain Tracker

```typescript
import { WhatsOnChain } from '@bsv/sdk'

async function runChainTrackerExample() {
  // Mainnet chain tracker
  const mainnetTracker = new WhatsOnChain('main')

  // Testnet chain tracker  
  const testnetTracker = new WhatsOnChain('test', {
    apiKey: 'your-api-key' // Optional for higher rate limits
  })

  try {
    // Check current blockchain height
    const currentHeight = await mainnetTracker.currentHeight()
    console.log('Current block height:', currentHeight)

    // Verify a merkle root for a specific height
    const isValidRoot = await mainnetTracker.isValidRootForHeight(
      'merkle-root-hex',
      850000
    )
    console.log('Valid merkle root:', isValidRoot)
    console.log('NOTE - this is expected to be false, as our example uses demonstration data')
  } catch (error) {
    console.error('Chain tracker error:', error)
  }
}

// Run the example
runChainTrackerExample().catch(console.error)
```

## Transaction Verification with SPV

The `Transaction.verify()` method performs complete SPV verification:

### Basic Transaction Verification

```typescript
import { Transaction, WhatsOnChain, SatoshisPerKilobyte } from '@bsv/sdk'

async function runTransactionVerificationExample() {
  // Create transaction from BEEF data
  const beefHex = 'your-beef-hex-data'
  const transaction = Transaction.fromHexBEEF(beefHex)

  // Set up chain tracker and fee model
  const chainTracker = new WhatsOnChain('main')
  const feeModel = new SatoshisPerKilobyte(1)

  // Verify the transaction
  try {
    const isValid = await transaction.verify(chainTracker, feeModel)
    console.log('Transaction valid:', isValid)
    console.log('NOTE - this is expected to be false, as our example uses demonstration data')
  } catch (error) {
    console.error('Verification failed:', error.message)
  }
}

// Run the example
runTransactionVerificationExample().catch(console.error)
```

### Scripts-Only Verification

```typescript
async function runScriptsOnlyVerificationExample() {
  // Assuming you have a transaction from previous example
  const beefHex = 'your-beef-hex-data'
  const transaction = Transaction.fromHexBEEF(beefHex)
  
  try {
    // Verify only scripts without checking block headers
    const isScriptValid = await transaction.verify('scripts only')
    console.log('Scripts valid:', isScriptValid)
  } catch (error) {
    console.error('Script verification failed:', error.message)
  }
}

// Run the example
runScriptsOnlyVerificationExample().catch(console.error)
```

### Working with BEEF Structures

BEEF (BRC-62) provides an efficient format for SPV data:

### Creating BEEF from Transaction

```typescript
// Create a transaction with inputs and merkle proofs
const tx = new Transaction()
// ... add inputs and outputs ...

// Convert to BEEF format
const beefData = tx.toBEEF()
const beefHex = Buffer.from(beefData).toString('hex')

console.log('BEEF hex:', beefHex)
```

### Parsing BEEF Data

```typescript
// Parse BEEF structure
const transaction = Transaction.fromHexBEEF(beefHex)

console.log('Transaction ID:', Buffer.from(transaction.id()).toString('hex'))
console.log('Input count:', transaction.inputs.length)
console.log('Output count:', transaction.outputs.length)

// Check if merkle paths are included
transaction.inputs.forEach((input, index) => {
  if (input.sourceTransaction?.merklePath) {
    console.log(`Input ${index} has merkle proof at height:`, 
      input.sourceTransaction.merklePath.blockHeight)
  }
})
```

## Practical Example: Payment Verification

Let's create a complete example that verifies a payment transaction:

```typescript
import { 
  Transaction, 
  WhatsOnChain, 
  SatoshisPerKilobyte,
  PrivateKey,
  P2PKH 
} from '@bsv/sdk'

async function verifyPayment(beefHex: string): Promise<boolean> {
  try {
    // Parse the BEEF transaction
    const transaction = Transaction.fromHexBEEF(beefHex)
    
    // Set up verification components
    const chainTracker = new WhatsOnChain('main')
    
    // Perform SPV verification
    const isValid = await transaction.verify(chainTracker)
    
    if (isValid) {
      console.log('✅ Payment verified successfully!')
      
      // Extract payment details
      const txid = Buffer.from(transaction.id()).toString('hex')
      console.log('Transaction ID:', txid)
      
      // Check outputs for payment amounts
      transaction.outputs.forEach((output, index) => {
        console.log(`Output ${index}: ${output.satoshis} satoshis`)
        
        // Check if it's a P2PKH output (OP_DUP OP_HASH160 <20-byte-hash> OP_EQUALVERIFY OP_CHECKSIG)
        try {
          const script = output.lockingScript
          const chunks = script.chunks
          if (chunks.length === 5 && 
              chunks[0].op === 118 && // OP_DUP
              chunks[1].op === 169 && // OP_HASH160
              chunks[2].data && chunks[2].data.length === 20 &&
              chunks[3].op === 136 && // OP_EQUALVERIFY
              chunks[4].op === 172) { // OP_CHECKSIG
            const pubKeyHash = Buffer.from(chunks[2].data).toString('hex')
            console.log(`  → P2PKH address hash: ${pubKeyHash}`)
          } else {
            console.log(`  → Custom script output`)
          }
        } catch {
          console.log(`  → Custom script output`)
        }
      })
      
      return true
    } else {
      console.log('❌ Payment verification failed')
      return false
    }
  } catch (error) {
    console.error('Verification error:', (error as Error).message)
    return false
  }
}

// Example usage
const exampleBEEF = '0100beef01fe636d0c0007021400fe507c0c7aa754cef1f7889d5fd395cf1f785dd7de98eed895dbedfe4e5bc70d1502ac4e164f5bc16746bb0868404292ac8318bbac3800e4aad13a014da427adce3e010b00bc4ff395efd11719b277694cface5aa50d085a0bb81f613f70313acd28cf4557010400574b2d9142b8d28b61d88e3b2c3f44d858411356b49a28a4643b6d1a6a092a5201030051a05fc84d531b5d250c23f4f886f6812f9fe3f402d61607f977b4ecd2701c19010000fd781529d58fc2523cf396a7f25440b409857e7e221766c57214b1d38c7b481f01010062f542f45ea3660f86c013ced80534cb5fd4c19d66c56e7e8c5d4bf2d40acc5e010100b121e91836fd7cd5102b654e9f72f3cf6fdbfd0b161c53a9c54b12c841126331020100000001cd4e4cac3c7b56920d1e7655e7e260d31f29d9a388d04910f1bbd72304a79029010000006b483045022100e75279a205a547c445719420aa3138bf14743e3f42618e5f86a19bde14bb95f7022064777d34776b05d816daf1699493fcdf2ef5a5ab1ad710d9c97bfb5b8f7cef3641210263e2dee22b1ddc5e11f6fab8bcd2378bdd19580d640501ea956ec0e786f93e76ffffffff013e660000000000001976a9146bfd5c7fbe21529d45803dbcf0c87dd3c71efbc288ac0000000001000100000001ac4e164f5bc16746bb0868404292ac8318bbac3800e4aad13a014da427adce3e000000006a47304402203a61a2e931612b4bda08d541cfb980885173b8dcf64a3471238ae7abcd368d6402204cbf24f04b9aa2256d8901f0ed97866603d2be8324c2bfb7a37bf8fc90edd5b441210263e2dee22b1ddc5e11f6fab8bcd2378bdd19580d640501ea956ec0e786f93e76ffffffff013c660000000000001976a9146bfd5c7fbe21529d45803dbcf0c87dd3c71efbc288ac0000000000'

async function runPaymentVerificationExample() {
  console.log('=== Payment Verification Example ===\n')
  
  // Verify the payment
  const isValid = await verifyPayment(exampleBEEF)
  
  if (isValid) {
    console.log('\n✅ Payment successfully verified using SPV!')
    console.log('This transaction can be trusted without downloading the full blockchain.')
  } else {
    console.log('\n❌ Payment verification failed!')
    console.log('This transaction should not be trusted.')
  }
  
  console.log('\n=== Payment Processing Workflow ===')
  console.log('1. Customer sends BEEF-encoded transaction')
  console.log('2. Merchant verifies transaction using SPV')
  console.log('3. If valid, merchant can safely accept payment')
  console.log('4. No need to wait for confirmations or run full node')
}

runPaymentVerificationExample().catch(console.error)
```

## Advanced SPV Patterns

### Batch Verification

Verify multiple transactions efficiently:

```typescript
async function verifyMultipleTransactions(beefHexArray: string[]): Promise<boolean[]> {
  const chainTracker = new WhatsOnChain('main')
  const feeModel = new SatoshisPerKilobyte(1)
  
  const results = await Promise.all(
    beefHexArray.map(async (beefHex) => {
      try {
        const tx = Transaction.fromHexBEEF(beefHex)
        return await tx.verify(chainTracker, feeModel)
      } catch (error) {
        console.error('Verification failed:', error.message)
        return false
      }
    })
  )
  
  return results
}
```

### Merkle Proof Validation

Manually validate merkle proofs:

```typescript
function validateMerkleProof(
  txid: string, 
  merklePath: MerklePath, 
  expectedRoot: string
): boolean {
  try {
    const computedRoot = merklePath.computeRoot(txid)
    return computedRoot === expectedRoot
  } catch (error) {
    console.error('Error computing merkle root:', error.message)
    return false
  }
}

// Example usage
const isValidProof = validateMerkleProof(
  'transaction-id',
  merklePath,
  'expected-merkle-root'
)
console.log('Merkle proof valid:', isValidProof)
```

### Custom Memory Limits

Control script execution memory usage:

```typescript
// Verify with custom memory limit (in bytes)
const isValid = await transaction.verify(
  chainTracker,
  feeModel,
  1024 * 1024 // 1MB memory limit
)
```

## Error Handling and Debugging

### Common Verification Errors

```typescript
async function robustVerification(beefHex: string): Promise<void> {
  try {
    const transaction = Transaction.fromHexBEEF(beefHex)
    
    // Set up verification components
    const chainTracker = new WhatsOnChain('main')
    
    console.log('🔍 Starting transaction verification...')
    console.log('Transaction ID:', Buffer.from(transaction.id()).toString('hex'))
    
    // Perform SPV verification
    const isValid = await transaction.verify(chainTracker)
    
    if (!isValid) {
      console.log('❌ Transaction verification failed. Checking components...')
      
      // Check individual merkle proofs
      for (let i = 0; i < transaction.inputs.length; i++) {
        const input = transaction.inputs[i]
        if (input.sourceTransaction?.merklePath) {
          try {
            const sourceTxid = Buffer.from(input.sourceTransaction.id()).toString('hex')
            const proofValid = await input.sourceTransaction.merklePath.verify(
              sourceTxid, 
              chainTracker
            )
            console.log(`  Input ${i} merkle proof: ${proofValid ? '✅' : '❌'}`)
          } catch (err) {
            console.log(`  Input ${i} merkle proof: ❌ (${(err as Error).message})`)
          }
        } else {
          console.log(`  Input ${i}: No merkle path provided`)
        }
      }
      
      // Try scripts-only verification
      try {
        const scriptsValid = await transaction.verify('scripts only')
        console.log('  Scripts validation:', scriptsValid ? '✅' : '❌')
      } catch (err) {
        console.log('  Scripts validation: ❌ (', (err as Error).message, ')')
      }
    } else {
      console.log('✅ Transaction verification successful!')
    }
    
  } catch (error) {
    const errorMessage = (error as Error).message
    
    console.log('❌ Verification failed with error:')
    
    if (errorMessage.includes('Missing source transaction')) {
      console.error('  → BEEF structure incomplete - missing input transactions')
      console.error('  → Solution: Ensure all input transactions are included in BEEF')
    } else if (errorMessage.includes('Merkle root')) {
      console.error('  → Merkle proof verification failed')
      console.error('  → Solution: Check merkle path data and chain tracker connectivity')
    } else if (errorMessage.includes('script')) {
      console.error('  → Script validation failed')
      console.error('  → Solution: Check unlocking scripts and signature validity')
    } else if (errorMessage.includes('BEEF')) {
      console.error('  → BEEF parsing error')
      console.error('  → Solution: Verify BEEF format and encoding')
    } else {
      console.error('  → Unexpected error:', errorMessage)
    }
  }
}
```

### Network-Specific Verification

```typescript
async function verifyOnNetwork(beefHex: string, network: 'main' | 'test'): Promise<boolean> {
  const chainTracker = new WhatsOnChain(network)
  const transaction = Transaction.fromHexBEEF(beefHex)
  
  console.log(`Verifying on ${network}net...`)
  return await transaction.verify(chainTracker)
}
```

## Best Practices

### 1. Chain Tracker Selection

```typescript
// Production: Use WhatsOnChain with API key
const productionTracker = new WhatsOnChain('main', {
  apiKey: process.env.WHATSONCHAIN_API_KEY
})

// Development: Use testnet
const devTracker = new WhatsOnChain('test')

// Testing: Use mock tracker
const testTracker: ChainTracker = {
  async isValidRootForHeight() { return true },
  async currentHeight() { return 850000 }
}
```

### 2. Error Recovery

```typescript
async function verifyWithRetry(beefHex: string, maxRetries = 3): Promise<boolean> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const transaction = Transaction.fromHexBEEF(beefHex)
      const chainTracker = new WhatsOnChain('main')
      
      return await transaction.verify(chainTracker)
    } catch (error) {
      console.log(`Attempt ${attempt} failed:`, (error as Error).message)
      
      if (attempt === maxRetries) {
        throw error
      }
      
      // Wait before retry
      await new Promise(resolve => setTimeout(resolve, 1000 * attempt))
    }
  }
  
  return false
}
```

## Summary

SPV and merkle proof verification enable lightweight Bitcoin clients to verify transactions without storing the full blockchain. The BSV TypeScript SDK provides comprehensive tools for:

- **MerklePath**: Computing and verifying merkle proofs
- **ChainTracker**: Validating merkle roots against block headers  
- **Transaction.verify()**: Complete SPV verification
- **BEEF**: Efficient SPV data structures

Key takeaways:
- SPV trades storage for bandwidth and computation
- Merkle proofs provide cryptographic inclusion proofs
- Chain trackers ensure merkle roots are valid
- BEEF structures optimize SPV data transmission
- Always use trusted chain trackers in production
- Implement proper error handling and retry logic

This foundation enables building lightweight Bitcoin applications that can verify payments and transactions without running a full node.

Understanding of `WalletClient` usage is also important for building robust applications. `WalletClient` provides high-level transaction verification, but understanding SPV verification gives you the ability to build lightweight applications that can verify transactions without downloading the entire blockchain.

## SPV vs Full Node vs `WalletClient` Verification

| Method | `WalletClient` | SPV | Full Node |
| --- | --- | --- | --- |
| **Storage** | High | Low | High |
| **Bandwidth** | Low | High | Low |
| **Verification** | High-level | Low-level | High-level |
| **Security** | High | High | High |

The `WalletClient` approach is recommended for most applications, while SPV verification is valuable for specialized lightweight applications.

```typescript
// Example usage
const walletClient = new WalletClient('main')
const isValid = await walletClient.verifyTransaction(beefHex)
console.log('Transaction valid:', isValid)
