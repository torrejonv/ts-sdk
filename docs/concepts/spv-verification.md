# SPV Verification

Understanding Simplified Payment Verification and how it enables lightweight Bitcoin applications.

## What is SPV?

SPV allows verification of Bitcoin transactions without downloading the entire blockchain:
- **Lightweight**: Only requires block headers and merkle proofs
- **Secure**: Cryptographically verifiable using merkle trees
- **Efficient**: Scales to millions of transactions
- **Practical**: Enables mobile and web applications

## How SPV Works

### 1. Block Headers
Download only block headers (80 bytes each) instead of full blocks:
```typescript
// Block header contains merkle root
const header = await chainTracker.getBlockHeader(blockHash)
```

### 2. Merkle Proofs
Verify transaction inclusion using merkle proofs:
```typescript
import { MerklePath } from '@bsv/sdk'

// Verify transaction is in block
const proof = MerklePath.fromHex(proofHex)
const isValid = proof.verify(txid, merkleRoot)
```

### 3. Transaction Verification
Combine proofs with block headers for full verification:
```typescript
import { Transaction } from '@bsv/sdk'

// Verify transaction with SPV
const isValid = await Transaction.verify(
  transaction,
  chainTracker,
  { merkleProof: proof }
)
```

## Merkle Trees

Bitcoin uses merkle trees to efficiently prove transaction inclusion:
- **Binary Tree**: Each leaf is a transaction ID
- **Hash Pairs**: Parent nodes are hashes of child pairs
- **Root Hash**: Single hash representing all transactions
- **Proof Path**: Minimal data needed to verify inclusion

## Security Model

SPV provides strong security guarantees:
- **Proof of Work**: Block headers contain proof of work
- **Cryptographic Hashes**: Merkle proofs use SHA-256
- **Chain Validation**: Verify header chain integrity
- **Fraud Detection**: Invalid proofs are cryptographically detectable

## Trade-offs

### Advantages
- **Low Resource Usage**: Minimal storage and bandwidth
- **Fast Synchronization**: Quick startup time
- **Scalability**: Works with any blockchain size
- **Privacy**: No need to reveal which transactions you care about

### Limitations
- **Trust Assumptions**: Relies on honest majority of miners
- **Network Dependency**: Requires connection to full nodes
- **Delayed Detection**: May not immediately detect invalid blocks

## SDK Implementation

The SDK provides comprehensive SPV support:

```typescript
// Configure SPV verification
const config = {
  spv: {
    enabled: true,
    maxMemoryLimit: 100000000, // 100MB
    chainTracker: chainTracker
  }
}

// Verify transaction with SPV
const result = await transaction.verify(chainTracker, {
  merkleProof: proof,
  blockHeader: header
})
```

## BEEF Integration

SPV works seamlessly with BEEF format:
- **Efficient Encoding**: BEEF includes merkle proofs
- **Batch Verification**: Verify multiple transactions together
- **Standardized Format**: Consistent across applications

## Next Steps

- Learn about [BEEF Format](./beef.md) for efficient data exchange
- Understand [Transaction Encoding](./transaction-encoding.md) formats
- Explore [Trust Model](./trust-model.md) considerations
