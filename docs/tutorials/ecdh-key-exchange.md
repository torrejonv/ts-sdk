# ECDH Key Exchange

**Duration**: 75 minutes

**Prerequisites**: Basic TypeScript knowledge, [Elliptic Curve Fundamentals](./elliptic-curve-fundamentals.md) tutorial completed

## Learning Goals

- Understand Elliptic Curve Diffie-Hellman (ECDH) key exchange principles
- Implement secure key exchange using the BSV TypeScript SDK
- Create shared secrets for encrypted communication
- Apply ECDH in practical Bitcoin applications
- Understand security considerations and best practices

## Introduction to ECDH

Elliptic Curve Diffie-Hellman (ECDH) is a key agreement protocol that allows two parties to establish a shared secret over an unsecured communication channel. Unlike traditional encryption where you need to share a secret key beforehand, ECDH allows two parties who have never met to create a shared secret that only they know.

The mathematical foundation of ECDH relies on the commutative property of elliptic curve point multiplication:

- Alice computes: `(Alice's private key) × (Bob's public key)`
- Bob computes: `(Bob's private key) × (Alice's public key)`
- Both arrive at the same shared secret point

## Setting Up Your Environment

```typescript
import { PrivateKey, PublicKey, Point, BigNumber } from '@bsv/sdk'
```

## Basic ECDH Key Exchange

### Step 1: Key Generation

Let's start by generating key pairs for Alice and Bob:

```typescript
function generateKeyPairs() {
  console.log('=== Generating Key Pairs ===')
  
  // Generate Alice's key pair
  const alicePrivKey = PrivateKey.fromRandom()
  const alicePubKey = alicePrivKey.toPublicKey()
  
  // Generate Bob's key pair
  const bobPrivKey = PrivateKey.fromRandom()
  const bobPubKey = bobPrivKey.toPublicKey()
  
  console.log('Alice private key:', alicePrivKey.toWif())
  console.log('Alice public key:', alicePubKey.toString())
  console.log('Bob private key:', bobPrivKey.toWif())
  console.log('Bob public key:', bobPubKey.toString())
  
  return { alicePrivKey, alicePubKey, bobPrivKey, bobPubKey }
}
```

### Step 2: Deriving Shared Secrets

Now both parties can derive the same shared secret:

```typescript
function performECDH(alicePrivKey: PrivateKey, alicePubKey: PublicKey, 
                     bobPrivKey: PrivateKey, bobPubKey: PublicKey) {
  console.log('\n=== ECDH Key Exchange ===')
  
  // Alice creates a shared secret using Bob's public key and her private key
  const aliceSharedSecret = alicePrivKey.deriveSharedSecret(bobPubKey)
  
  // Bob creates the same shared secret using Alice's public key and his private key
  const bobSharedSecret = bobPrivKey.deriveSharedSecret(alicePubKey)
  
  // Verify they're identical
  const aliceSecretHex = aliceSharedSecret.getX().toHex()
  const bobSecretHex = bobSharedSecret.getX().toHex()
  
  console.log('Alice\'s shared secret (x-coordinate):', aliceSecretHex)
  console.log('Bob\'s shared secret (x-coordinate):', bobSecretHex)
  console.log('Secrets match:', aliceSecretHex === bobSecretHex)
  
  return aliceSharedSecret
}
```

### Step 3: Complete Example

```typescript
function basicECDHExample() {
  try {
    // Generate key pairs
    const { alicePrivKey, alicePubKey, bobPrivKey, bobPubKey } = generateKeyPairs()
    
    // Perform ECDH
    const sharedSecret = performECDH(alicePrivKey, alicePubKey, bobPrivKey, bobPubKey)
    
    // The shared secret is a point on the curve
    console.log('\nShared secret point:')
    console.log('X:', sharedSecret.getX().toHex())
    console.log('Y:', sharedSecret.getY().toHex())
    
  } catch (error) {
    console.error('ECDH Error:', error.message)
  }
}

// Run the example
basicECDHExample()
```

## Security Validation

The SDK includes built-in security checks to prevent twist attacks:

```typescript
function demonstrateSecurityValidation() {
  console.log('\n=== Security Validation ===')
  
  const validPrivKey = PrivateKey.fromRandom()
  
  // This will work - valid public key
  const validPubKey = PrivateKey.fromRandom().toPublicKey()
  const validSecret = validPrivKey.deriveSharedSecret(validPubKey)
  console.log('Valid ECDH succeeded')
  
  // This will fail - invalid point (not on curve)
  try {
    const invalidPubKey = new PublicKey(new BigNumber(14), new BigNumber(16))
    validPrivKey.deriveSharedSecret(invalidPubKey)
  } catch (error) {
    console.log('Security check prevented invalid key usage:', error.message)
  }
}

demonstrateSecurityValidation()
```

