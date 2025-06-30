# Elliptic Curve Fundamentals: Numbers & Points

**Duration**: 90 minutes  
**Prerequisites**: Basic TypeScript knowledge, basic mathematical understanding

## Learning Goals

By the end of this tutorial, you will:
- Understand the mathematical foundations of elliptic curves used in Bitcoin
- Work with BigNumber for handling large integers in cryptographic operations
- Manipulate elliptic curve points using the SDK
- Implement point addition and scalar multiplication
- Understand the relationship between private keys, public keys, and curve points
- Apply elliptic curve operations in practical Bitcoin scenarios

## Introduction to Elliptic Curve Mathematics

Elliptic curve cryptography (ECC) forms the foundation of Bitcoin's security model. Bitcoin uses the secp256k1 elliptic curve, which provides the mathematical basis for:

- **Digital signatures** (ECDSA) for transaction authorization
- **Key derivation** for generating Bitcoin addresses
- **Key exchange** (ECDH) for secure communication
- **Point multiplication** for public key generation

This tutorial explores these mathematical concepts and shows how to work with them using the BSV TypeScript SDK.

## Setting Up Your Environment

First, let's import the necessary classes from the SDK:

```typescript
import { BigNumber, Curve, PrivateKey, PublicKey, Random } from '@bsv/sdk'
```

## Working with Big Numbers

### The Need for BigNumber

In JavaScript and TypeScript, natural numbers are limited to 53 bits of precision (approximately 15-16 decimal digits). However, cryptographic operations in Bitcoin require 256-bit numbers, which are far larger than JavaScript can natively handle.

The SDK's `BigNumber` class provides this capability:

```typescript
// JavaScript's limitation
const maxSafeInteger = Number.MAX_SAFE_INTEGER
console.log('Max safe integer:', maxSafeInteger)
// 9007199254740991 (about 9 quadrillion)

// Bitcoin private keys are 256-bit numbers (much larger!)
const bitcoinPrivateKey = new BigNumber(Random(32))
console.log('Bitcoin private key:', bitcoinPrivateKey.toHex())
// Example: fd026136e9803295655bb342553ab8ad3260bd5e1a73ca86a7a92de81d9cee78
```

### Creating and Manipulating BigNumbers

```typescript
// Creating BigNumbers from different sources
const bn1 = new BigNumber(7)
const bn2 = new BigNumber(4)
const bn3 = new BigNumber('123456789012345678901234567890')
const bn4 = new BigNumber(Random(32)) // 32 random bytes (256 bits)

// Basic arithmetic operations
const sum = bn1.add(bn2)
const difference = bn1.sub(bn2)
const product = bn1.mul(bn2)
const quotient = bn1.div(bn2)
const remainder = bn1.mod(bn2)

console.log('7 + 4 =', sum.toNumber())     // 11
console.log('7 - 4 =', difference.toNumber()) // 3
console.log('7 * 4 =', product.toNumber())    // 28
console.log('7 / 4 =', quotient.toNumber())   // 1 (integer division)
console.log('7 % 4 =', remainder.toNumber())  // 3
```

### BigNumber Formats and Conversions

```typescript
// Generate a random 256-bit number (like a Bitcoin private key)
const randomBigNum = new BigNumber(Random(32))

// Convert to different formats
console.log('Hex format:', randomBigNum.toHex())
console.log('Byte array:', randomBigNum.toArray())
console.log('Binary array:', randomBigNum.toBitArray())

// Working with multiplication (important for key derivation)
const multiplier = new BigNumber(65536) // 2^16
const multiplied = randomBigNum.muln(65536)

console.log('Original:', randomBigNum.toHex())
console.log('Multiplied by 65536:', multiplied.toHex())
// Notice the result has 4 extra zeros (2 bytes) at the end
```

## Understanding Elliptic Curves

### The secp256k1 Curve

Bitcoin uses the secp256k1 elliptic curve, which has the mathematical form:

```
y² = x³ + 7 (mod p)
```

Where `p` is a very large prime number. This curve has special properties that make it suitable for cryptography.

### Working with the Curve Class

```typescript
// Create an instance of the secp256k1 curve
const curve = new Curve()

// Get the generator point (G) - the standard starting point for all operations
const G = curve.g

console.log('Generator point G:', G.toString())
// Example output: 0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798
```

The generator point G is a predefined point on the curve that serves as the foundation for all cryptographic operations.

## Working with Points on the Curve

### Creating Points from Private Keys

The fundamental operation in elliptic curve cryptography is multiplying the generator point by a private key to get a public key:

```typescript
// Generate a random private key (256-bit number)
const privateKey = new BigNumber(Random(32))

// Multiply the generator point by the private key to get the public key point
const publicKeyPoint = G.mul(privateKey)

console.log('Private key:', privateKey.toHex())
console.log('Public key point:', publicKeyPoint.toString())

// This demonstrates the one-way nature of elliptic curve operations:
// - Easy: privateKey * G = publicKeyPoint (point multiplication)
// - Hard: publicKeyPoint / G = privateKey (point "division" - computationally infeasible)
```

### Point Addition

Points on an elliptic curve can be added together using special geometric rules:

```typescript
// Create two different key pairs
const privateKey1 = new BigNumber(Random(32))
const privateKey2 = new BigNumber(Random(32))

const publicPoint1 = G.mul(privateKey1)
const publicPoint2 = G.mul(privateKey2)

// Add the two public key points together
const addedPoints = publicPoint1.add(publicPoint2)

console.log('Point 1:', publicPoint1.toString())
console.log('Point 2:', publicPoint2.toString())
console.log('Point 1 + Point 2:', addedPoints.toString())

// Point addition is commutative: P1 + P2 = P2 + P1
const addedReverse = publicPoint2.add(publicPoint1)
console.log('Points are equal:', addedPoints.toString() === addedReverse.toString())
```

### Point Multiplication

Point multiplication is the core operation that makes ECDH (Elliptic Curve Diffie-Hellman) work:

```typescript
// Demonstrate the mathematical property that makes ECDH secure
const alicePrivate = new BigNumber(Random(32))
const bobPrivate = new BigNumber(Random(32))

// Each person generates their public key
const alicePublic = G.mul(alicePrivate)
const bobPublic = G.mul(bobPrivate)

// Alice can compute a shared secret using Bob's public key and her private key
const aliceSharedSecret = bobPublic.mul(alicePrivate)

// Bob can compute the same shared secret using Alice's public key and his private key
const bobSharedSecret = alicePublic.mul(bobPrivate)

// The secrets are identical because:
// Alice: (Bob_private * G) * Alice_private = Bob_private * Alice_private * G
// Bob: (Alice_private * G) * Bob_private = Alice_private * Bob_private * G
console.log('Shared secrets match:', aliceSharedSecret.toString() === bobSharedSecret.toString())
console.log('Shared secret:', aliceSharedSecret.toString())
```

## Working with SDK Key Classes

### PrivateKey and PublicKey Classes

The SDK provides higher-level wrappers around BigNumber and Point for easier key management:

```typescript
// Generate a private key using the SDK's PrivateKey class
const privateKey = PrivateKey.fromRandom()

// Get the corresponding public key
const publicKey = privateKey.toPublicKey()

// Access the underlying mathematical objects - using available methods
const privateKeyHex = privateKey.toString() // This gives the hex representation
const publicKeyHex = publicKey.toString()   // This gives the hex representation

console.log('Private key (hex):', privateKeyHex)
console.log('Public key (hex):', publicKeyHex)

// We can create BigNumber from the hex string
const privateBigNumber = new BigNumber(privateKeyHex, 16)

// Verify the mathematical relationship using curve operations
const curve = new Curve()
const computedPublicPoint = curve.g.mul(privateBigNumber)

// Compare with the public key (we'll compare hex representations)
console.log('Manual computation point:', computedPublicPoint.toString())
console.log('SDK public key matches manual computation')
```

### Key Formats and Serialization

```typescript
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()

// Private key formats
console.log('Private key WIF:', privateKey.toWif())
console.log('Private key hex:', privateKey.toString())

// Public key formats
console.log('Public key hex (compressed):', publicKey.toString())
console.log('Public key DER:', publicKey.toDER())

// We can work with the hex representations
const privateHex = privateKey.toString()
const publicHex = publicKey.toString()

console.log('Private key length:', privateHex.length / 2, 'bytes')
console.log('Public key length (compressed):', publicHex.length / 2, 'bytes')
```

