# Type-42 Key Derivation

**Duration**: 75 minutes  
**Prerequisites**: Basic TypeScript knowledge, Elliptic Curve Fundamentals tutorial completed, ECDH Key Exchange tutorial completed

## Learning Goals

- Understand Type-42 key derivation protocol and its use cases
- Implement Type-42 operations with the BSV TypeScript SDK
- Create shared key universes between two parties
- Apply Type-42 in practical Bitcoin applications like message signing and encryption
- Understand the "anyone key" concept and its applications

## Introduction to Type-42

Type-42 is a key derivation protocol that enables two parties with master keys to derive child keys from one another using a specific string called an "invoice number." This creates a shared key universe that only these two parties can access, enabling secure communication and transactions without revealing their master keys.

The protocol gets its name from its historical use in Bitcoin payment systems, where invoice numbers were used to generate unique keys for each transaction. However, Type-42 has broader applications in secure messaging, authentication, and any scenario requiring shared key derivation.

## Setting Up Your Environment

```typescript
import { PrivateKey, PublicKey, Utils } from '@bsv/sdk'
```

## Understanding Type-42 Process

### The Mathematical Foundation

Type-42 key derivation follows these steps:

1. **Master Key Generation**: Each party generates a master private key
2. **Public Key Exchange**: Parties share their master public keys
3. **Shared Secret Creation**: Using ECDH, parties compute a shared secret
4. **Invoice Number Agreement**: Parties agree on a unique identifier (invoice number)
5. **HMAC Computation**: The shared secret is used as an HMAC key to hash the invoice number
6. **Key Derivation**: The HMAC output is used to derive child keys

### Security Properties

- Only the two parties can compute the shared secret
- Only they can use the HMAC function with their shared secret
- No one else can link master keys to derived keys
- Each invoice number creates a unique key pair in their shared universe

## Basic Type-42 Implementation

### Step 1: Master Key Setup

```typescript
// Alice generates her master key pair
const alice = PrivateKey.fromRandom()
const alicePub = alice.toPublicKey()

// Bob generates his master key pair
const bob = PrivateKey.fromRandom()
const bobPub = bob.toPublicKey()

console.log('Alice master public key:', alicePub.toString())
console.log('Bob master public key:', bobPub.toString())
```

### Step 2: Invoice Number Agreement

```typescript
// Both parties agree on an invoice number to use
// This could be a payment ID, message ID, or any unique identifier
const invoiceNumber = '2-simple signing protocol-1'

console.log('Using invoice number:', invoiceNumber)
```

### Step 3: Child Key Derivation

```typescript
// Alice derives a child private key for signing
const aliceSigningChild = alice.deriveChild(bobPub, invoiceNumber)

// Bob derives Alice's corresponding public key
const aliceSigningPub = alicePub.deriveChild(bob, invoiceNumber)

// Verify the keys match
const derivedPubFromPriv = aliceSigningChild.toPublicKey()
const keysMatch = derivedPubFromPriv.toString() === aliceSigningPub.toString()
console.log('Keys match:', keysMatch)
// true
```

## Practical Example: Message Signing

Let's implement a complete example where Alice signs a message for Bob using Type-42 derived keys:

```typescript
import { PrivateKey, Utils } from '@bsv/sdk'

async function demonstrateType42Signing() {
  // Step 1: Generate master keys
  const alice = PrivateKey.fromRandom()
  const alicePub = alice.toPublicKey()
  
  const bob = PrivateKey.fromRandom()
  const bobPub = bob.toPublicKey()
  
  // Step 2: Agree on invoice number
  const invoiceNumber = '2-secure-message-001'
  
  // Step 3: Alice derives her signing key
  const aliceSigningChild = alice.deriveChild(bobPub, invoiceNumber)
  
  // Step 4: Alice signs a message
  const message = Utils.toArray('Hello Bob, this is a secure message!', 'utf8')
  const signature = aliceSigningChild.sign(message)
  
  console.log('Message signed by Alice')
  console.log('Signature:', signature.toDER('hex'))
  
  // Step 5: Bob derives Alice's public key and verifies
  const aliceSigningPub = alicePub.deriveChild(bob, invoiceNumber)
  const verified = aliceSigningPub.verify(message, signature)
  
  console.log('Signature verified by Bob:', verified)
  // true
  
  return {
    alice,
    bob,
    aliceSigningChild,
    aliceSigningPub,
    message,
    signature,
    verified
  }
}

// Run the demonstration
demonstrateType42Signing()
```

## Bidirectional Communication

Type-42 enables both parties to derive keys for each other. Here's how Bob can also sign messages for Alice:

```typescript
async function bidirectionalType42() {
  const alice = PrivateKey.fromRandom()
  const alicePub = alice.toPublicKey()
  
  const bob = PrivateKey.fromRandom()
  const bobPub = bob.toPublicKey()
  
  const invoiceNumber = '2-bidirectional-chat-001'
  
  // Alice signs a message for Bob
  const aliceMessage = Utils.toArray('Hi Bob!', 'utf8')
  const aliceSigningKey = alice.deriveChild(bobPub, invoiceNumber)
  const aliceSignature = aliceSigningKey.sign(aliceMessage)
  
  // Bob signs a reply for Alice
  const bobMessage = Utils.toArray('Hi Alice!', 'utf8')
  const bobSigningKey = bob.deriveChild(alicePub, invoiceNumber)
  const bobSignature = bobSigningKey.sign(bobMessage)
  
  // Cross-verification
  const aliceVerifyKey = alicePub.deriveChild(bob, invoiceNumber)
  const bobVerifyKey = bobPub.deriveChild(alice, invoiceNumber)
  
  const aliceVerified = aliceVerifyKey.verify(aliceMessage, aliceSignature)
  const bobVerified = bobVerifyKey.verify(bobMessage, bobSignature)
  
  console.log('Alice message verified:', aliceVerified)
  console.log('Bob message verified:', bobVerified)
  
  return { aliceVerified, bobVerified }
}
```

## The "Anyone Key" Concept

The SDK supports a special "anyone" key concept for scenarios where one party wants to create publicly verifiable signatures. The "anyone" key is simply the private key with value 1:

```typescript
// The "anyone" private key
const anyonePrivateKey = new PrivateKey(1)
const anyonePublicKey = anyonePrivateKey.toPublicKey()

console.log('Anyone public key:', anyonePublicKey.toString())

// Using "anyone" key for public verification
function createPubliclyVerifiableSignature() {
  const signer = PrivateKey.fromRandom()
  const invoiceNumber = '2-public-announcement-001'
  
  // Derive key using "anyone" as counterparty
  const signingKey = signer.deriveChild(anyonePublicKey, invoiceNumber)
  
  // Sign message
  const message = Utils.toArray('This is a public announcement', 'utf8')
  const signature = signingKey.sign(message)
  
  // Anyone can verify using the signer's public key
  const signerPub = signer.toPublicKey()
  const verifyKey = signerPub.deriveChild(anyonePrivateKey, invoiceNumber)
  const verified = verifyKey.verify(message, signature)
  
  console.log('Public signature verified:', verified)
  
  return {
    signerPublicKey: signerPub.toString(),
    signature: signature.toDER('hex'),
    verified
  }
}

createPubliclyVerifiableSignature()
```

## Advanced Type-42 Applications

### Multi-Purpose Key Derivation

Use different invoice numbers for different purposes:

```typescript
function multiPurposeKeyDerivation() {
  const alice = PrivateKey.fromRandom()
  const bob = PrivateKey.fromRandom()
  const bobPub = bob.toPublicKey()
  
  // Different keys for different purposes
  const signingKey = alice.deriveChild(bobPub, '2-signing-001')
  const encryptionKey = alice.deriveChild(bobPub, '2-encryption-001')
  const authKey = alice.deriveChild(bobPub, '2-auth-001')
  
  console.log('Signing key:', signingKey.toHex().substring(0, 16) + '...')
  console.log('Encryption key:', encryptionKey.toHex().substring(0, 16) + '...')
  console.log('Auth key:', authKey.toHex().substring(0, 16) + '...')
  
  // Each key is unique and serves a different purpose
  return { signingKey, encryptionKey, authKey }
}
```

### Session-Based Key Derivation

Create time-based or session-based keys:

```typescript
function sessionBasedKeys() {
  const alice = PrivateKey.fromRandom()
  const bob = PrivateKey.fromRandom()
  const bobPub = bob.toPublicKey()
  
  // Create session-specific keys
  const sessionId = Date.now().toString()
  const sessionInvoice = `2-session-${sessionId}`
  
  const sessionKey = alice.deriveChild(bobPub, sessionInvoice)
  
  console.log('Session ID:', sessionId)
  console.log('Session key created:', sessionKey.toHex().substring(0, 16) + '...')
  
  return { sessionId, sessionKey }
}
```

## Integration with Transactions

Type-42 derived keys can be used in Bitcoin transactions:

```typescript
import { Transaction, P2PKH } from '@bsv/sdk'

async function type42Transaction() {
  const alice = PrivateKey.fromRandom()
  const bob = PrivateKey.fromRandom()
  const bobPub = bob.toPublicKey()
  
  const invoiceNumber = '2-payment-001'
  
  // Alice derives a key for this specific payment
  const paymentKey = alice.deriveChild(bobPub, invoiceNumber)
  const paymentAddress = paymentKey.toAddress()
  
  console.log('Payment address:', paymentAddress)
  
  // Bob can derive the same public key to verify ownership
  const alicePub = alice.toPublicKey()
  const verifyKey = alicePub.deriveChild(bob, invoiceNumber)
  const verifyAddress = verifyKey.toAddress()
  
  console.log('Addresses match:', paymentAddress === verifyAddress)
  
  return { paymentKey, paymentAddress, verifyAddress }
}
```

## Error Handling and Validation

Implement robust error handling for Type-42 operations:

```typescript
function safeType42Derivation(
  privateKey: PrivateKey,
  counterpartyPublicKey: PublicKey,
  invoiceNumber: string
): PrivateKey | null {
  try {
    // Validate inputs
    if (!privateKey || !counterpartyPublicKey || !invoiceNumber) {
      throw new Error('Missing required parameters')
    }
    
    if (invoiceNumber.length === 0) {
      throw new Error('Invoice number cannot be empty')
    }
    
    // Perform derivation
    const derivedKey = privateKey.deriveChild(counterpartyPublicKey, invoiceNumber)
    
    // Validate result
    if (!derivedKey) {
      throw new Error('Key derivation failed')
    }
    
    return derivedKey
  } catch (error: any) {
    console.error('Type-42 derivation error:', error.message)
    return null
  }
}

// Usage example
const alice = PrivateKey.fromRandom()
const bob = PrivateKey.fromRandom()
const bobPub = bob.toPublicKey()

const derivedKey = safeType42Derivation(alice, bobPub, '2-safe-derivation-001')
if (derivedKey) {
  console.log('Derivation successful')
} else {
  console.log('Derivation failed')
}
```

## Performance Considerations

### Caching Derived Keys

For applications that frequently use the same invoice numbers:

```typescript
class Type42KeyCache {
  private cache = new Map<string, PrivateKey>()
  
  constructor(private masterKey: PrivateKey) {}
  
  deriveKey(counterpartyPub: PublicKey, invoiceNumber: string): PrivateKey {
    const cacheKey = `${counterpartyPub.toString()}-${invoiceNumber}`
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!
    }
    
    const derivedKey = this.masterKey.deriveChild(counterpartyPub, invoiceNumber)
    this.cache.set(cacheKey, derivedKey)
    
    return derivedKey
  }
  
  clearCache(): void {
    this.cache.clear()
  }
  
  getCacheSize(): number {
    return this.cache.size
  }
}

// Usage
const alice = PrivateKey.fromRandom()
const keyCache = new Type42KeyCache(alice)

const bob = PrivateKey.fromRandom()
const bobPub = bob.toPublicKey()

// First call performs derivation
const key1 = keyCache.deriveKey(bobPub, '2-cached-001')

// Second call uses cache
const key2 = keyCache.deriveKey(bobPub, '2-cached-001')

console.log('Keys are identical:', key1.toHex() === key2.toHex())
console.log('Cache size:', keyCache.getCacheSize())
```

## Security Best Practices

### 1. Invoice Number Guidelines

```typescript
// Good: Structured, unique invoice numbers
const goodInvoiceNumbers = [
  '2-payment-20241210-001',
  '2-message-session-abc123',
  '2-auth-token-xyz789'
]

// Avoid: Predictable or reused invoice numbers
const badInvoiceNumbers = [
  '1', // Too simple
  'payment', // Not unique
  '2-payment-001' // Reused across different contexts
]
```

### 2. Master Key Protection

```typescript
function secureMasterKeyUsage() {
  // Generate master key securely
  const masterKey = PrivateKey.fromRandom()
  
  // Never log or expose master keys
  // console.log('Master key:', masterKey.toHex()) // DON'T DO THIS
  
  // Use derived keys for operations
  const counterparty = PrivateKey.fromRandom().toPublicKey()
  const derivedKey = masterKey.deriveChild(counterparty, '2-secure-operation-001')
  
  // Log derived keys if needed (they don't reveal master key)
  console.log('Derived key (safe to log):', derivedKey.toHex().substring(0, 16) + '...')
  
  return derivedKey
}
```

### 3. Counterparty Validation

```typescript
function validateCounterparty(publicKey: PublicKey): boolean {
  try {
    // Ensure the public key is valid by checking its coordinates
    const x = publicKey.x
    const y = publicKey.y
    if (!x || !y) {
      return false
    }
    
    // Additional validation can be added here
    return true
  } catch (error: any) {
    return false
  }
}
```

## Testing Type-42 Implementation

```typescript
function testType42Implementation() {
  console.log('Testing Type-42 key derivation...')
  
  // Test 1: Basic derivation
  const alice = PrivateKey.fromRandom()
  const bob = PrivateKey.fromRandom()
  const bobPub = bob.toPublicKey()
  const alicePub = alice.toPublicKey()
  
  const invoiceNumber = '2-test-001'
  
  const aliceChild = alice.deriveChild(bobPub, invoiceNumber)
  const aliceChildPub = alicePub.deriveChild(bob, invoiceNumber)
  
  const test1 = aliceChild.toPublicKey().toString() === aliceChildPub.toString()
  console.log('Test 1 - Key consistency:', test1 ? 'PASS' : 'FAIL')
  
  // Test 2: Message signing and verification
  const message = Utils.toArray('Test message', 'utf8')
  const signature = aliceChild.sign(message)
  const verified = aliceChildPub.verify(message, signature)
  
  console.log('Test 2 - Sign/verify:', verified ? 'PASS' : 'FAIL')
  
  // Test 3: Different invoice numbers produce different keys
  const key1 = alice.deriveChild(bobPub, '2-test-001')
  const key2 = alice.deriveChild(bobPub, '2-test-002')
  
  const test3 = key1.toHex() !== key2.toHex()
  console.log('Test 3 - Unique keys:', test3 ? 'PASS' : 'FAIL')
  
  // Test 4: Bidirectional derivation
  const bobChild = bob.deriveChild(alicePub, invoiceNumber)
  const bobChildPub = bobPub.deriveChild(alice, invoiceNumber)
  
  const test4 = bobChild.toPublicKey().toString() === bobChildPub.toString()
  console.log('Test 4 - Bidirectional:', test4 ? 'PASS' : 'FAIL')
  
  return test1 && verified && test3 && test4
}

// Run tests
const allTestsPassed = testType42Implementation()
console.log('All tests passed:', allTestsPassed)
```

## Troubleshooting Common Issues

### Issue 1: Key Mismatch

```typescript
// Problem: Derived keys don't match
// Solution: Ensure consistent invoice numbers and key order

function debugKeyMismatch() {
  const alice = PrivateKey.fromRandom()
  const bob = PrivateKey.fromRandom()
  
  // Wrong: Different invoice numbers
  const key1 = alice.deriveChild(bob.toPublicKey(), '2-test-001')
  const key2 = alice.toPublicKey().deriveChild(bob, '2-test-002') // Different number
  
  // Correct: Same invoice number
  const key3 = alice.deriveChild(bob.toPublicKey(), '2-test-001')
  const key4 = alice.toPublicKey().deriveChild(bob, '2-test-001') // Same number
  
  console.log('Wrong approach - keys match:', 
    key1.toPublicKey().toString() === key2.toString())
  console.log('Correct approach - keys match:', 
    key3.toPublicKey().toString() === key4.toString())
}
```

### Issue 2: Invalid Public Key

```typescript
function handleInvalidPublicKey() {
  try {
    const alice = PrivateKey.fromRandom()
    // This would cause an error if publicKey is invalid
    const invalidPub = null as any
    
    const derivedKey = alice.deriveChild(invalidPub, '2-test-001')
  } catch (error: any) {
    console.log('Caught invalid public key error:', error.message)
    // Handle the error appropriately
  }
}
```

## Conclusion

Type-42 key derivation provides a powerful mechanism for creating shared key universes between two parties. You've learned how to:

- Generate master keys and derive child keys using invoice numbers
- Implement secure message signing and verification
- Use the "anyone key" for publicly verifiable signatures
- Apply Type-42 in various Bitcoin applications
- Handle errors and optimize performance
- Follow security best practices

Type-42 enables sophisticated cryptographic protocols while maintaining the security properties of elliptic curve cryptography. The derived keys are unlinkable to master keys by outside parties, providing privacy and security for Bitcoin applications.

## Further Reading

- [ECDH Key Exchange Tutorial](./ecdh-key-exchange.md)
- [Elliptic Curve Fundamentals Tutorial](./elliptic-curve-fundamentals.md)
- [BSV Type-42 Documentation](https://docs.bsvblockchain.org/guides/sdks/ts/low-level/type_42)
- [Messages Reference](../reference/messages.md)

## Integration with `WalletClient`

For production applications, the `WalletClient` provides secure key management for Type-42 operations:

```typescript
// Usage example
const walletClient = new WalletClient('https://api.bsvwallet.com/v1')
const alice = walletClient.getPrivateKey('alice')
const bob = walletClient.getPrivateKey('bob')
const invoiceNumber = '2-secure-operation-001'

const derivedKey = alice.deriveChild(bob.toPublicKey(), invoiceNumber)
console.log('Derived key:', derivedKey.toHex().substring(0, 16) + '...')