## Practical Applications

### Secure Message Exchange

Here's how to use ECDH for encrypting messages:

```typescript
import { createHash, createCipheriv, createDecipheriv, randomBytes } from 'crypto'

function deriveEncryptionKey(sharedSecret: Point): Buffer {
  // Use the x-coordinate of the shared secret as key material
  const keyMaterial = sharedSecret.getX().toArray('be', 32)
  
  // Hash to create a proper encryption key
  return createHash('sha256').update(Buffer.from(keyMaterial)).digest()
}

function encryptMessage(message: string, sharedSecret: Point): { 
  encrypted: string, 
  iv: string 
} {
  const key = deriveEncryptionKey(sharedSecret)
  const iv = randomBytes(16)
  const cipher = createCipheriv('aes-256-cbc', key, iv)
  
  let encrypted = cipher.update(message, 'utf8', 'hex')
  encrypted += cipher.final('hex')
  
  return {
    encrypted,
    iv: iv.toString('hex')
  }
}

function decryptMessage(encryptedData: { encrypted: string, iv: string }, 
                       sharedSecret: Point): string {
  const key = deriveEncryptionKey(sharedSecret)
  const decipher = createDecipheriv('aes-256-cbc', key, Buffer.from(encryptedData.iv, 'hex'))
  
  let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8')
  decrypted += decipher.final('utf8')
  
  return decrypted
}

function secureMessagingExample() {
  console.log('\n=== Secure Messaging with ECDH ===')
  
  // Setup key pairs
  const alicePrivKey = PrivateKey.fromRandom()
  const bobPrivKey = PrivateKey.fromRandom()
  const alicePubKey = alicePrivKey.toPublicKey()
  const bobPubKey = bobPrivKey.toPublicKey()
  
  // Derive shared secret
  const sharedSecret = alicePrivKey.deriveSharedSecret(bobPubKey)
  
  // Alice encrypts a message
  const message = "Hello Bob! This is a secret message."
  const encryptedData = encryptMessage(message, sharedSecret)
  
  console.log('Original message:', message)
  console.log('Encrypted:', encryptedData.encrypted)
  
  // Bob decrypts the message using the same shared secret
  const bobSharedSecret = bobPrivKey.deriveSharedSecret(alicePubKey)
  const decryptedMessage = decryptMessage(encryptedData, bobSharedSecret)
  
  console.log('Decrypted message:', decryptedMessage)
  console.log('Messages match:', message === decryptedMessage)
}

secureMessagingExample()
```

### Key Exchange with Authentication

Combine ECDH with digital signatures for authenticated key exchange:

```typescript
function authenticatedKeyExchange() {
  console.log('\n=== Authenticated Key Exchange ===')
  
  // Generate long-term identity keys
  const aliceIdentityPrivKey = PrivateKey.fromRandom()
  const bobIdentityPrivKey = PrivateKey.fromRandom()
  const aliceIdentityPubKey = aliceIdentityPrivKey.toPublicKey()
  const bobIdentityPubKey = bobIdentityPrivKey.toPublicKey()
  
  // Generate ephemeral keys for this session
  const aliceEphemeralPrivKey = PrivateKey.fromRandom()
  const bobEphemeralPrivKey = PrivateKey.fromRandom()
  const aliceEphemeralPubKey = aliceEphemeralPrivKey.toPublicKey()
  const bobEphemeralPubKey = bobEphemeralPrivKey.toPublicKey()
  
  // Alice signs her ephemeral public key with her identity key
  const aliceSignature = aliceIdentityPrivKey.sign(
    Buffer.from(aliceEphemeralPubKey.toString(), 'utf8')
  )
  
  // Bob signs his ephemeral public key with his identity key
  const bobSignature = bobIdentityPrivKey.sign(
    Buffer.from(bobEphemeralPubKey.toString(), 'utf8')
  )
  
  // Verify signatures (in practice, you'd exchange these over the network)
  const aliceSignatureValid = aliceIdentityPubKey.verify(
    Buffer.from(aliceEphemeralPubKey.toString(), 'utf8'),
    aliceSignature
  )
  
  const bobSignatureValid = bobIdentityPubKey.verify(
    Buffer.from(bobEphemeralPubKey.toString(), 'utf8'),
    bobSignature
  )
  
  console.log('Alice signature valid:', aliceSignatureValid)
  console.log('Bob signature valid:', bobSignatureValid)
  
  if (aliceSignatureValid && bobSignatureValid) {
    // Perform ECDH with ephemeral keys
    const sharedSecret = aliceEphemeralPrivKey.deriveSharedSecret(bobEphemeralPubKey)
    console.log('Authenticated shared secret established')
    console.log('Secret (x-coordinate):', sharedSecret.getX().toHex().substring(0, 16) + '...')
  }
}

authenticatedKeyExchange()
```