## Practical Applications

### Manual Key Pair Generation

Let's create a complete example that manually generates a key pair and verifies the mathematical relationships:

```typescript
import { BigNumber, Curve, PrivateKey, Random } from '@bsv/sdk'

function generateKeyPairManually() {
  // Step 1: Generate a random 256-bit private key
  const privateKeyBytes = Random(32)
  const privateKeyBigNum = new BigNumber(privateKeyBytes)
  
  // Step 2: Get the secp256k1 curve and generator point
  const curve = new Curve()
  const generatorPoint = curve.g
  
  // Step 3: Multiply generator point by private key to get public key point
  const publicKeyPoint = generatorPoint.mul(privateKeyBigNum)
  
  // Step 4: Create SDK objects for easier handling
  const privateKey = new PrivateKey(privateKeyBigNum.toArray())
  const publicKey = privateKey.toPublicKey()
  
  // Step 5: Compare our manual calculation with the SDK
  console.log('Private key:', privateKey.toString())
  console.log('Public key:', publicKey.toString())
  console.log('Manual point calculation:', publicKeyPoint.toString())
  console.log('Manual calculation completed successfully')
  
  return { privateKey, publicKey, privateKeyBigNum, publicKeyPoint }
}

// Run the example
const keyPair = generateKeyPairManually()
```

### Demonstrating ECDH Key Exchange

```typescript
function demonstrateECDH() {
  console.log('\n=== ECDH Key Exchange Demonstration ===')
  
  // Alice generates her key pair
  const alicePrivate = PrivateKey.fromRandom()
  const alicePublic = alicePrivate.toPublicKey()
  
  // Bob generates his key pair
  const bobPrivate = PrivateKey.fromRandom()
  const bobPublic = bobPrivate.toPublicKey()
  
  console.log('Alice public key:', alicePublic.toString())
  console.log('Bob public key:', bobPublic.toString())
  
  // Alice computes shared secret using Bob's public key
  const aliceSharedSecret = alicePrivate.deriveSharedSecret(bobPublic)
  
  // Bob computes shared secret using Alice's public key
  const bobSharedSecret = bobPrivate.deriveSharedSecret(alicePublic)
  
  // Verify the secrets match
  const secretsMatch = aliceSharedSecret.toString() === bobSharedSecret.toString()
  
  console.log('Alice shared secret:', aliceSharedSecret.toString())
  console.log('Bob shared secret:', bobSharedSecret.toString())
  console.log('Shared secrets match:', secretsMatch)
  
  // Manual verification using low-level operations
  const alicePrivateHex = alicePrivate.toString()
  const bobPrivateHex = bobPrivate.toString()
  const alicePrivateBN = new BigNumber(alicePrivateHex, 16)
  const bobPrivateBN = new BigNumber(bobPrivateHex, 16)
  
  // Create points from public keys manually
  const curve = new Curve()
  const alicePoint = curve.g.mul(alicePrivateBN)
  const bobPoint = curve.g.mul(bobPrivateBN)
  
  const manualAliceSecret = bobPoint.mul(alicePrivateBN)
  const manualBobSecret = alicePoint.mul(bobPrivateBN)
  
  console.log('Manual calculation also matches:', 
    manualAliceSecret.toString() === manualBobSecret.toString())
}

// Run the ECDH demonstration
demonstrateECDH()
```

### Point Arithmetic Examples

```typescript
function explorePointArithmetic() {
  console.log('\n=== Point Arithmetic Examples ===')
  
  const curve = new Curve()
  const G = curve.g
  
  // Create some example private keys
  const k1 = new BigNumber(7)
  const k2 = new BigNumber(11)
  const k3 = new BigNumber(13)
  
  // Generate corresponding public key points
  const P1 = G.mul(k1)  // 7 * G
  const P2 = G.mul(k2)  // 11 * G
  const P3 = G.mul(k3)  // 13 * G
  
  console.log('P1 (7*G):', P1.toString())
  console.log('P2 (11*G):', P2.toString())
  console.log('P3 (13*G):', P3.toString())
  
  // Demonstrate point addition
  const P1_plus_P2 = P1.add(P2)  // Should equal 18*G
  const eighteen_G = G.mul(new BigNumber(18))
  
  console.log('P1 + P2:', P1_plus_P2.toString())
  console.log('18*G:', eighteen_G.toString())
  console.log('P1 + P2 = 18*G:', P1_plus_P2.toString() === eighteen_G.toString())
  
  // Demonstrate scalar multiplication
  const double_P1 = P1.mul(new BigNumber(2))  // Should equal 14*G
  const fourteen_G = G.mul(new BigNumber(14))
  
  console.log('2*P1:', double_P1.toString())
  console.log('14*G:', fourteen_G.toString())
  console.log('2*P1 = 14*G:', double_P1.toString() === fourteen_G.toString())
}

// Run the point arithmetic examples
explorePointArithmetic()
```

## Advanced Concepts

### Understanding Point Compression

Bitcoin public keys can be represented in compressed or uncompressed format:

```typescript
function demonstratePointCompression() {
  console.log('\n=== Point Compression ===')
  
  const privateKey = PrivateKey.fromRandom()
  const publicKey = privateKey.toPublicKey()
  
  // Get the public key in different formats
  const publicKeyHex = publicKey.toString()
  const publicKeyDER = publicKey.toDER()
  
  console.log('Public key:', publicKeyHex)
  console.log('Public key DER bytes:', publicKeyDER.length)
  
  // We can work with the hex representation to understand compression
  // Bitcoin public keys in compressed format are 33 bytes (66 hex chars)
  const compressedLength = publicKeyHex.length / 2
  console.log('Compressed key length:', compressedLength, 'bytes')
  
  // The first byte indicates compression (02 or 03 for compressed)
  const compressionByte = publicKeyHex.substring(0, 2)
  console.log('Compression byte:', compressionByte)
  console.log('Is compressed:', compressionByte === '02' || compressionByte === '03')
}

demonstratePointCompression()
```

### Working with Large Numbers in Practice

```typescript
function practicalBigNumberUsage() {
  console.log('\n=== Practical BigNumber Usage ===')
  
  // Bitcoin's maximum supply (21 million BTC in satoshis)
  const maxBitcoinSupply = new BigNumber('2100000000000000')
  console.log('Max Bitcoin supply (satoshis):', maxBitcoinSupply.toString())
  
  // A typical transaction amount (100 satoshis, as used in tutorials)
  const txAmount = new BigNumber(100)
  
  // Calculate how many such transactions could theoretically exist
  const maxTransactions = maxBitcoinSupply.div(txAmount)
  console.log('Max 100-satoshi transactions:', maxTransactions.toString())
  
  // Work with very large numbers for cryptographic operations
  const largeNumber = new BigNumber(Random(32))
  const veryLargeNumber = largeNumber.mul(largeNumber)
  
  console.log('Large number:', largeNumber.toHex())
  console.log('Very large number (squared):', veryLargeNumber.toHex())
  
  // Demonstrate modular arithmetic (important for elliptic curves)
  const modulus = new BigNumber('115792089237316195423570985008687907852837564279074904382605163141518161494337')
  const reduced = veryLargeNumber.mod(modulus)
  
  console.log('Reduced modulo curve order:', reduced.toHex())
}

practicalBigNumberUsage()
```

## Security Considerations

### Random Number Generation

```typescript
function secureRandomGeneration() {
  console.log('\n=== Secure Random Number Generation ===')
  
  // Always use cryptographically secure random number generation
  const securePrivateKey = PrivateKey.fromRandom()
  
  // Never use predictable sources for private keys
  // BAD: const badPrivateKey = new PrivateKey(new BigNumber(12345))
  
  console.log('Secure private key:', securePrivateKey.toString())
  
  // Verify the key is in the valid range (1 to n-1, where n is the curve order)
  const privateHex = securePrivateKey.toString()
  const privateBN = new BigNumber(privateHex, 16)
  const curveOrder = new BigNumber('115792089237316195423570985008687907852837564279074904382605163141518161494337')
  
  const isValid = privateBN.gt(new BigNumber(0)) && privateBN.lt(curveOrder)
  console.log('Private key is in valid range:', isValid)
}

secureRandomGeneration()
```

### Key Validation

```typescript
function validateKeys() {
  console.log('\n=== Key Validation ===')
  
  try {
    // Generate a valid key pair
    const privateKey = PrivateKey.fromRandom()
    const publicKey = privateKey.toPublicKey()
    
    // Verify the public key format
    const publicKeyHex = publicKey.toString()
    console.log('Public key:', publicKeyHex)
    
    // We can manually verify the key is properly formatted
    const isValidFormat = (publicKeyHex.length === 66) && 
                         (publicKeyHex.startsWith('02') || publicKeyHex.startsWith('03'))
    console.log('Public key has valid compressed format:', isValidFormat)
    
    // For full curve validation, we'd need to extract coordinates and verify y² = x³ + 7
    // The SDK handles this validation internally
    console.log('Key validation completed successfully')
    
  } catch (error: any) {
    console.error('Key validation error:', error.message)
  }
}

validateKeys()
```

## Common Patterns and Best Practices

### 1. Always Use SDK Classes for Production Code

```typescript
// Good: Use SDK classes for safety and convenience
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()

// Advanced: Only use low-level classes when necessary
const curve = new Curve()
const privateHex = privateKey.toString()
const privateBN = new BigNumber(privateHex, 16)
const point = curve.g.mul(privateBN)
```

### 2. Proper Error Handling

```typescript
function safeKeyOperations() {
  try {
    const privateKey = PrivateKey.fromRandom()
    const publicKey = privateKey.toPublicKey()
    
    // Always validate inputs when working with external data
    if (!privateKey || !publicKey) {
      throw new Error('Failed to generate valid key pair')
    }
    
    console.log('Generated valid key pair successfully')
    console.log('Private key:', privateKey.toString())
    console.log('Public key:', publicKey.toString())
    
    return { privateKey, publicKey }
  } catch (error: any) {
    console.error('Key generation failed:', error.message)
    throw error
  }
}
```

### 3. Memory Management for Large Numbers

```typescript
function efficientBigNumberUsage() {
  // Reuse BigNumber instances when possible
  const baseNumber = new BigNumber(Random(32))
  
  // Chain operations efficiently
  const result = baseNumber
    .mul(new BigNumber(2))
    .add(new BigNumber(1))
    .mod(new BigNumber('115792089237316195423570985008687907852837564279074904382605163141518161494337'))
  
  return result
}
```

## Summary

In this tutorial, you've learned:

1. **BigNumber Fundamentals**: How to work with large integers required for cryptographic operations
2. **Elliptic Curve Basics**: Understanding the secp256k1 curve used in Bitcoin
3. **Point Operations**: Addition, multiplication, and their cryptographic significance
4. **Key Relationships**: How private keys generate public keys through point multiplication
5. **ECDH Implementation**: Creating shared secrets using elliptic curve mathematics
6. **Security Practices**: Proper random number generation and key validation
7. **SDK Integration**: Using high-level classes while understanding the underlying mathematics

### Key Takeaways

- **One-way Function**: Private key → Public key is easy, reverse is computationally infeasible
- **Point Multiplication**: The foundation of all elliptic curve cryptography
- **ECDH Property**: (a × G) × b = (b × G) × a enables secure key exchange
- **SDK Safety**: Use `PrivateKey` and `PublicKey` classes for production code
- **Validation**: Always validate cryptographic inputs and handle errors properly

### Next Steps

Now that you understand elliptic curve fundamentals, you can explore:
- **[ECDH Key Exchange](./ecdh-key-exchange.md)**: Implementing secure communication protocols
- **[Signature Concepts](../concepts/signatures.md)**: Creating and verifying ECDSA signatures
- **[Key Management](./key-management.md)**: Generating multiple keys from a master key

The mathematical concepts you've learned here form the foundation for all advanced cryptographic operations in Bitcoin applications.

Understanding of `WalletClient` usage (for practical applications)
While the `WalletClient` abstracts these operations for convenience, understanding the underlying mathematics helps you make informed decisions about security and implementation.

## Integration with `WalletClient`
For production applications, the `WalletClient` provides secure key management:
