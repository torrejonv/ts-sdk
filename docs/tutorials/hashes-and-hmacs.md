# Cryptographic Hashing and HMACs

**Duration**: 75 minutes  
**Prerequisites**: Basic TypeScript knowledge, understanding of cryptographic concepts

## Learning Goals

By completing this tutorial, you will:
- Understand cryptographic hash functions and their properties
- Master the Hash module classes and helper functions in the BSV TypeScript SDK
- Implement various hash algorithms (SHA-256, SHA-512, SHA-1, RIPEMD-160)
- Create and verify HMACs for message authentication
- Apply Bitcoin-specific hashing patterns (hash256, hash160)
- Build practical applications using hashing for data integrity and authentication
- Understand performance considerations and security best practices

## Introduction to Cryptographic Hashing

Cryptographic hash functions are mathematical algorithms that transform input data of any size into fixed-size output values. They are fundamental to Bitcoin's security architecture, providing:

- **Data Integrity**: Detect any changes to data
- **Digital Fingerprints**: Unique identifiers for data
- **Proof of Work**: Foundation for Bitcoin's consensus mechanism
- **Address Generation**: Converting public keys to Bitcoin addresses

The BSV TypeScript SDK provides comprehensive hashing capabilities through the `Hash` module, supporting both class-based and functional approaches.

## Setting Up Your Environment

First, import the necessary modules from the BSV SDK:

```typescript
import { Hash, Utils } from '@bsv/sdk'
```

The `Hash` module contains:
- Hash function classes (`SHA256`, `SHA512`, `SHA1`, `RIPEMD160`)
- HMAC classes (`SHA256HMAC`, `SHA512HMAC`, `SHA1HMAC`)
- Helper functions (`sha256`, `sha512`, `hash256`, `hash160`, `sha256hmac`)
- Utility functions for data conversion and encoding

## Basic Hash Function Usage

### SHA-256 Hashing

SHA-256 is Bitcoin's primary hash function. Here's how to use it:

```typescript
// Method 1: Using the SHA256 class
const sha256Hasher = new Hash.SHA256()
sha256Hasher.update('Message to hash')
const hashedMessage = sha256Hasher.digestHex()
console.log('SHA-256 hash:', hashedMessage)
// Output: f1aa45b0f5f6703468f9b9bc2b9874d4fa6b001a170d0f132aa5a26d00d0c7e5

// Method 2: Using the helper function
const message = 'Hello, Bitcoin!'
const hashResult = Hash.sha256(Utils.toArray(message, 'utf8'))
console.log('SHA-256 hash (binary):', hashResult)

// Convert to hex for display
const hashHex = Utils.toHex(hashResult)
console.log('SHA-256 hash (hex):', hashHex)
```

### Working with Different Data Types

The Hash functions can process various data formats:

```typescript
// String input
const stringHash = Hash.sha256(Utils.toArray('Hello World', 'utf8'))

// Hex string input
const hexHash = Hash.sha256(Utils.toArray('deadbeef', 'hex'))

// Binary array input
const binaryData = [0x01, 0x02, 0x03, 0x04]
const binaryHash = Hash.sha256(binaryData)

// JSON data hashing
const jsonData = { name: 'Alice', amount: 100 }
const jsonString = JSON.stringify(jsonData)
const jsonHash = Hash.sha256(Utils.toArray(jsonString, 'utf8'))

console.log('String hash:', Utils.toHex(stringHash))
console.log('Hex hash:', Utils.toHex(hexHash))
console.log('Binary hash:', Utils.toHex(binaryHash))
console.log('JSON hash:', Utils.toHex(jsonHash))
```

### Other Hash Algorithms

The SDK supports multiple hash algorithms:

```typescript
// SHA-512
const sha512Hasher = new Hash.SHA512()
sha512Hasher.update('Message for SHA-512')
const sha512Result = sha512Hasher.digestHex()
console.log('SHA-512 hash:', sha512Result)

// SHA-1 (legacy, use with caution)
const sha1Hasher = new Hash.SHA1()
sha1Hasher.update('Message for SHA-1')
const sha1Result = sha1Hasher.digestHex()
console.log('SHA-1 hash:', sha1Result)

// RIPEMD-160 (used in Bitcoin address generation)
const ripemdHasher = new Hash.RIPEMD160()
ripemdHasher.update('Message for RIPEMD-160')
const ripemdResult = ripemdHasher.digestHex()
console.log('RIPEMD-160 hash:', ripemdResult)

// Using helper functions
const sha512Helper = Hash.sha512(Utils.toArray('Hello', 'utf8'))
console.log('SHA-512 helper result:', Utils.toHex(sha512Helper))
```

## Bitcoin-Specific Hash Functions

### Double SHA-256 (hash256)

Bitcoin uses double SHA-256 hashing for block headers and transaction IDs:

```typescript
// Double SHA-256 using hash256 helper
const message = 'Bitcoin transaction data'
const doubleHash = Hash.hash256(Utils.toArray(message, 'utf8'))
console.log('Double SHA-256 hash:', Utils.toHex(doubleHash))

// Manual double hashing
const firstHash = Hash.sha256(Utils.toArray(message, 'utf8'))
const secondHash = Hash.sha256(firstHash)
console.log('Manual double hash:', Utils.toHex(secondHash))

// Both methods produce the same result
console.log('Results match:', Utils.toHex(doubleHash) === Utils.toHex(secondHash))
```

### Hash160 (SHA-256 + RIPEMD-160)

Used for Bitcoin address generation:

```typescript
// Hash160: SHA-256 followed by RIPEMD-160
const publicKeyData = 'compressed_public_key_hex_data'
const hash160Result = Hash.hash160(Utils.toArray(publicKeyData, 'hex'))
console.log('Hash160 result:', Utils.toHex(hash160Result))

// Manual implementation
const sha256First = Hash.sha256(Utils.toArray(publicKeyData, 'hex'))
const ripemd160Second = new Hash.RIPEMD160().update(sha256First).digest()
console.log('Manual Hash160:', Utils.toHex(ripemd160Second))

// Results should match
console.log('Hash160 results match:', 
  Utils.toHex(hash160Result) === Utils.toHex(ripemd160Second))
```

## HMAC Implementation

HMACs (Hash-based Message Authentication Codes) provide both data integrity and authentication by incorporating a secret key into the hashing process.

### Basic HMAC Usage

```typescript
// SHA-256 HMAC
const key = 'secret_key'
const message = 'Message to authenticate'

// Method 1: Using HMAC class
const hmacHasher = new Hash.SHA256HMAC(key)
hmacHasher.update(message)
const hmacResult = hmacHasher.digestHex()
console.log('HMAC-SHA256:', hmacResult)
// Output: b4d897472c73a052733d0796a5f71cf8253bab7d3969811b64f41ff6aa89d86f

// Method 2: Using helper function
const hmacHelper = Hash.sha256hmac(key, message)
console.log('HMAC helper result:', Utils.toHex(hmacHelper))

// Both methods produce the same result
console.log('HMAC results match:', hmacResult === Utils.toHex(hmacHelper))
```

### HMAC with Different Algorithms

```typescript
// SHA-512 HMAC
const sha512Hmac = new Hash.SHA512HMAC('my_secret_key')
sha512Hmac.update('Data to authenticate')
const sha512HmacResult = sha512Hmac.digestHex()
console.log('HMAC-SHA512:', sha512HmacResult)

// SHA-1 HMAC (legacy)
const sha1Hmac = new Hash.SHA1HMAC('legacy_key')
sha1Hmac.update('Legacy data')
const sha1HmacResult = sha1Hmac.digestHex()
console.log('HMAC-SHA1:', sha1HmacResult)
```

### HMAC Key Management

```typescript
// Strong key generation
function generateHmacKey(): string {
  const randomBytes = new Array(32)
  for (let i = 0; i < 32; i++) {
    randomBytes[i] = Math.floor(Math.random() * 256)
  }
  return Utils.toHex(randomBytes)
}

// Key derivation from password
function deriveKeyFromPassword(password: string, salt: string): number[] {
  const combined = password + salt
  return Hash.sha256(Utils.toArray(combined, 'utf8'))
}

// Example usage
const strongKey = generateHmacKey()
const derivedKey = deriveKeyFromPassword('user_password', 'random_salt')

console.log('Strong key:', strongKey)
console.log('Derived key:', Utils.toHex(derivedKey))

// Use derived key for HMAC
const secureHmac = Hash.sha256hmac(derivedKey, 'sensitive_data')
console.log('Secure HMAC:', Utils.toHex(secureHmac))
```

## Practical Applications

### Data Integrity Verification

```typescript
class DataIntegrityChecker {
  private data: string
  private hash: string

  constructor(data: string) {
    this.data = data
    this.hash = this.calculateHash(data)
  }

  private calculateHash(data: string): string {
    const hashResult = Hash.sha256(Utils.toArray(data, 'utf8'))
    return Utils.toHex(hashResult)
  }

  verify(): boolean {
    const currentHash = this.calculateHash(this.data)
    return currentHash === this.hash
  }

  getData(): string {
    return this.data
  }

  getHash(): string {
    return this.hash
  }

  // Simulate data corruption
  corruptData(): void {
    this.data += '_corrupted'
  }
}

// Example usage
const checker = new DataIntegrityChecker('Important document content')
console.log('Original hash:', checker.getHash())
console.log('Data is valid:', checker.verify()) // true

checker.corruptData()
console.log('After corruption, data is valid:', checker.verify()) // false
```

### Message Authentication System

```typescript
class MessageAuthenticator {
  private secretKey: string

  constructor(secretKey: string) {
    this.secretKey = secretKey
  }

  createAuthenticatedMessage(message: string): {
    message: string
    hmac: string
    timestamp: number
  } {
    const timestamp = Date.now()
    const messageWithTimestamp = `${message}:${timestamp}`
    const hmac = Hash.sha256hmac(this.secretKey, messageWithTimestamp)
    
    return {
      message,
      hmac: Utils.toHex(hmac),
      timestamp
    }
  }

  verifyMessage(authenticatedMessage: {
    message: string
    hmac: string
    timestamp: number
  }): boolean {
    try {
      const messageWithTimestamp = `${authenticatedMessage.message}:${authenticatedMessage.timestamp}`
      const expectedHmac = Hash.sha256hmac(this.secretKey, messageWithTimestamp)
      const expectedHmacHex = Utils.toHex(expectedHmac)
      
      // Constant-time comparison to prevent timing attacks
      return this.constantTimeCompare(authenticatedMessage.hmac, expectedHmacHex)
    } catch (error) {
      console.error('Verification error:', error)
      return false
    }
  }

  private constantTimeCompare(a: string, b: string): boolean {
    if (a.length !== b.length) {
      return false
    }
    
    let result = 0
    for (let i = 0; i < a.length; i++) {
      result |= a.charCodeAt(i) ^ b.charCodeAt(i)
    }
    
    return result === 0
  }
}

// Example usage
const authenticator = new MessageAuthenticator('super_secret_key_123')

const authMessage = authenticator.createAuthenticatedMessage('Transfer 100 satoshis to Alice')
console.log('Authenticated message:', authMessage)

const isValid = authenticator.verifyMessage(authMessage)
console.log('Message is authentic:', isValid) // true

// Tamper with the message
authMessage.message = 'Transfer 1000 satoshis to Alice'
const isTamperedValid = authenticator.verifyMessage(authMessage)
console.log('Tampered message is authentic:', isTamperedValid) // false
```

### Transaction Metadata Protection

```typescript
interface TransactionMetadata {
  description: string
  category: string
  tags: string[]
  amount: number
}

class SecureTransactionMetadata {
  private key: number[]

  constructor(password: string) {
    // Derive key from password
    this.key = Hash.sha256(Utils.toArray(password, 'utf8'))
  }

  protectMetadata(metadata: TransactionMetadata): {
    data: string
    integrity: string
  } {
    const jsonData = JSON.stringify(metadata)
    const dataBytes = Utils.toArray(jsonData, 'utf8')
    
    // Create integrity hash
    const integrity = Hash.sha256hmac(this.key, dataBytes)
    
    return {
      data: Utils.toBase64(dataBytes),
      integrity: Utils.toHex(integrity)
    }
  }

  verifyAndExtract(protectedData: {
    data: string
    integrity: string
  }): TransactionMetadata | null {
    try {
      const dataBytes = Array.from(Buffer.from(protectedData.data, 'base64'))
      const expectedIntegrity = Hash.sha256hmac(this.key, dataBytes)
      const expectedIntegrityHex = Utils.toHex(expectedIntegrity)
      
      if (protectedData.integrity !== expectedIntegrityHex) {
        console.error('Integrity check failed')
        return null
      }
      
      const jsonString = Utils.toUTF8(dataBytes)
      return JSON.parse(jsonString) as TransactionMetadata
    } catch (error) {
      console.error('Extraction error:', error)
      return null
    }
  }
}

// Example usage
const metadataProtector = new SecureTransactionMetadata('user_password_123')

const originalMetadata: TransactionMetadata = {
  description: 'Payment for services',
  category: 'business',
  tags: ['consulting', 'development'],
  amount: 100
}

const protectedData = metadataProtector.protectMetadata(originalMetadata)
console.log('Protected metadata:', protectedData)

const extracted = metadataProtector.verifyAndExtract(protectedData)
console.log('Extracted metadata:', extracted)
console.log('Metadata matches:', JSON.stringify(originalMetadata) === JSON.stringify(extracted))
```

## Performance Optimization

### Batch Hashing

```typescript
class BatchHashProcessor {
  private hasher: Hash.SHA256

  constructor() {
    this.hasher = new Hash.SHA256()
  }

  hashMultipleMessages(messages: string[]): string[] {
    const results: string[] = []
    
    for (const message of messages) {
      // Reset hasher for each message
      this.hasher = new Hash.SHA256()
      this.hasher.update(message)
      results.push(this.hasher.digestHex())
    }
    
    return results
  }

  createMerkleRoot(hashes: string[]): string {
    if (hashes.length === 0) {
      throw new Error('Cannot create merkle root from empty array')
    }
    
    if (hashes.length === 1) {
      return hashes[0]
    }
    
    const nextLevel: string[] = []
    
    for (let i = 0; i < hashes.length; i += 2) {
      const left = hashes[i]
      const right = i + 1 < hashes.length ? hashes[i + 1] : left
      
      const combined = left + right
      const combinedBytes = Utils.toArray(combined, 'hex')
      const hash = Hash.sha256(combinedBytes)
      nextLevel.push(Utils.toHex(hash))
    }
    
    return this.createMerkleRoot(nextLevel)
  }
}

// Performance testing
function performanceTest() {
  const processor = new BatchHashProcessor()
  const testMessages = Array.from({ length: 1000 }, (_, i) => `Message ${i}`)
  
  console.time('Batch hashing 1000 messages')
  const hashes = processor.hashMultipleMessages(testMessages)
  console.timeEnd('Batch hashing 1000 messages')
  
  console.time('Creating merkle root')
  const merkleRoot = processor.createMerkleRoot(hashes)
  console.timeEnd('Creating merkle root')
  
  console.log('Merkle root:', merkleRoot)
  console.log('Processed', hashes.length, 'messages')
}

performanceTest()
```

### Memory-Efficient Streaming

```typescript
class StreamingHasher {
  private hasher: Hash.SHA256

  constructor() {
    this.hasher = new Hash.SHA256()
  }

  processLargeData(data: string, chunkSize: number = 1024): string {
    for (let i = 0; i < data.length; i += chunkSize) {
      const chunk = data.slice(i, i + chunkSize)
      this.hasher.update(chunk)
    }
    
    return this.hasher.digestHex()
  }

  reset(): void {
    this.hasher = new Hash.SHA256()
  }
}

// Example with large data
const streamingHasher = new StreamingHasher()
const largeData = 'A'.repeat(1000000) // 1MB of data
const streamHash = streamingHasher.processLargeData(largeData)
console.log('Streaming hash result:', streamHash)
```

## Security Best Practices

### Secure Key Generation and Storage

```typescript
class SecureKeyManager {
  static generateSecureKey(length: number = 32): number[] {
    // In production, use a cryptographically secure random number generator
    const key = new Array(length)
    for (let i = 0; i < length; i++) {
      key[i] = Math.floor(Math.random() * 256)
    }
    return key
  }

  static deriveKeyFromPassword(
    password: string, 
    salt: number[], 
    iterations: number = 10000
  ): number[] {
    let derived = Hash.sha256(Utils.toArray(password + Utils.toHex(salt), 'utf8'))
    
    for (let i = 1; i < iterations; i++) {
      derived = Hash.sha256(derived)
    }
    
    return derived
  }

  static secureCompare(a: number[], b: number[]): boolean {
    if (a.length !== b.length) {
      return false
    }
    
    let result = 0
    for (let i = 0; i < a.length; i++) {
      result |= a[i] ^ b[i]
    }
    
    return result === 0
  }

  static clearSensitiveData(data: number[]): void {
    for (let i = 0; i < data.length; i++) {
      data[i] = 0
    }
  }
}

// Example secure usage
const salt = SecureKeyManager.generateSecureKey(16)
const derivedKey = SecureKeyManager.deriveKeyFromPassword('user_password', salt)

console.log('Salt:', Utils.toHex(salt))
console.log('Derived key:', Utils.toHex(derivedKey))

// Clear sensitive data when done
SecureKeyManager.clearSensitiveData(derivedKey)
SecureKeyManager.clearSensitiveData(salt)
```

### Input Validation and Error Handling

```typescript
class SafeHasher {
  static validateInput(input: any): void {
    if (input === null || input === undefined) {
      throw new Error('Input cannot be null or undefined')
    }
    
    if (typeof input === 'string' && input.length === 0) {
      throw new Error('Input string cannot be empty')
    }
    
    if (Array.isArray(input) && input.length === 0) {
      throw new Error('Input array cannot be empty')
    }
  }

  static safeHash(input: string | number[], algorithm: 'sha256' | 'sha512' = 'sha256'): string {
    try {
      this.validateInput(input)
      
      let data: number[]
      if (typeof input === 'string') {
        data = Utils.toArray(input, 'utf8')
      } else {
        data = input
      }
      
      let result: number[]
      switch (algorithm) {
        case 'sha256':
          result = Hash.sha256(data)
          break
        case 'sha512':
          result = Hash.sha512(data)
          break
        default:
          throw new Error(`Unsupported algorithm: ${algorithm}`)
      }
      
      return Utils.toHex(result)
    } catch (error) {
      console.error('Hashing error:', error)
      throw new Error(`Failed to hash input: ${error instanceof Error ? error.message : 'Unknown error'}`)
    }
  }

  static safeHmac(key: string | number[], message: string | number[]): string {
    try {
      this.validateInput(key)
      this.validateInput(message)
      
      const result = Hash.sha256hmac(key, message)
      return Utils.toHex(result)
    } catch (error) {
      console.error('HMAC error:', error)
      throw new Error(`Failed to create HMAC: ${error instanceof Error ? error.message : 'Unknown error'}`)
    }
  }
}

// Example safe usage
try {
  const hash = SafeHasher.safeHash('Valid input')
  console.log('Safe hash:', hash)
  
  const hmac = SafeHasher.safeHmac('secret_key', 'message')
  console.log('Safe HMAC:', hmac)
} catch (error) {
  console.error('Operation failed:', error)
}
```

## Testing Your Implementation

```typescript
// Comprehensive test suite
function runHashTests(): void {
  console.log('Running hash function tests...')
  
  // Test SHA-256 consistency
  const testMessage = 'Hello, Bitcoin!'
  const hash1 = Hash.sha256(Utils.toArray(testMessage, 'utf8'))
  const hash2 = new Hash.SHA256().update(testMessage).digest()
  
  console.assert(
    Utils.toHex(hash1) === Utils.toHex(hash2),
    'SHA-256 methods should produce same result'
  )
  
  // Test HMAC consistency
  const key = 'test_key'
  const message = 'test_message'
  const hmac1 = Hash.sha256hmac(key, message)
  const hmac2 = new Hash.SHA256HMAC(key).update(message).digest()
  
  console.assert(
    Utils.toHex(hmac1) === Utils.toHex(hmac2),
    'HMAC methods should produce same result'
  )
  
  // Test Bitcoin-specific functions
  const data = 'bitcoin_data'
  const hash256Result = Hash.hash256(Utils.toArray(data, 'utf8'))
  const manualDouble = Hash.sha256(Hash.sha256(Utils.toArray(data, 'utf8')))
  
  console.assert(
    Utils.toHex(hash256Result) === Utils.toHex(manualDouble),
    'hash256 should equal double SHA-256'
  )
  
  console.log('All tests passed!')
}

runHashTests()
```

## Troubleshooting Common Issues

### Issue 1: Encoding Problems

```typescript
// Problem: Incorrect encoding leads to different hashes
// Note: Hash.sha256() requires number[] input, not strings directly
const correctHash = Hash.sha256(Utils.toArray('hello', 'utf8')) // Correct approach

console.log('Correct hash with proper encoding:', Utils.toHex(correctHash))
console.log('Always use Utils.toArray() for proper encoding')
```

### Issue 2: Key Management

```typescript
// Problem: Weak keys or improper key derivation
const weakKey = 'password123' // Weak
const strongKey = SecureKeyManager.generateSecureKey() // Strong

// Problem: Reusing keys across different purposes
const hmacKey = strongKey
const encryptionKey = strongKey // Wrong: same key for different purposes

// Solution: Derive different keys for different purposes
const hmacKeyDerived = Hash.sha256(Utils.toArray('hmac:' + Utils.toHex(strongKey), 'utf8'))
const encryptionKeyDerived = Hash.sha256(Utils.toArray('encrypt:' + Utils.toHex(strongKey), 'utf8'))
```

### Issue 3: Performance Problems

```typescript
// Problem: Creating new hasher instances unnecessarily
function inefficientHashing(messages: string[]): string[] {
  return messages.map(msg => {
    const hasher = new Hash.SHA256() // Inefficient: new instance each time
    return hasher.update(msg).digestHex()
  })
}

// Solution: Reuse hasher or use helper functions
function efficientHashing(messages: string[]): string[] {
  return messages.map(msg => {
    return Utils.toHex(Hash.sha256(Utils.toArray(msg, 'utf8'))) // Efficient
  })
}
```

## Summary

This tutorial covered comprehensive usage of cryptographic hashing and HMACs in the BSV TypeScript SDK:

**Key Concepts Learned:**
- Hash function fundamentals and Bitcoin-specific applications
- SHA-256, SHA-512, SHA-1, and RIPEMD-160 implementation
- HMAC creation and verification for message authentication
- Bitcoin-specific functions: hash256 (double SHA-256) and hash160
- Performance optimization techniques for batch processing
- Security best practices for key management and validation

**Practical Applications:**
- Data integrity verification systems
- Message authentication protocols
- Transaction metadata protection
- Merkle tree construction
- Secure key derivation patterns

**Security Considerations:**
- Proper input validation and error handling
- Constant-time comparison to prevent timing attacks
- Secure key generation and storage practices
- Memory management for sensitive data

The Hash module in the BSV TypeScript SDK provides both low-level control through classes and high-level convenience through helper functions, enabling developers to implement robust cryptographic solutions for Bitcoin applications.

Continue exploring advanced cryptographic topics with the [ECDH Key Exchange](./ecdh-key-exchange.md) and [AES Symmetric Encryption](./aes-encryption.md) tutorials to build complete cryptographic systems.

```typescript
// Example usage
const metadataProtector = new SecureTransactionMetadata('user_password_123')

const originalMetadata: TransactionMetadata = {
  description: 'Payment for services',
  category: 'business',
  tags: ['consulting', 'development'],
  amount: 100
}

const protectedData = metadataProtector.protectMetadata(originalMetadata)
console.log('Protected metadata:', protectedData)

const extracted = metadataProtector.verifyAndExtract(protectedData)
console.log('Extracted metadata:', extracted)
console.log('Metadata matches:', JSON.stringify(originalMetadata) === JSON.stringify(extracted))
```

Follow these instructions to make the following change to my code document.

Instruction: Update all occurrences of "WalletClient" to be formatted as inline code `WalletClient`, except in headers and when it appears in actual code blocks.

Code Edit:
```typescript
// Understanding of `WalletClient` usage (for practical applications)
While the `WalletClient` provides high-level transaction operations, understanding hashes and HMACs enables you to build sophisticated cryptographic applications and secure data verification systems.

## Integration with `WalletClient`
For production applications, the `WalletClient` provides secure hash operations:
