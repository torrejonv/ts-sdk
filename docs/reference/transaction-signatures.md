# Transaction Signatures Reference
** SKELETON PLACEHOLDER - check  https://docs.bsvblockchain.org/guides/sdks/ts/low-level/tx_sig ***
*This is a technical reference document for transaction signatures in the BSV TypeScript SDK.*

## Introduction

This reference document provides detailed technical information about transaction signatures in Bitcoin SV and how they are implemented in the TypeScript SDK. It complements the [Key Management and Cryptography tutorial](../tutorials/key-management.md) and the [Advanced Transaction Signing guide](../guides/advanced-transaction-signing.md).

## Signature Components

### ECDSA Signatures

Bitcoin uses the Elliptic Curve Digital Signature Algorithm (ECDSA) with the secp256k1 curve. Each signature consists of two components:

- `r`: A value derived from a random nonce used during signing
- `s`: A value that combines the private key, message hash, and the nonce

```typescript
// Structure of a Signature class instance
class Signature {
  r: BigNumber; // The r component
  s: BigNumber; // The s component
  
  // Methods for serialization
  toDER(encoding?: 'hex'): number[] | string;
  
  // Static methods for deserialization
  static fromDER(bytes: number[]): Signature;
  // Additional methods...
}
```

### DER Encoding

Signatures in Bitcoin are encoded using the Distinguished Encoding Rules (DER) format, which provides a standardized way to represent the ECDSA signature components.

## Signature Hash (SIGHASH) Types

Bitcoin transactions use SIGHASH flags to indicate which parts of a transaction are included in the signature hash:

| Flag | Hex Value | Description |
|------|-----------|-------------|
| SIGHASH_ALL | 0x01 | Signs all inputs and outputs (default) |
| SIGHASH_NONE | 0x02 | Signs inputs only, allows any outputs |
| SIGHASH_SINGLE | 0x03 | Signs inputs and the output with the same index |
| SIGHASH_ANYONECANPAY | 0x80 | Can be combined with above flags, allows additional inputs |

## Transaction Signature Verification Process

1. Extract signature and public key from the unlockingScript
2. Determine the SIGHASH flag from the signature
3. Recreate the signature hash based on the SIGHASH flag
4. Verify the signature against the hash using the public key

## API Reference

### PrivateKey Methods

```typescript
// Sign a message or transaction with a private key
sign(message: string | number[], sigHashType?: number): Promise<Signature>;
```

### PublicKey Methods

```typescript
// Verify a signature with a public key
verify(message: string | number[], signature: Signature): boolean;
```

### Transaction Methods

```typescript
// Sign a transaction
sign(sigHashType?: number): Promise<void>;

// Verify all signatures in a transaction
verify(): Promise<boolean>;
```

## Example Implementation Details

```typescript
// Example of how signature hashing is implemented
function calculateSignatureHash(tx: Transaction, inputIndex: number, sigHashType: number): number[] {
  // Implementation details...
}
```

## Related Resources

- [Advanced Transaction Signing guide](../guides/advanced-transaction-signing.md)
- [Key Management and Cryptography tutorial](../tutorials/key-management.md)
- [Bitcoin Transaction Format](https://reference.cash/protocol/blockchain/transaction)
