# Transaction Signatures


--- BASE THIS ON https://docs.bsvblockchain.org/guides/sdks/concepts -- 

--- IMPROVE WITH Code examples

This document explains the concept of digital signatures in Bitcoin transactions and their implementation in the BSV TypeScript SDK.

## Understanding Bitcoin Signatures

- Digital signature fundamentals
- ECDSA and the secp256k1 curve
- Signature serialization formats

## Signature Components

### Private and Public Keys

- Key pair generation
- Key formats (WIF, raw bytes, hex)
- Conversion between formats using `toWif()` (not `toWIF()`)

### Signature Hash (SIGHASH) Types

- SIGHASH_ALL
- SIGHASH_NONE
- SIGHASH_SINGLE
- SIGHASH_ANYONECANPAY combinations
- Use cases for different SIGHASH types

## Implementation in the SDK

- Creating and verifying signatures
- Working with multi-signature scenarios
- Signature serialization using `toHex()` rather than `toString()`

## Advanced Signature Topics

- Low-S value enforcement
- Signature malleability considerations
- Future signature schemes

## Implementation Examples

```typescript
// Example of signing a transaction
// Note: Implementation examples will use proper method calls rather than method chaining
// Code examples will be added here
```

## Best Practices

- Secure key management
- Testing signature validity
- Performance considerations

## References

- [ECDSA Specification]()
- [Bitcoin Signature Formats]()
