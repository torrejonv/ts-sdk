# Creating Encrypted Messages

This guide demonstrates how to create secure, encrypted messaging systems using the BSV TypeScript SDK. You'll learn multiple approaches from high-level BRC-78 protocols to low-level cryptographic implementations.

## Overview

The BSV TypeScript SDK provides several approaches for encrypting messages:

1. **BRC-78 Message Encryption** (Recommended) - High-level protocol with built-in key derivation
2. **Wallet-Managed Encryption** - Protocol-based encryption with wallet integration
3. **Manual AES + ECDH Implementation** - Custom implementation with full control
4. **Group Messaging** - Advanced patterns for multi-party communication

## Prerequisites

- BSV TypeScript SDK installed
- Basic understanding of public key cryptography
- Familiarity with elliptic curve key exchange (ECDH)
- Understanding of symmetric encryption (AES)

## BRC-78 Message Encryption (Recommended)

BRC-78 provides a standardized approach to message encryption with built-in key derivation and security best practices.

### Basic Implementation

```typescript
import { encrypt, decrypt } from '@bsv/sdk/messages/EncryptedMessage'
import { PrivateKey, PublicKey, Utils } from '@bsv/sdk'

class MessageEncryption {
  private senderPrivateKey: PrivateKey
  
  constructor(senderPrivateKey: PrivateKey) {
    this.senderPrivateKey = senderPrivateKey
  }
  
  // Encrypt a message to a recipient
  encryptMessage(message: string, recipientPublicKey: PublicKey): number[] {
    const messageBytes = Utils.toArray(message, 'utf8')
    return encrypt(messageBytes, this.senderPrivateKey, recipientPublicKey)
  }
  
  // Decrypt a message received from a sender
  decryptMessage(encryptedMessage: number[]): string {
    const decryptedBytes = decrypt(encryptedMessage, this.senderPrivateKey)
    return Utils.toUTF8(decryptedBytes)
  }
}
```

### Practical Example: Secure Chat

```typescript
import { encrypt, decrypt } from '@bsv/sdk/messages/EncryptedMessage'
import { PrivateKey, PublicKey, Utils } from '@bsv/sdk'

class SecureChat {
  private myPrivateKey: PrivateKey
  private myPublicKey: PublicKey
  
  constructor() {
    this.myPrivateKey = PrivateKey.fromRandom()
    this.myPublicKey = this.myPrivateKey.toPublicKey()
  }
  
  // Send an encrypted message
  sendMessage(message: string, recipientPublicKey: PublicKey): {
    encryptedMessage: number[]
    senderPublicKey: string
  } {
    try {
      const messageBytes = Utils.toArray(message, 'utf8')
      const encryptedMessage = encrypt(messageBytes, this.myPrivateKey, recipientPublicKey)
      
      return {
        encryptedMessage,
        senderPublicKey: this.myPublicKey.toString()
      }
    } catch (error) {
      throw new Error(`Failed to encrypt message: ${error.message}`)
    }
  }
  
  // Receive and decrypt a message
  receiveMessage(encryptedMessage: number[]): {
    message: string
    senderPublicKey: string
  } {
    try {
      const decryptedBytes = decrypt(encryptedMessage, this.myPrivateKey)
      const message = Utils.toUTF8(decryptedBytes)
      
      // Extract sender public key from encrypted message
      const senderPublicKey = PublicKey.fromString(
        Utils.toHex(encryptedMessage.slice(4, 37))
      ).toString()
      
      return { message, senderPublicKey }
    } catch (error) {
      throw new Error(`Failed to decrypt message: ${error.message}`)
    }
  }
  
  getPublicKey(): string {
    return this.myPublicKey.toString()
  }
}
```

### Usage Example