## Advanced ECDH Patterns

### Multi-Party Key Agreement

Extend ECDH to multiple parties:

```typescript
function multiPartyKeyAgreement() {
  console.log('\n=== Multi-Party Key Agreement ===')
  
  // Generate keys for three parties
  const parties = ['Alice', 'Bob', 'Charlie'].map(name => ({
    name,
    privKey: PrivateKey.fromRandom(),
    pubKey: null as PublicKey | null
  }))
  
  // Generate public keys
  parties.forEach(party => {
    party.pubKey = party.privKey.toPublicKey()
  })
  
  // Each party computes pairwise shared secrets
  const sharedSecrets = new Map<string, Point>()
  
  for (let i = 0; i < parties.length; i++) {
    for (let j = i + 1; j < parties.length; j++) {
      const party1 = parties[i]
      const party2 = parties[j]
      
      const secret = party1.privKey.deriveSharedSecret(party2.pubKey!)
      const pairKey = `${party1.name}-${party2.name}`
      sharedSecrets.set(pairKey, secret)
      
      console.log(`${pairKey} shared secret:`, secret.getX().toHex().substring(0, 16) + '...')
    }
  }
  
  return sharedSecrets
}

multiPartyKeyAgreement()
```

### Key Derivation Functions

Use proper key derivation for different purposes:

```typescript
function keyDerivationExample() {
  console.log('\n=== Key Derivation Functions ===')
  
  const alicePrivKey = PrivateKey.fromRandom()
  const bobPrivKey = PrivateKey.fromRandom()
  const sharedSecret = alicePrivKey.deriveSharedSecret(bobPrivKey.toPublicKey())
  
  // Derive different keys for different purposes
  function deriveKey(purpose: string, length: number = 32): Buffer {
    const keyMaterial = sharedSecret.getX().toArray('be', 32)
    const hash = createHash('sha256')
    hash.update(Buffer.from(keyMaterial))
    hash.update(Buffer.from(purpose, 'utf8'))
    return hash.digest().slice(0, length)
  }
  
  const encryptionKey = deriveKey('encryption', 32)
  const macKey = deriveKey('authentication', 32)
  const ivKey = deriveKey('iv', 16)
  
  console.log('Encryption key:', encryptionKey.toString('hex'))
  console.log('MAC key:', macKey.toString('hex'))
  console.log('IV key:', ivKey.toString('hex'))
}

keyDerivationExample()
```

## Security Considerations

### Best Practices

1. **Key Validation**: Always validate public keys before use
2. **Ephemeral Keys**: Use ephemeral keys for forward secrecy
3. **Authentication**: Combine with signatures to prevent man-in-the-middle attacks
4. **Key Derivation**: Use proper KDFs to derive encryption keys from shared secrets

### Common Pitfalls

```typescript
function securityPitfalls() {
  console.log('\n=== Security Pitfalls to Avoid ===')
  
  // ❌ DON'T: Use shared secret directly as encryption key
  console.log('❌ Never use the shared secret point directly for encryption')
  
  // ✅ DO: Use proper key derivation
  console.log('✅ Always use key derivation functions')
  
  // ❌ DON'T: Reuse ephemeral keys
  console.log('❌ Never reuse ephemeral keys across sessions')
  
  // ✅ DO: Generate fresh ephemeral keys for each session
  console.log('✅ Generate fresh keys for each exchange')
  
  // ❌ DON'T: Skip public key validation
  console.log('❌ Never skip public key validation')
  
  // ✅ DO: Always validate received public keys
  console.log('✅ SDK automatically validates keys in deriveSharedSecret()')
}

securityPitfalls()
```

## Performance Considerations

### Optimizing ECDH Operations

```typescript
function performanceExample() {
  console.log('\n=== Performance Optimization ===')
  
  const iterations = 1000
  
  // Pre-generate keys
  const privateKeys = Array.from({ length: iterations }, () => PrivateKey.fromRandom())
  const publicKeys = privateKeys.map(pk => pk.toPublicKey())
  
  // Measure ECDH performance
  const startTime = Date.now()
  
  for (let i = 0; i < iterations; i++) {
    const sharedSecret = privateKeys[i].deriveSharedSecret(publicKeys[(i + 1) % iterations])
    // In practice, you'd process the shared secret here
  }
  
  const endTime = Date.now()
  const avgTime = (endTime - startTime) / iterations
  
  console.log(`Performed ${iterations} ECDH operations`)
  console.log(`Average time per operation: ${avgTime.toFixed(2)}ms`)
}

performanceExample()
```

## Error Handling

### Robust ECDH Implementation

```typescript
function robustECDH(privateKey: PrivateKey, publicKey: PublicKey): Point | null {
  try {
    // Validate inputs
    if (!privateKey || !publicKey) {
      throw new Error('Invalid key parameters')
    }
    
    // Perform ECDH with built-in validation
    const sharedSecret = privateKey.deriveSharedSecret(publicKey)
    
    // Additional validation if needed
    if (sharedSecret.getX().isZero() || sharedSecret.getY().isZero()) {
      throw new Error('Invalid shared secret generated')
    }
    
    return sharedSecret
    
  } catch (error) {
    console.error('ECDH operation failed:', error.message)
    return null
  }
}

function errorHandlingExample() {
  console.log('\n=== Error Handling ===')
  
  const validPrivKey = PrivateKey.fromRandom()
  const validPubKey = PrivateKey.fromRandom().toPublicKey()
  
  // Test with valid keys
  const result1 = robustECDH(validPrivKey, validPubKey)
  console.log('Valid ECDH result:', result1 ? 'Success' : 'Failed')
  
  // Test with invalid key (will be caught by SDK validation)
  try {
    const invalidPubKey = new PublicKey(new BigNumber(1), new BigNumber(1))
    const result2 = robustECDH(validPrivKey, invalidPubKey)
    console.log('Invalid ECDH result:', result2 ? 'Success' : 'Failed')
  } catch (error) {
    console.log('Caught invalid key error:', error.message)
  }
}

errorHandlingExample()
```

## Testing Your ECDH Implementation

### Comprehensive Test Suite

```typescript
function testECDHImplementation() {
  console.log('\n=== ECDH Test Suite ===')
  
  let passed = 0
  let total = 0
  
  function test(name: string, testFn: () => boolean) {
    total++
    try {
      if (testFn()) {
        console.log(`✅ ${name}`)
        passed++
      } else {
        console.log(`❌ ${name}`)
      }
    } catch (error) {
      console.log(`❌ ${name}: ${error.message}`)
    }
  }
  
  // Test 1: Basic ECDH symmetry
  test('Basic ECDH symmetry', () => {
    const privA = PrivateKey.fromRandom()
    const privB = PrivateKey.fromRandom()
    const secretA = privA.deriveSharedSecret(privB.toPublicKey())
    const secretB = privB.deriveSharedSecret(privA.toPublicKey())
    return secretA.getX().toHex() === secretB.getX().toHex()
  })
  
  // Test 2: Different key formats
  test('Different key formats', () => {
    const privA = PrivateKey.fromRandom()
    const privB = PrivateKey.fromRandom()
    const pubB = PublicKey.fromString(privB.toPublicKey().toDER('hex') as string)
    const secret1 = privA.deriveSharedSecret(privB.toPublicKey())
    const secret2 = privA.deriveSharedSecret(pubB)
    return secret1.getX().toHex() === secret2.getX().toHex()
  })
  
  // Test 3: Invalid key rejection
  test('Invalid key rejection', () => {
    const privKey = PrivateKey.fromRandom()
    const invalidPubKey = new PublicKey(new BigNumber(14), new BigNumber(16))
    try {
      privKey.deriveSharedSecret(invalidPubKey)
      return false // Should have thrown
    } catch (error) {
      return error.message.includes('not valid for ECDH')
    }
  })
  
  console.log(`\nTest Results: ${passed}/${total} passed`)
  return passed === total
}

testECDHImplementation()
```

## Conclusion

In this tutorial, you've learned how to implement ECDH key exchange using the BSV TypeScript SDK. You now understand:

- The mathematical principles behind ECDH
- How to generate key pairs and derive shared secrets
- Security considerations and validation
- Practical applications including secure messaging
- Advanced patterns like authenticated key exchange
- Performance optimization and error handling

The BSV TypeScript SDK provides robust ECDH implementation with built-in security validations, making it safe and easy to implement secure key exchange protocols.

## Next Steps

- **[Script Construction](./script-construction.md)**: Learn to create custom Bitcoin scripts
- **[Advanced Transaction Construction](./advanced-transaction.md)**: Build complex transactions
- **[SPV and Merkle Proofs](./spv-merkle-proofs.md)**: Implement lightweight verification

## Further Reading

- [RFC 3526 - Diffie-Hellman Key Agreement](https://tools.ietf.org/html/rfc3526)
- [SEC 1: Elliptic Curve Cryptography](https://www.secg.org/sec1-v2.pdf)
- [BSV TypeScript SDK Documentation](../reference/primitives.md)
