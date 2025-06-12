# Chain Tracking

How the BSV TypeScript SDK interacts with the Bitcoin network to retrieve transaction and blockchain data.

## Chain Tracker Concept

A chain tracker provides access to Bitcoin blockchain data without running a full node:

```typescript
import { WhatsOnChain } from '@bsv/sdk'

// Create a chain tracker
const chainTracker = new WhatsOnChain('mainnet')

// Get transaction data
const txData = await chainTracker.getTransaction('txid')
```

## Key Functions

### Transaction Lookup
- Retrieve transaction details by ID
- Get transaction status and confirmations
- Access transaction inputs and outputs

### UTXO Queries
- Find unspent transaction outputs
- Check UTXO availability and value
- Retrieve UTXO locking scripts

### Block Information
- Get block headers and merkle proofs
- Verify transaction inclusion in blocks
- Access block timestamps and difficulty

### Network Status
- Check network connectivity
- Monitor chain tip and height
- Get fee rate recommendations

## SPV Integration

Chain trackers enable SPV (Simplified Payment Verification):
- **Merkle Proofs**: Verify transaction inclusion without full blocks
- **Header Chain**: Maintain block headers for proof verification
- **Lightweight**: Minimal data requirements compared to full nodes

## Multiple Providers

The SDK supports multiple chain tracking services:

```typescript
// Primary and fallback providers
const config = {
  chainTracker: {
    provider: 'WhatsOnChain',
    network: 'mainnet',
    fallbacks: ['GorillaPool', 'TAAL']
  }
}
```

## Benefits

### Scalability
- No need to store the entire blockchain
- Fast startup and synchronization
- Minimal storage requirements

### Reliability
- Multiple provider support
- Automatic failover capabilities
- Redundant data sources

### Performance
- Targeted data queries
- Caching of frequently accessed data
- Optimized for application needs

## Common Patterns

### Transaction Verification
```typescript
// Verify a transaction exists
const exists = await chainTracker.getTransaction(txid)
if (exists) {
  // Transaction is confirmed on-chain
}
```

### UTXO Validation
```typescript
// Check if UTXO is still unspent
const utxo = await chainTracker.getUTXO(txid, outputIndex)
if (utxo) {
  // UTXO is available for spending
}
```

## Error Handling

Chain tracker operations can fail due to:
- Network connectivity issues
- Service provider downtime
- Invalid transaction IDs
- Rate limiting

The SDK provides automatic retry and failover mechanisms.

## Configuration

Chain trackers can be configured for:
- **Network**: Mainnet, testnet, or regtest
- **Endpoints**: Custom service URLs
- **Timeouts**: Request timeout settings
- **Retry Logic**: Failure handling behavior

## Next Steps

- Understand [SPV Verification](./spv-verification.md) concepts
- Learn about [BEEF Format](./beef.md) for efficient data exchange
- Explore [Trust Model](./trust-model.md) considerations
