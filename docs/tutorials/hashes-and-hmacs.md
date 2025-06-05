# Using Hashes and HMACs

--- BASE THIS ON https://docs.bsvblockchain.org/guides/sdks/ts/low-level/using_hashes_and_hmacs -- 

--- IMPROVE WITH Code examples

**Duration**: 60 minutes
**Prerequisites**: Basic TypeScript knowledge, First Transaction tutorial completed

## Learning Goals
- Understand cryptographic hashing functions and their properties
- Implement various hash algorithms using the BSV TypeScript SDK
- Create and verify HMACs (Hash-based Message Authentication Codes)
- Apply hashing techniques in practical Bitcoin scenarios

## Introduction to Cryptographic Hashing

Cryptographic hash functions are fundamental to Bitcoin's security. This tutorial explores how to implement and use various hash functions and HMACs using the BSV TypeScript SDK.

## Setting Up Your Environment

```typescript
import { Hash, Hmac } from '@bsv/sdk'
```

## Common Hash Functions

### SHA-256

SHA-256 is the primary hash function used in Bitcoin:

```typescript
// Creating a SHA-256 hash
const message = 'Hello, Bitcoin!'
const sha256Hash = Hash.sha256(Buffer.from(message))

// Remember to use toHex() instead of toString() per API requirements
console.log('SHA-256 hash:', sha256Hash.toHex())
```

### Double SHA-256

Bitcoin often uses a double SHA-256 hash:

```typescript
// Double SHA-256 hash
const doubleSha256Hash = Hash.sha256d(Buffer.from(message))
console.log('Double SHA-256 hash:', doubleSha256Hash.toHex())
```

### RIPEMD-160

RIPEMD-160 is used in Bitcoin addresses:

```typescript
// RIPEMD-160 hash
const ripemd160Hash = Hash.ripemd160(Buffer.from(message))
console.log('RIPEMD-160 hash:', ripemd160Hash.toHex())
```

### Hash160 (SHA-256 + RIPEMD-160)

Hash160 is used for public key hashing in Bitcoin addresses:

```typescript
// Hash160 (SHA-256 then RIPEMD-160)
const hash160 = Hash.hash160(Buffer.from(message))
console.log('Hash160:', hash160.toHex())
```

## Hash-based Message Authentication Codes (HMACs)

HMACs provide both integrity and authenticity verification:

```typescript
// Creating an HMAC using SHA-256
const key = Buffer.from('my-secret-key')
const hmacSha256 = Hmac.sha256(key, Buffer.from(message))
console.log('HMAC-SHA-256:', hmacSha256.toHex())
```

## Practical Applications

### Data Integrity Verification

```typescript
// Example of verifying data integrity using hashes
const verifyDataIntegrity = (data, expectedHash) => {
  const actualHash = Hash.sha256(Buffer.from(data)).toHex()
  return actualHash === expectedHash
}
```

### Transaction ID Calculation

```typescript
// Example of calculating a transaction ID
const calculateTxId = (tx) => {
  // Transaction ID is the reversed double SHA-256 hash of the serialized tx
  const txHash = Hash.sha256d(tx.toBuffer())
  // Must convert from byte array per API recommendations
  return Buffer.from(txHash).reverse().toString('hex')
}
```

### Secure Authentication

```typescript
// Example of an HMAC-based authentication system
```

## Advanced Hashing Techniques

### Custom Hash Chains

```typescript
// Example of creating custom hash chains
```

### Streaming Large Data

```typescript
// Hashing large data efficiently
```

## Security Considerations

- Hash collision resistance
- Timing attack prevention
- Key management for HMACs

## Conclusion

In this tutorial, you've learned how to implement and use cryptographic hash functions and HMACs using the BSV TypeScript SDK. You now understand how to generate various types of hashes, create and verify HMACs, and apply these techniques in practical Bitcoin applications.

## Further Reading

- [NIST Hash Function Standards]()
- [HMAC Specification]()
