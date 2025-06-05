# ECDH Key Exchange

--- BASE THIS ON https://docs.bsvblockchain.org/guides/sdks/ts/low-level/ecdh -- 

--- IMPROVE WITH Code examples

**Duration**: 75 minutes
**Prerequisites**: Basic TypeScript knowledge, Key Management tutorial completed

## Learning Goals
- Understand Elliptic Curve Diffie-Hellman (ECDH) key exchange principles
- Implement secure key exchange using the BSV TypeScript SDK
- Create shared secrets for encrypted communication
- Apply ECDH in practical Bitcoin applications

## Introduction to ECDH

Elliptic Curve Diffie-Hellman (ECDH) is a key agreement protocol that allows two parties to establish a shared secret over an unsecured communication channel. This tutorial explores how to implement ECDH using the BSV TypeScript SDK.

## Setting Up Your Environment

```typescript
import { PrivateKey, PublicKey, Ecdh } from '@bsv/sdk'
```

## Key Generation for ECDH

### Creating Private and Public Keys

```typescript
// Generate Alice's key pair
const alicePrivKey = PrivateKey.fromRandom()
// Remember to use toWif() not toWIF() per SDK requirements
const alicePrivKeyWif = alicePrivKey.toWif()
const alicePubKey = alicePrivKey.toPublicKey()

// Generate Bob's key pair
const bobPrivKey = PrivateKey.fromRandom()
const bobPrivKeyWif = bobPrivKey.toWif()
const bobPubKey = bobPrivKey.toPublicKey()
```

## Implementing ECDH Key Exchange

### Creating a Shared Secret

```typescript
// Alice creates a shared secret using Bob's public key and her private key
const aliceSharedSecret = Ecdh.deriveSharedSecret(
  alicePrivKey,
  bobPubKey
)

// Bob creates the same shared secret using Alice's public key and his private key
const bobSharedSecret = Ecdh.deriveSharedSecret(
  bobPrivKey,
  alicePubKey
)

// aliceSharedSecret and bobSharedSecret are identical
// Use toHex() instead of toString() per SDK requirements
console.log('Alice\'s shared secret:', aliceSharedSecret.toHex())
console.log('Bob\'s shared secret:', bobSharedSecret.toHex())
```

## Verifying the Shared Secret

```typescript
// Verification code to ensure both secrets match
```

## Practical Applications

### Secure Communication Channel

Implementing encrypted messaging using the shared secret:

```typescript
// Example of using ECDH with AES encryption
```

### On-chain Key Exchange Protocol

```typescript
// Example of an on-chain key exchange protocol
```

## Security Considerations

- Protecting private keys
- Man-in-the-middle attack prevention
- Key derivation functions

## Advanced ECDH Techniques

### Combining with Digital Signatures

```typescript
// Example of authenticated key exchange
```

### Multiple Party Key Agreement

```typescript
// Example of multi-party ECDH
```

## Conclusion

In this tutorial, you've learned how to implement ECDH key exchange using the BSV TypeScript SDK. You now understand how to generate key pairs, derive shared secrets, and apply these techniques in practical Bitcoin applications.

## Further Reading

- [ECDH Specification]()
- [Cryptographic Protocol Security]()