```typescript
async function demonstrateSecureChat() {
  console.log('=== Secure Chat Demo ===')
  
  // Create two chat participants
  const alice = new SecureChat()
  const bob = new SecureChat()
  
  console.log('Alice public key:', alice.getPublicKey())
  console.log('Bob public key:', bob.getPublicKey())
  
  // Alice sends a message to Bob
  const aliceMessage = "Hello Bob! This is a secret message."
  const encryptedFromAlice = alice.sendMessage(
    aliceMessage, 
    PublicKey.fromString(bob.getPublicKey())
  )
  
  console.log('Encrypted message length:', encryptedFromAlice.encryptedMessage.length)
  
  // Bob receives and decrypts Alice's message
  const decryptedByBob = bob.receiveMessage(encryptedFromAlice.encryptedMessage)
  console.log('Decrypted message:', decryptedByBob.message)
  console.log('Sender was:', decryptedByBob.senderPublicKey)
  
  // Bob replies to Alice
  const bobReply = "Hi Alice! I received your secret message safely."
  const encryptedFromBob = bob.sendMessage(
    bobReply,
    PublicKey.fromString(alice.getPublicKey())
  )
  
  // Alice receives Bob's reply
  const decryptedByAlice = alice.receiveMessage(encryptedFromBob.encryptedMessage)
  console.log('Bob replied:', decryptedByAlice.message)
}

// Run the demo
demonstrateSecureChat().catch(console.error)
```

## Wallet-Managed Encryption

For applications integrated with BSV wallets, you can use protocol-based encryption that leverages the wallet's key management.

### Implementation

```typescript
import { WalletClient } from '@bsv/sdk'

class WalletMessageEncryption {
  private wallet: WalletClient
  private protocolID: [number, string] = [2, 'secure-messaging']
  
  constructor(wallet: WalletClient) {
    this.wallet = wallet
  }
  
  // Encrypt message for self (backup/storage)
  async encryptForSelf(message: string, keyID: string): Promise<number[]> {
    const messageBytes = Utils.toArray(message, 'utf8')
    
    const result = await this.wallet.encrypt({
      plaintext: messageBytes,
      protocolID: this.protocolID,
      keyID: keyID,
      counterparty: 'self',
      privilegedReason: 'Message backup encryption'
    })
    
    return result.ciphertext
  }
  
  // Decrypt message from self
  async decryptFromSelf(ciphertext: number[], keyID: string): Promise<string> {
    const result = await this.wallet.decrypt({
      ciphertext,
      protocolID: this.protocolID,
      keyID: keyID,
      counterparty: 'self',
      privilegedReason: 'Message backup decryption'
    })
    
    return Utils.toUTF8(result.plaintext)
  }
  
  // Encrypt message for specific counterparty
  async encryptForCounterparty(
    message: string, 
    keyID: string, 
    counterpartyPubKey: string
  ): Promise<number[]> {
    const messageBytes = Utils.toArray(message, 'utf8')
    
    const result = await this.wallet.encrypt({
      plaintext: messageBytes,
      protocolID: this.protocolID,
      keyID: keyID,
      counterparty: counterpartyPubKey,
      privilegedReason: 'Secure message to counterparty'
    })
    
    return result.ciphertext
  }
  
  // Decrypt message from counterparty
  async decryptFromCounterparty(
    ciphertext: number[], 
    keyID: string, 
    counterpartyPubKey: string
  ): Promise<string> {
    const result = await this.wallet.decrypt({
      ciphertext,
      protocolID: this.protocolID,
      keyID: keyID,
      counterparty: counterpartyPubKey,
      privilegedReason: 'Decrypt message from counterparty'
    })
    
    return Utils.toUTF8(result.plaintext)
  }
}
```

### Usage with Wallet

```typescript
async function demonstrateWalletEncryption() {
  console.log('=== Wallet Encryption Demo ===')
  
  const wallet = new WalletClient('https://staging-dojo.babbage.systems')
  const encryption = new WalletMessageEncryption(wallet)
  
  try {
    // Encrypt a message for self-storage
    const message = "This is my private note that I want to store securely."
    const keyID = "personal-note-001"
    
    const encrypted = await encryption.encryptForSelf(message, keyID)
    console.log('Encrypted message length:', encrypted.length)
    
    // Decrypt the message
    const decrypted = await encryption.decryptFromSelf(encrypted, keyID)
    console.log('Decrypted message:', decrypted)
    
  } catch (error) {
    console.error('Wallet encryption error:', error.message)
  }
}
```

## ECIES Implementation

Elliptic Curve Integrated Encryption Scheme (ECIES) is a hybrid encryption scheme that combines the benefits of both asymmetric and symmetric encryption. This implementation follows the standard ECIES protocol using the BSV TypeScript SDK.

### Standard ECIES Implementation

