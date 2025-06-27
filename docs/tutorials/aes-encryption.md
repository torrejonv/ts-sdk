# AES Symmetric Encryption

**Duration**: 60 minutes  
**Prerequisites**: Basic TypeScript knowledge, [First Transaction](./first-transaction.md) tutorial completed

## Learning Goals

- Understand AES-GCM symmetric encryption principles
- Use the `SymmetricKey` class for encryption and decryption
- Implement secure key generation and management
- Apply AES encryption in practical Bitcoin applications
- Combine AES with ECDH for secure communication
- Handle different data formats and encoding

## Introduction to AES Encryption

Advanced Encryption Standard (AES) is a symmetric encryption algorithm where the same key is used for both encryption and decryption. The BSV TypeScript SDK provides the `SymmetricKey` class that implements AES-GCM (Galois/Counter Mode), which provides both confidentiality and authenticity.

AES-GCM offers several advantages:
- **Confidentiality**: Data is encrypted and unreadable without the key
- **Authenticity**: Built-in authentication prevents tampering
- **Performance**: Fast encryption/decryption operations
- **Security**: Resistant to various cryptographic attacks

## Setting Up Your Environment

First, let's import the necessary classes from the SDK:

```typescript
import { SymmetricKey, Utils, Random } from '@bsv/sdk'

// Helper function to convert hex string to byte array
function hexToBytes(hex: string): number[] {
  const bytes = []
  for (let i = 0; i < hex.length; i += 2) {
    bytes.push(parseInt(hex.substr(i, 2), 16))
  }
  return bytes
}

// Helper function to convert byte array to hex string
function bytesToHex(bytes: number[]): string {
  const hex = []
  for (const byte of bytes) {
    hex.push(byte.toString(16).padStart(2, '0'))
  }
  return hex.join('')
}
```

## Basic AES Encryption

### Generating Encryption Keys

The `SymmetricKey` class provides methods to create secure encryption keys:

```typescript
// Generate a random 256-bit AES key
const symmetricKey = SymmetricKey.fromRandom()
console.log('Generated key:', symmetricKey.toHex())

// Create a key from existing data (32 bytes)
const keyData = Random(32)
const customKey = new SymmetricKey(keyData)
console.log('Custom key:', customKey.toHex())

// Create a key from hex string
const hexKey = 'a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456'
const keyFromHex = new SymmetricKey(hexToBytes(hexKey))
```

### Encrypting Data

The `encrypt` method supports both string and binary data:

```typescript
const symmetricKey = SymmetricKey.fromRandom()
const message = 'Hello, this is a secret message!'

// Encrypt a string (UTF-8 encoding)
const encryptedMessage = symmetricKey.encrypt(message) as number[]
console.log('Encrypted message length:', encryptedMessage.length)

// Encrypt binary data
const binaryData = Utils.toArray('Binary data example', 'utf8')
const encryptedBinary = symmetricKey.encrypt(binaryData) as number[]

// Encrypt with hex output (using manual conversion for clarity)
const encryptedBytes = symmetricKey.encrypt(message) as number[]
const hexEncrypted = bytesToHex(encryptedBytes)
console.log('Hex encrypted:', hexEncrypted)

// Alternative: SDK's built-in hex handling (for hex input data)
// Note: The 'hex' parameter treats the input as hex, not the output format
const messageAsHex = Buffer.from(message).toString('hex') // Convert message to hex first
const sdkHexEncrypted = symmetricKey.encrypt(messageAsHex, 'hex') as string
console.log('SDK hex encrypted:', sdkHexEncrypted)
```

### Decrypting Data

The `decrypt` method reverses the encryption process:

```typescript
// Decrypt to string (UTF-8)
const decryptedMessage = symmetricKey.decrypt(encryptedMessage, 'utf8') as string
console.log('Decrypted message:', decryptedMessage)

// Decrypt to binary array
const decryptedBinary = symmetricKey.decrypt(encryptedBinary) as number[]
console.log('Decrypted binary:', Utils.toUTF8(decryptedBinary))

// Decrypt hex-encoded data (manual conversion method)
const hexBytes = hexToBytes(hexEncrypted)
const decryptedFromHex = symmetricKey.decrypt(hexBytes, 'utf8') as string
console.log('Decrypted from hex:', decryptedFromHex)

// Alternative: SDK hex handling method
const sdkHexBytes = hexToBytes(sdkHexEncrypted)
const sdkDecryptedHex = symmetricKey.decrypt(sdkHexBytes, 'hex') as string
const sdkDecryptedMessage = Buffer.from(sdkDecryptedHex, 'hex').toString('utf8')
console.log('SDK decrypted from hex:', sdkDecryptedMessage)
```

### Understanding the Hex Parameter

The `enc` parameter in the SDK's `encrypt()` and `decrypt()` methods can be confusing. Here's how it actually works:

**In `encrypt(data, enc)`:**
- `enc` specifies how to **interpret the input data**
- `enc: 'hex'` means the input data is a hex string that should be converted to bytes
- The output format is determined by the `enc` parameter (hex string if `enc: 'hex'`, byte array otherwise)

**In `decrypt(data, enc)`:**
- `enc` specifies the **output format**
- `enc: 'hex'` returns the decrypted data as a hex string
- `enc: 'utf8'` returns the decrypted data as a UTF-8 string

```typescript
// Example: Encrypting hex data with SDK's built-in hex handling
const message = 'Hello, World!'
const messageAsHex = Buffer.from(message).toString('hex') // '48656c6c6f2c20576f726c6421'

// Encrypt hex input data
const encrypted = key.encrypt(messageAsHex, 'hex') as string

// Decrypt and get result as hex
const decryptedHex = key.decrypt(hexToBytes(encrypted), 'hex') as string
console.log('Decrypted as hex:', decryptedHex) // '48656c6c6f2c20576f726c6421'

// Convert hex result back to UTF-8
const finalMessage = Buffer.from(decryptedHex, 'hex').toString('utf8')
console.log('Final message:', finalMessage) // 'Hello, World!'
```


## Complete Encryption Example

Here's a comprehensive example demonstrating the full encryption workflow:

```typescript
import { SymmetricKey, Utils } from '@bsv/sdk'

function demonstrateAESEncryption() {
  console.log('=== AES Encryption Demonstration ===')
  
  // 1. Generate a random encryption key
  const symmetricKey = SymmetricKey.fromRandom()
  console.log('Generated key:', symmetricKey.toHex().substring(0, 16) + '...')
  
  // 2. Prepare the message to encrypt
  const originalMessage = 'This is a confidential message that needs protection!'
  console.log('Original message:', originalMessage)
  
  // 3. Encrypt the message
  const encryptedData = symmetricKey.encrypt(originalMessage) as number[]
  console.log('Encrypted data length:', encryptedData.length, 'bytes')
  console.log('Encrypted (first 32 bytes):', encryptedData.slice(0, 32))
  
  // 4. Decrypt the message
  const decryptedMessage = symmetricKey.decrypt(encryptedData, 'utf8') as string
  
  // 5. Verify integrity
  const isValid = originalMessage === decryptedMessage
  console.log('Decryption successful:', isValid)
  
  return {
    key: symmetricKey,
    original: originalMessage,
    encrypted: encryptedData,
    decrypted: decryptedMessage,
    valid: isValid
  }
}

// Run the demonstration
const result = demonstrateAESEncryption()
```

## Working with Different Data Types

### Encrypting JSON Data

```typescript
function encryptJSON(data: any, key: SymmetricKey): number[] {
  const jsonString = JSON.stringify(data)
  return key.encrypt(jsonString) as number[]
}

function decryptJSON(encryptedData: number[], key: SymmetricKey): any {
  const jsonString = key.decrypt(encryptedData, 'utf8') as string
  return JSON.parse(jsonString)
}

// Example usage
const symmetricKey = SymmetricKey.fromRandom()
const userData = {
  name: 'Alice',
  email: 'alice@example.com',
  balance: 1000,
  transactions: ['tx1', 'tx2', 'tx3']
}

const encryptedJSON = encryptJSON(userData, symmetricKey)
const decryptedData = decryptJSON(encryptedJSON, symmetricKey)
console.log('Original data:', userData)
console.log('Decrypted data:', decryptedData)
```

### Encrypting Files and Large Data

```typescript
function encryptLargeData(data: string, key: SymmetricKey): {
  encrypted: number[],
  size: number,
  checksum: string
} {
  // Convert to binary
  const binaryData = Utils.toArray(data, 'utf8')
  
  // Encrypt the data
  const encrypted = key.encrypt(binaryData) as number[]
  
  // Calculate checksum of original data for verification
  const checksum = Utils.toBase64(binaryData.slice(0, 32))
  
  return {
    encrypted,
    size: binaryData.length,
    checksum
  }
}

function decryptLargeData(encryptedData: {
  encrypted: number[],
  size: number,
  checksum: string
}, key: SymmetricKey): string {
  // Decrypt the data
  const decrypted = key.decrypt(encryptedData.encrypted) as number[]
  
  // Verify size
  if (decrypted.length !== encryptedData.size) {
    throw new Error('Decrypted data size mismatch')
  }
  
  // Verify checksum
  const checksum = Utils.toBase64(decrypted.slice(0, 32))
  if (checksum !== encryptedData.checksum) {
    console.warn('Checksum mismatch - data may be corrupted')
  }
  
  return Utils.toUTF8(decrypted)
}
```

## Key Derivation and Management

### Deriving Keys from Passwords

```typescript
import { Hash } from '@bsv/sdk'

function deriveKeyFromPassword(password: string, salt?: number[]): SymmetricKey {
  // Use provided salt or generate random one
  const keySalt = salt || Random(32)
  
  // Create key material by hashing password + salt
  const passwordBytes = Utils.toArray(password, 'utf8')
  const keyMaterial = [...passwordBytes, ...keySalt]
  
  // Hash to create 256-bit key
  const keyHash = Hash.sha256(keyMaterial)
  
  return new SymmetricKey(keyHash)
}

// Example usage
const password = 'MySecurePassword123!'
const salt = Random(32)
const derivedKey = deriveKeyFromPassword(password, salt)

// Store salt separately - needed for key recreation
console.log('Derived key:', derivedKey.toHex())
console.log('Salt (store this):', Utils.toBase64(salt))

// Recreate the same key later
const recreatedKey = deriveKeyFromPassword(password, salt)
console.log('Keys match:', derivedKey.toHex() === recreatedKey.toHex())
```

### Key Rotation and Versioning

```typescript
class KeyManager {
  private keys: Map<number, SymmetricKey> = new Map()
  private currentVersion: number = 1
  
  generateNewKey(): number {
    const newKey = SymmetricKey.fromRandom()
    this.keys.set(this.currentVersion, newKey)
    return this.currentVersion++
  }
  
  encrypt(data: string, version?: number): { 
    encrypted: number[], 
    version: number 
  } {
    const keyVersion = version || this.currentVersion - 1
    const key = this.keys.get(keyVersion)
    
    if (!key) {
      throw new Error(`Key version ${keyVersion} not found`)
    }
    
    return {
      encrypted: key.encrypt(data) as number[],
      version: keyVersion
    }
  }
  
  decrypt(encryptedData: {
    encrypted: number[], 
    version: number 
  }): string {
    const key = this.keys.get(encryptedData.version)
    
    if (!key) {
      throw new Error(`Key version ${encryptedData.version} not found`)
    }
    
    return key.decrypt(encryptedData.encrypted, 'utf8') as string
  }
}

// Example usage
const keyManager = new KeyManager()
const v1 = keyManager.generateNewKey()
const v2 = keyManager.generateNewKey()

const message = 'Data encrypted with version 1'
const encrypted = keyManager.encrypt(message, v1)
const decrypted = keyManager.decrypt(encrypted)
console.log('Decrypted:', decrypted)
```

## Combining AES with ECDH

For secure communication between parties, combine AES encryption with ECDH key exchange:

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

function createSecureChannel(
  senderPrivateKey: PrivateKey,
  recipientPublicKey: PublicKey
): SymmetricKey {
  // Derive shared secret using ECDH
  const sharedSecret = senderPrivateKey.deriveSharedSecret(recipientPublicKey)
  
  // Use shared secret as AES key material
  const keyMaterial = sharedSecret.encode(true).slice(1) // Remove prefix byte
  
  return new SymmetricKey(keyMaterial)
}

function secureMessageExchange() {
  // Generate key pairs for Alice and Bob
  const alicePrivate = PrivateKey.fromRandom()
  const alicePublic = alicePrivate.toPublicKey()
  
  const bobPrivate = PrivateKey.fromRandom()
  const bobPublic = bobPrivate.toPublicKey()
  
  // Alice creates encryption key using Bob's public key
  const aliceEncryptionKey = createSecureChannel(alicePrivate, bobPublic)
  
  // Bob creates the same encryption key using Alice's public key
  const bobDecryptionKey = createSecureChannel(bobPrivate, alicePublic)
  
  // Verify both parties have the same key
  console.log('Keys match:', 
    aliceEncryptionKey.toHex() === bobDecryptionKey.toHex())
  
  // Alice encrypts a message
  const message = 'Hello Bob, this is a secure message from Alice!'
  const encrypted = aliceEncryptionKey.encrypt(message) as number[]
  
  // Bob decrypts the message
  const decrypted = bobDecryptionKey.decrypt(encrypted, 'utf8') as string
  
  console.log('Original message:', message)
  console.log('Decrypted message:', decrypted)
  console.log('Secure communication successful:', message === decrypted)
  
  return { aliceEncryptionKey, bobDecryptionKey, message, decrypted }
}

// Run secure message exchange
secureMessageExchange()
```

## Error Handling and Security

### Robust Encryption with Error Handling

```typescript
class SecureAESManager {
  private key: SymmetricKey
  
  constructor(key?: SymmetricKey) {
    this.key = key || SymmetricKey.fromRandom()
  }
  
  safeEncrypt(data: string): { 
    success: boolean, 
    encrypted?: number[], 
    error?: string 
  } {
    try {
      if (!data || data.length === 0) {
        return { success: false, error: 'Data cannot be empty' }
      }
      
      const encrypted = this.key.encrypt(data) as number[]
      
      if (!encrypted || encrypted.length === 0) {
        return { success: false, error: 'Encryption failed' }
      }
      
      return { success: true, encrypted }
    } catch (error) {
      return { 
        success: false, 
        error: `Encryption error: ${error.message}` 
      }
    }
  }
  
  safeDecrypt(encryptedData: number[]): { 
    success: boolean, 
    decrypted?: string, 
    error?: string 
  } {
    try {
      if (!encryptedData || encryptedData.length === 0) {
        return { success: false, error: 'Encrypted data cannot be empty' }
      }
      
      const decrypted = this.key.decrypt(encryptedData, 'utf8') as string
      
      return { success: true, decrypted }
    } catch (error) {
      return { 
        success: false, 
        error: `Decryption failed: ${error.message}` 
      }
    }
  }
  
  rotateKey(): void {
    this.key = SymmetricKey.fromRandom()
  }
  
  getKeyFingerprint(): string {
    // Create a fingerprint of the key for identification
    const keyBytes = this.key.toArray()
    const hash = Hash.sha256(keyBytes)
    return Utils.toBase64(hash).substring(0, 16)
  }
}

// Example usage with error handling
const aesManager = new SecureAESManager()
console.log('Key fingerprint:', aesManager.getKeyFingerprint())

const testData = 'Sensitive information that needs protection'
const encryptResult = aesManager.safeEncrypt(testData)

if (encryptResult.success) {
  console.log('Encryption successful')
  
  const decryptResult = aesManager.safeDecrypt(encryptResult.encrypted!)
  
  if (decryptResult.success) {
    console.log('Decryption successful:', decryptResult.decrypted)
  } else {
    console.error('Decryption failed:', decryptResult.error)
  }
} else {
  console.error('Encryption failed:', encryptResult.error)
}
```

## Practical Applications

### Encrypting Transaction Metadata

```typescript
import { Transaction, PrivateKey } from '@bsv/sdk'

function encryptTransactionMetadata(
  transaction: Transaction,
  metadata: any,
  encryptionKey: SymmetricKey
): number[] {
  const metadataWithTx = {
    txid: Buffer.from(transaction.id()).toString('hex'),
    timestamp: Date.now(),
    metadata: metadata
  }
  
  const jsonString = JSON.stringify(metadataWithTx)
  return encryptionKey.encrypt(jsonString) as number[]
}

function decryptTransactionMetadata(
  encryptedMetadata: number[],
  encryptionKey: SymmetricKey
): any {
  const jsonString = encryptionKey.decrypt(encryptedMetadata, 'utf8') as string
  return JSON.parse(jsonString)
}

// Example: Encrypt private notes about a transaction
const privateKey = PrivateKey.fromRandom()
const encryptionKey = SymmetricKey.fromRandom()

// Create a simple transaction (placeholder)
const transaction = new Transaction()
// ... transaction setup ...

const privateMetadata = {
  purpose: 'Payment to supplier',
  invoiceNumber: 'INV-2024-001',
  notes: 'Quarterly payment for services',
  category: 'business-expense'
}

const encrypted = encryptTransactionMetadata(transaction, privateMetadata, encryptionKey)
const decrypted = decryptTransactionMetadata(encrypted, encryptionKey)

console.log('Original metadata:', privateMetadata)
console.log('Decrypted metadata:', decrypted)
```

### Secure Configuration Storage

```typescript
class SecureConfig {
  private encryptionKey: SymmetricKey
  private config: Map<string, number[]> = new Map()
  
  constructor(password: string) {
    // Derive encryption key from password
    const salt = Random(32)
    const passwordHash = Hash.sha256([
      ...Utils.toArray(password, 'utf8'),
      ...salt
    ])
    this.encryptionKey = new SymmetricKey(passwordHash)
  }
  
  set(key: string, value: any): void {
    const jsonValue = JSON.stringify(value)
    const encrypted = this.encryptionKey.encrypt(jsonValue) as number[]
    this.config.set(key, encrypted)
  }
  
  get(key: string): any {
    const encrypted = this.config.get(key)
    if (!encrypted) {
      return undefined
    }
    
    try {
      const decrypted = this.encryptionKey.decrypt(encrypted, 'utf8') as string
      return JSON.parse(decrypted)
    } catch (error) {
      console.error('Failed to decrypt config value:', error)
      return undefined
    }
  }
  
  has(key: string): boolean {
    return this.config.has(key)
  }
  
  delete(key: string): boolean {
    return this.config.delete(key)
  }
  
  export(): string {
    const exportData = {}
    for (const [key, encrypted] of this.config.entries()) {
      exportData[key] = Utils.toBase64(encrypted)
    }
    return JSON.stringify(exportData)
  }
  
  import(data: string): void {
    const importData = JSON.parse(data)
    for (const [key, base64Value] of Object.entries(importData)) {
      const encrypted = Utils.fromBase64(base64Value as string)
      this.config.set(key, encrypted)
    }
  }
}

// Example usage
const secureConfig = new SecureConfig('MySecurePassword123!')

// Store sensitive configuration
secureConfig.set('apiKey', 'sk-1234567890abcdef')
secureConfig.set('databaseUrl', 'postgresql://user:pass@host:5432/db')
secureConfig.set('walletSeed', 'abandon abandon abandon...')

// Retrieve configuration
console.log('API Key:', secureConfig.get('apiKey'))
console.log('Has database URL:', secureConfig.has('databaseUrl'))

// Export encrypted configuration
const exportedConfig = secureConfig.export()
console.log('Exported config length:', exportedConfig.length)
```

## Performance Considerations

### Benchmarking AES Operations

```typescript
function benchmarkAESPerformance() {
  const key = SymmetricKey.fromRandom()
  const testSizes = [100, 1000, 10000, 100000] // bytes
  
  console.log('=== AES Performance Benchmark ===')
  
  for (const size of testSizes) {
    const testData = 'x'.repeat(size)
    
    // Benchmark encryption
    const encryptStart = performance.now()
    const encrypted = key.encrypt(testData) as number[]
    const encryptTime = performance.now() - encryptStart
    
    // Benchmark decryption
    const decryptStart = performance.now()
    const decrypted = key.decrypt(encrypted, 'utf8') as string
    const decryptTime = performance.now() - decryptStart
    
    console.log(`Size: ${size} bytes`)
    console.log(`  Encrypt: ${encryptTime.toFixed(2)}ms`)
    console.log(`  Decrypt: ${decryptTime.toFixed(2)}ms`)
    console.log(`  Total: ${(encryptTime + decryptTime).toFixed(2)}ms`)
    console.log(`  Throughput: ${(size / (encryptTime + decryptTime) * 1000).toFixed(0)} bytes/sec`)
    console.log()
  }
}

// Run benchmark
benchmarkAESPerformance()
```

## Security Best Practices

### Key Security Guidelines

```typescript
class SecureKeyPractices {
  // Good: Generate random keys
  static generateSecureKey(): SymmetricKey {
    return SymmetricKey.fromRandom()
  }
  
  // Good: Derive keys from strong passwords
  static deriveFromPassword(password: string, salt: number[]): SymmetricKey {
    if (password.length < 12) {
      throw new Error('Password must be at least 12 characters')
    }
    
    const keyMaterial = Hash.sha256([
      ...Utils.toArray(password, 'utf8'),
      ...salt
    ])
    return new SymmetricKey(keyMaterial)
  }
  
  // Good: Secure key comparison
  static keysEqual(key1: SymmetricKey, key2: SymmetricKey): boolean {
    const bytes1 = key1.toArray()
    const bytes2 = key2.toArray()
    
    if (bytes1.length !== bytes2.length) {
      return false
    }
    
    // Constant-time comparison to prevent timing attacks
    let result = 0
    for (let i = 0; i < bytes1.length; i++) {
      result |= bytes1[i] ^ bytes2[i]
    }
    return result === 0
  }
  
  // Good: Secure key destruction
  static destroyKey(key: SymmetricKey): void {
    // Overwrite key material (note: this is conceptual in JavaScript)
    const keyArray = key.toArray()
    for (let i = 0; i < keyArray.length; i++) {
      keyArray[i] = 0
    }
  }
}

// Security validation example
function validateSecurityPractices() {
  console.log('=== Security Practices Validation ===')
  
  // Generate secure keys
  const key1 = SecureKeyPractices.generateSecureKey()
  const key2 = SecureKeyPractices.generateSecureKey()
  
  console.log('Keys are different:', !SecureKeyPractices.keysEqual(key1, key2))
  
  // Test password-based key derivation
  const password = 'MyVerySecurePassword123!'
  const salt = Random(32)
  const derivedKey1 = SecureKeyPractices.deriveFromPassword(password, salt)
  const derivedKey2 = SecureKeyPractices.deriveFromPassword(password, salt)
  
  console.log('Derived keys are identical:', 
    SecureKeyPractices.keysEqual(derivedKey1, derivedKey2))
  
  // Clean up
  SecureKeyPractices.destroyKey(key1)
  SecureKeyPractices.destroyKey(key2)
  
  console.log('Security validation complete')
}

validateSecurityPractices()
```

## Troubleshooting Common Issues

### Common Problems and Solutions

```typescript
function troubleshootAESIssues() {
  console.log('=== AES Troubleshooting Guide ===')
  
  // Issue 1: Decryption fails with "Decryption failed!" error
  try {
    const key = SymmetricKey.fromRandom()
    const message = 'Test message'
    const encrypted = key.encrypt(message) as number[]
    
    // Corrupt the encrypted data to simulate tampering
    encrypted[10] = encrypted[10] ^ 1
    
    const decrypted = key.decrypt(encrypted, 'utf8')
  } catch (error) {
    console.log('Detected tampered data:', error.message)
    console.log('  Solution: Verify data integrity, check for transmission errors')
  }
  
  // Issue 2: Wrong key used for decryption
  try {
    const key1 = SymmetricKey.fromRandom()
    const key2 = SymmetricKey.fromRandom()
    const message = 'Test message'
    
    const encrypted = key1.encrypt(message) as number[]
    const decrypted = key2.decrypt(encrypted, 'utf8') // Wrong key!
  } catch (error) {
    console.log('Detected wrong key usage:', error.message)
    console.log('  Solution: Ensure same key is used for encryption and decryption')
  }
  
  // Issue 3: Empty or invalid data
  try {
    const key = SymmetricKey.fromRandom()
    // Empty strings are actually supported and work fine
    const encrypted = key.encrypt('') as number[]
    const decrypted = key.decrypt(encrypted, 'utf8') as string
    console.log('Empty string encryption works:', decrypted === '')
    console.log('  Note: Empty strings are supported by the SDK')
  } catch (error) {
    console.log('Unexpected error with empty data:', error.message)
  }
  
  // Issue 4: Hex string handling
  try {
    const hexKey = 'a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456'
    // This will fail - constructor expects byte array, not hex string
    // const wrongKey = new SymmetricKey(hexKey)
    
    // Correct approach - convert hex to bytes first
    const correctKey = new SymmetricKey(hexToBytes(hexKey))
    console.log('Hex key creation works with helper function')
    console.log('  Solution: Use hexToBytes() helper function for hex strings')
    console.log('  Note: SDK does not provide Utils.fromHex() method')
  } catch (error) {
    console.log('Hex key creation issue:', error.message)
  }
  
  console.log('\nTroubleshooting complete')
}

troubleshootAESIssues()
```

## Testing Your Implementation

### Comprehensive Test Suite

```typescript
function runAESTests() {
  console.log('=== AES Implementation Tests ===')
  let passed = 0
  let total = 0
  
  function test(name: string, testFn: () => boolean) {
    total++
    try {
      const result = testFn()
      if (result) {
        console.log(` ${name}`)
        passed++
      } else {
        console.log(` ${name}`)
      }
    } catch (error) {
      console.log(` ${name} - Error: ${error.message}`)
    }
  }
  
  // Test 1: Basic encryption/decryption
  test('Basic encryption/decryption', () => {
    const key = SymmetricKey.fromRandom()
    const message = 'Hello, World!'
    const encrypted = key.encrypt(message) as number[]
    const decrypted = key.decrypt(encrypted, 'utf8') as string
    return message === decrypted
  })
  
  // Test 2: Binary data handling
  test('Binary data encryption', () => {
    const key = SymmetricKey.fromRandom()
    const binaryData = Random(100)
    const encrypted = key.encrypt(binaryData) as number[]
    const decrypted = key.decrypt(encrypted) as number[]
    return JSON.stringify(binaryData) === JSON.stringify(decrypted)
  })
  
  // Test 3: Large data encryption
  test('Large data encryption', () => {
    const key = SymmetricKey.fromRandom()
    const largeMessage = 'x'.repeat(10000)
    const encrypted = key.encrypt(largeMessage) as number[]
    const decrypted = key.decrypt(encrypted, 'utf8') as string
    return largeMessage === decrypted
  })
  
  // Test 4: Key derivation consistency
  test('Key derivation consistency', () => {
    const password = 'TestPassword123'
    const salt = Random(32)
    const key1 = new SymmetricKey(Hash.sha256([
      ...Utils.toArray(password, 'utf8'),
      ...salt
    ]))
    const key2 = new SymmetricKey(Hash.sha256([
      ...Utils.toArray(password, 'utf8'),
      ...salt
    ]))
    return key1.toHex() === key2.toHex()
  })
  
  // Test 5: Hex encoding/decoding
  test('Hex encoding support', () => {
    const key = SymmetricKey.fromRandom()
    const message = 'Test message'
    const encryptedBytes = key.encrypt(message) as number[]
    const encrypted = bytesToHex(encryptedBytes)
    const hexBytes = hexToBytes(encrypted)
    const decrypted = key.decrypt(hexBytes, 'utf8') as string
    return message === decrypted
  })
  
  console.log(`\nTests completed: ${passed}/${total} passed`)
  return passed === total
}

// Run the test suite
const allTestsPassed = runAESTests()
console.log('\nAll tests passed:', allTestsPassed)
```

## Summary

In this tutorial, you've learned how to:

1. **Generate secure AES encryption keys** using `SymmetricKey.fromRandom()`
2. **Encrypt and decrypt data** with the `encrypt()` and `decrypt()` methods
3. **Handle different data formats** including strings, binary data, and JSON
4. **Derive keys from passwords** using secure hashing techniques
5. **Implement key management** with versioning and rotation
6. **Combine AES with ECDH** for secure communication channels
7. **Apply security best practices** including error handling and validation
8. **Build practical applications** like secure configuration storage
9. **Optimize performance** and troubleshoot common issues

The BSV TypeScript SDK's `SymmetricKey` class provides a robust, secure implementation of AES-GCM encryption that's suitable for production applications. The built-in authentication prevents tampering, while the straightforward API makes it easy to integrate encryption into your Bitcoin applications.

## Next Steps

- Explore the [ECDH Key Exchange](./ecdh-key-exchange.md) tutorial for asymmetric encryption
- Learn about [Messages Reference](../reference/messages.md) for authentication
- Study [Advanced Transaction Construction](./advanced-transaction.md) for complex applications
- Review the [Security Best Practices](../guides/security-best-practices.md) guide

## Additional Resources

- [AES Wikipedia](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)
- [GCM Mode Wikipedia](https://en.wikipedia.org/wiki/Galois/Counter_Mode)
- [BSV SDK Primitives Reference](../reference/primitives.md)