```typescript
import { PrivateKey, PublicKey, SymmetricKey, Utils, Hash } from '@bsv/sdk'

class ECIESEncryption {
  private privateKey: PrivateKey
  
  constructor(privateKey: PrivateKey) {
    this.privateKey = privateKey
  }
  
  /**
   * Encrypt a message using ECIES
   */
  encrypt(message: string, recipientPublicKey: PublicKey): {
    encryptedData: number[]
  } {
    try {
      const messageBytes = Utils.toArray(message, 'utf8')
      const encryptedData = ECIES.electrumEncrypt(
        messageBytes,
        recipientPublicKey,
        this.privateKey
      )
      
      return {
        encryptedData
      }
      
    } catch (error: any) {
      throw new Error(`ECIES encryption failed: ${error.message}`)
    }
  }
  
  /**
   * Decrypt a message using ECIES
   */
  decrypt(eciesData: {
    encryptedData: number[]
  }): string {
    try {
      const decryptedBytes = ECIES.electrumDecrypt(
        eciesData.encryptedData,
        this.privateKey
      )
      
      return Utils.toUTF8(decryptedBytes)
      
    } catch (error: any) {
      throw new Error(`ECIES decryption failed: ${error.message}`)
    }
  }
  
  getPublicKey(): PublicKey {
    return this.privateKey.toPublicKey()
  }
}
```

### Practical ECIES Example

```typescript
// Example usage of ECIES encryption
async function demonstrateECIES() {
  try {
    // Create ECIES instances for Alice and Bob
    const alice = new ECIESEncryption(PrivateKey.fromRandom())
    const bob = new ECIESEncryption(PrivateKey.fromRandom())
    
    console.log('Alice public key:', alice.getPublicKey().toString())
    console.log('Bob public key:', bob.getPublicKey().toString())
    
    // Alice encrypts a message for Bob
    const message = 'Hello Bob! This is a secure ECIES message.'
    const encrypted = alice.encrypt(message, bob.getPublicKey())
    
    console.log('Encrypted message:', {
      ephemeralPublicKey: encrypted.ephemeralPublicKey.substring(0, 20) + '...',
      encryptedMessage: encrypted.encryptedMessage.length + ' bytes',
      mac: encrypted.mac.length + ' bytes'
    })
    
    // Bob decrypts the message
    const decrypted = bob.decrypt(encrypted)
    console.log('Decrypted message:', decrypted)
    
    // Verify the message was transmitted correctly
    console.log('Message integrity verified:', message === decrypted)
    
  } catch (error) {
    console.error('ECIES demonstration error:', error.message)
  }
}

// Run the demonstration
demonstrateECIES().catch(console.error)
```

### Advanced ECIES with Custom KDF

```typescript
class AdvancedECIES extends ECIESEncryption {
  /**
   * Custom Key Derivation Function using HKDF-like approach
   */
  private deriveKeys(sharedSecret: string, salt: number[] = []): {
    encryptionKey: SymmetricKey
    macKey: number[]
  } {
    // Use salt if provided, otherwise use default
    const saltBytes = salt.length > 0 ? salt : Utils.toArray('ECIES-v1', 'utf8')
    
    // HKDF Extract: HMAC(salt, shared_secret)
    const sharedSecretBytes = Utils.toArray(sharedSecret, 'hex')
    const prk = Hash.sha256hmac(saltBytes, sharedSecretBytes)
    
    // HKDF Expand: Generate encryption key
    const encInfo = Utils.toArray('encryption', 'utf8')
    const encKeyMaterial = Hash.sha256hmac([], [...Array.from(prk), ...encInfo, 0x01])
    const encryptionKey = new SymmetricKey(Array.from(encKeyMaterial.slice(0, 16)))
    
    // HKDF Expand: Generate MAC key
    const macInfo = Utils.toArray('authentication', 'utf8')
    const macKeyMaterial = Hash.sha256hmac([], [...Array.from(prk), ...macInfo, 0x02])
    const macKey = Array.from(macKeyMaterial.slice(0, 32))
    
    return { encryptionKey, macKey }
  }
  
  /**
   * Encrypt with custom salt for additional security
   */
  encryptWithSalt(message: string, recipientPublicKey: PublicKey, salt?: number[]): {
    ephemeralPublicKey: string
    encryptedMessage: number[]
    mac: number[]
    salt: number[]
  } {
    try {
      // Generate random salt if not provided
      const usedSalt = salt || Array.from(crypto.getRandomValues(new Uint8Array(16)))
      
      // Generate ephemeral key pair
      const ephemeralPrivateKey = PrivateKey.fromRandom()
      const ephemeralPublicKey = ephemeralPrivateKey.toPublicKey()
      
      // Derive shared secret
      const sharedSecret = ephemeralPrivateKey.deriveSharedSecret(recipientPublicKey)
      
      // Derive keys using custom KDF
      const { encryptionKey, macKey } = this.deriveKeys(sharedSecret.toString(), usedSalt)
      
      // Encrypt message
      const messageBytes = Utils.toArray(message, 'utf8')
      const encryptedMessage = encryptionKey.encrypt(messageBytes) as number[]
      
      // Calculate MAC over all components
      const ephemeralKeyBytes = Utils.toArray(ephemeralPublicKey.toString(), 'hex')
      const macInput = [...usedSalt, ...ephemeralKeyBytes, ...encryptedMessage]
      const mac = Hash.sha256hmac(macKey, macInput)
      
      return {
        ephemeralPublicKey: ephemeralPublicKey.toString(),
        encryptedMessage,
        mac: Array.from(mac),
        salt: usedSalt
      }
      
    } catch (error) {
      throw new Error(`Advanced ECIES encryption failed: ${error.message}`)
    }
  }
}
```

### ECIES Security Considerations

1. **Ephemeral Keys**: Each encryption uses a fresh ephemeral key pair for forward secrecy
2. **MAC Verification**: Always verify the MAC before decryption to detect tampering
3. **Constant-Time Comparison**: Use constant-time MAC comparison to prevent timing attacks
4. **Key Derivation**: Proper KDF ensures encryption and MAC keys are independent
5. **Salt Usage**: Optional salt adds additional security for key derivation

### ECIES vs Other Approaches

| Feature | ECIES | BRC-78 | Manual ECDH+AES |
|---------|-------|--------|------------------|
| **Standardization** | IEEE 1363a | BRC-78 Protocol | Custom |
| **Forward Secrecy** | ✅ Ephemeral keys | ✅ Built-in | ⚠️ Manual |
| **MAC Protection** | ✅ Integrated | ✅ Built-in | ⚠️ Manual |
| **Complexity** | Medium | Low | High |
| **Interoperability** | ✅ Standard | ⚠️ BSV-specific | ❌ Custom |

## Manual AES + ECDH Implementation

For maximum control, you can manually combine AES encryption with ECDH key exchange.

### Advanced Implementation

```typescript
import { PrivateKey, PublicKey, SymmetricKey, Utils } from '@bsv/sdk'

class ManualMessageEncryption {
  private privateKey: PrivateKey
  
  constructor(privateKey?: PrivateKey) {
    this.privateKey = privateKey || PrivateKey.fromRandom()
  }
  
  // Encrypt message using ECDH + AES
  encryptMessage(message: string, recipientPublicKey: PublicKey): {
    encryptedMessage: number[]
    senderPublicKey: string
    ephemeralPublicKey?: string
  } {
    try {
      // Derive shared secret using ECDH
      const sharedSecret = this.privateKey.deriveSharedSecret(recipientPublicKey)
      
      // Create AES key from shared secret
      const symmetricKey = new SymmetricKey(sharedSecret.encode(true).slice(1))
      
      // Encrypt the message
      const messageBytes = Utils.toArray(message, 'utf8')
      const encryptedMessage = symmetricKey.encrypt(messageBytes) as number[]
      
      return {
        encryptedMessage,
        senderPublicKey: this.privateKey.toPublicKey().toString()
      }
    } catch (error) {
      throw new Error(`Encryption failed: ${error.message}`)
    }
  }
  
  // Decrypt message using ECDH + AES
  decryptMessage(
    encryptedMessage: number[], 
    senderPublicKey: PublicKey
  ): string {
    try {
      // Derive the same shared secret
      const sharedSecret = this.privateKey.deriveSharedSecret(senderPublicKey)
      
      // Create the same AES key
      const symmetricKey = new SymmetricKey(sharedSecret.encode(true).slice(1))
      
      // Decrypt the message
      const decryptedBytes = symmetricKey.decrypt(encryptedMessage) as number[]
      return Utils.toUTF8(decryptedBytes)
    } catch (error) {
      throw new Error(`Decryption failed: ${error.message}`)
    }
  }
  
  getPublicKey(): PublicKey {
    return this.privateKey.toPublicKey()
  }
}
```

## Group Messaging with Key Rotation

For more advanced scenarios, implement group messaging with periodic key rotation.

### Group Encryption Implementation

```typescript
class GroupMessageEncryption {
  private groupKey: SymmetricKey
  private keyVersion: number
  private keyRotationInterval: number // in milliseconds
  private lastKeyRotation: number
  
  constructor(keyRotationInterval: number = 24 * 60 * 60 * 1000) { // 24 hours
    this.keyRotationInterval = keyRotationInterval
    this.rotateGroupKey()
  }
  
  // Rotate the group encryption key
  rotateGroupKey(): void {
    this.groupKey = SymmetricKey.fromRandom()
    this.keyVersion = Date.now()
    this.lastKeyRotation = Date.now()
    console.log(`Group key rotated. New version: ${this.keyVersion}`)
  }
  
  // Check if key rotation is needed
  checkKeyRotation(): void {
    const now = Date.now()
    if (now - this.lastKeyRotation > this.keyRotationInterval) {
      this.rotateGroupKey()
    }
  }
  
  // Encrypt message for group
  encryptGroupMessage(message: string): {
    encryptedMessage: number[]
    keyVersion: number
    timestamp: number
  } {
    this.checkKeyRotation()
    
    const messageBytes = Utils.toArray(message, 'utf8')
    const encryptedMessage = this.groupKey.encrypt(messageBytes) as number[]
    
    return {
      encryptedMessage,
      keyVersion: this.keyVersion,
      timestamp: Date.now()
    }
  }
  
  // Decrypt group message (simplified - in practice, you'd need key versioning)
  decryptGroupMessage(encryptedMessage: number[]): string {
    const decryptedBytes = this.groupKey.decrypt(encryptedMessage) as number[]
    return Utils.toUTF8(decryptedBytes)
  }
  
  // Get current group key for sharing with new members
  getCurrentGroupKey(): {
    key: string
    version: number
  } {
    return {
      key: this.groupKey.toHex(),
      version: this.keyVersion
    }
  }
  
  // Set group key (for new members joining)
  setGroupKey(keyHex: string, version: number): void {
    const keyBytes = Utils.toArray(keyHex, 'hex')
    this.groupKey = new SymmetricKey(keyBytes)
    this.keyVersion = version
    this.lastKeyRotation = Date.now()
  }
}
```

## Error Handling and Validation

Implement robust error handling for production message encryption systems.

### Comprehensive Error Handling

```typescript
export enum MessageEncryptionError {
  INVALID_KEY = 'INVALID_KEY',
  ENCRYPTION_FAILED = 'ENCRYPTION_FAILED',
  DECRYPTION_FAILED = 'DECRYPTION_FAILED',
  INVALID_MESSAGE_FORMAT = 'INVALID_MESSAGE_FORMAT',
  KEY_DERIVATION_FAILED = 'KEY_DERIVATION_FAILED'
}

class SecureMessageHandler {
  private static validatePrivateKey(privateKey: PrivateKey): void {
    if (!privateKey) {
      throw new Error(`${MessageEncryptionError.INVALID_KEY}: Private key is required`)
    }
    
    try {
      // Validate by attempting to get public key
      privateKey.toPublicKey()
    } catch (error) {
      throw new Error(`${MessageEncryptionError.INVALID_KEY}: Invalid private key format`)
    }
  }
  
  private static validatePublicKey(publicKey: PublicKey): void {
    if (!publicKey) {
      throw new Error(`${MessageEncryptionError.INVALID_KEY}: Public key is required`)
    }
    
    try {
      // Validate by attempting to encode
      publicKey.encode(true)
    } catch (error) {
      throw new Error(`${MessageEncryptionError.INVALID_KEY}: Invalid public key format`)
    }
  }
  
  private static validateMessage(message: string): void {
    if (!message || message.trim().length === 0) {
      throw new Error(`${MessageEncryptionError.INVALID_MESSAGE_FORMAT}: Message cannot be empty`)
    }
    
    if (message.length > 1000000) { // 1MB limit
      throw new Error(`${MessageEncryptionError.INVALID_MESSAGE_FORMAT}: Message too large`)
    }
  }
  
  static encryptMessageSafely(
    message: string,
    senderPrivateKey: PrivateKey,
    recipientPublicKey: PublicKey
  ): number[] {
    try {
      // Validate inputs
      this.validateMessage(message)
      this.validatePrivateKey(senderPrivateKey)
      this.validatePublicKey(recipientPublicKey)
      
      // Encrypt using BRC-78
      const messageBytes = Utils.toArray(message, 'utf8')
      return encrypt(messageBytes, senderPrivateKey, recipientPublicKey)
      
    } catch (error) {
      if (error.message.includes(MessageEncryptionError.INVALID_KEY) ||
          error.message.includes(MessageEncryptionError.INVALID_MESSAGE_FORMAT)) {
        throw error // Re-throw validation errors
      }
      throw new Error(`${MessageEncryptionError.ENCRYPTION_FAILED}: ${error.message}`)
    }
  }
  
  static decryptMessageSafely(
    encryptedMessage: number[],
    recipientPrivateKey: PrivateKey
  ): string {
    try {
      // Validate inputs
      this.validatePrivateKey(recipientPrivateKey)
      
      if (!encryptedMessage || encryptedMessage.length === 0) {
        throw new Error(`${MessageEncryptionError.INVALID_MESSAGE_FORMAT}: Encrypted message is empty`)
      }
      
      // Decrypt using BRC-78
      const decryptedBytes = decrypt(encryptedMessage, recipientPrivateKey)
      return Utils.toUTF8(decryptedBytes)
      
    } catch (error) {
      if (error.message.includes(MessageEncryptionError.INVALID_KEY) ||
          error.message.includes(MessageEncryptionError.INVALID_MESSAGE_FORMAT)) {
        throw error // Re-throw validation errors
      }
      throw new Error(`${MessageEncryptionError.DECRYPTION_FAILED}: ${error.message}`)
    }
  }
}
```

## Performance Optimization

For high-throughput messaging applications, implement caching and batching strategies.

### Optimized Message Handler

```typescript
class OptimizedMessageHandler {
  private keyCache = new Map<string, SymmetricKey>()
  private maxCacheSize = 1000
  
  // Cache derived keys to avoid repeated ECDH operations
  private getCachedSymmetricKey(
    privateKey: PrivateKey, 
    publicKey: PublicKey
  ): SymmetricKey {
    const cacheKey = `${privateKey.toWif()}-${publicKey.toString()}`
    
    if (this.keyCache.has(cacheKey)) {
      return this.keyCache.get(cacheKey)!
    }
    
    // Derive new key
    const sharedSecret = privateKey.deriveSharedSecret(publicKey)
    const symmetricKey = new SymmetricKey(sharedSecret.encode(true).slice(1))
    
    // Cache with LRU eviction
    if (this.keyCache.size >= this.maxCacheSize) {
      const firstKey = this.keyCache.keys().next().value
      this.keyCache.delete(firstKey)
    }
    
    this.keyCache.set(cacheKey, symmetricKey)
    return symmetricKey
  }
  
  // Batch encrypt multiple messages
  encryptBatch(
    messages: string[],
    senderPrivateKey: PrivateKey,
    recipientPublicKey: PublicKey
  ): number[][] {
    const symmetricKey = this.getCachedSymmetricKey(senderPrivateKey, recipientPublicKey)
    
    return messages.map(message => {
      const messageBytes = Utils.toArray(message, 'utf8')
      return symmetricKey.encrypt(messageBytes) as number[]
    })
  }
  
  // Clear cache when needed
  clearCache(): void {
    this.keyCache.clear()
  }
}
```

## Best Practices

### Security Guidelines

1. **Use BRC-78 for standard messaging** - It provides proper key derivation and message format
2. **Validate all inputs** - Check keys and message formats before processing
3. **Implement proper error handling** - Don't expose sensitive information in error messages
4. **Use ephemeral keys when possible** - Generate new keys for each session
5. **Implement key rotation** - Regularly rotate group keys and session keys

### Performance Guidelines

1. **Cache derived keys** - Avoid repeated ECDH operations for the same key pairs
2. **Batch operations** - Process multiple messages together when possible
3. **Limit message size** - Set reasonable limits to prevent memory issues
4. **Use streaming for large messages** - Consider chunking very large messages
