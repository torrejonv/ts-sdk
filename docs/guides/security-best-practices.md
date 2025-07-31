# Security Best Practices

This comprehensive guide covers essential security practices when developing Bitcoin applications with the BSV TypeScript SDK. Following these guidelines will help you build secure, production-ready applications that protect user funds and data.

## Prerequisites

- Understanding of Bitcoin cryptography fundamentals
- Familiarity with the BSV TypeScript SDK
- Basic knowledge of secure coding practices
- Understanding of common attack vectors in cryptocurrency applications

> **📚 Related Concepts**: This guide builds on [Key Management](../concepts/key-management.md), [Digital Signatures](../concepts/signatures.md), and [Transaction Verification](../concepts/verification.md) concepts.

## Key Security Principles

### 1. Private Key Management

#### Never Expose Private Keys

```typescript
// ❌ NEVER do this - exposing private key in logs or UI
console.log('Private key:', privateKey.toWif())
alert(`Your key: ${privateKey.toWif()}`)

// ✅ Proper handling - keep private keys secure
const privateKey = PrivateKey.fromRandom()
// Use the key for operations without exposing it
const publicKey = privateKey.toPublicKey()
```

#### Secure Key Generation

```typescript
import { PrivateKey } from '@bsv/sdk'

// ✅ Use cryptographically secure random generation
const secureKey = PrivateKey.fromRandom()

// ❌ Never use predictable sources
// const weakKey = PrivateKey.fromString('1') // Predictable
// const timeKey = PrivateKey.fromString(Date.now().toString()) // Predictable
```

#### Key Storage Best Practices

```typescript
// ✅ For production applications, use secure storage
class SecureKeyManager {
  private encryptionKey: SymmetricKey
  
  constructor() {
    // Derive encryption key from user password or hardware security module
    this.encryptionKey = SymmetricKey.fromRandom()
  }
  
  async storePrivateKey(privateKey: PrivateKey, identifier: string): Promise<void> {
    const keyData = privateKey.toWif()
    const encrypted = this.encryptionKey.encrypt(keyData)
    
    // Store encrypted key in secure storage (not localStorage for production)
    await this.secureStorage.set(identifier, Buffer.from(encrypted).toString('base64'))
  }
  
  async retrievePrivateKey(identifier: string): Promise<PrivateKey> {
    const encryptedData = await this.secureStorage.get(identifier)
    const encryptedBuffer = Buffer.from(encryptedData, 'base64')
    const decrypted = this.encryptionKey.decrypt(Array.from(encryptedBuffer), 'utf8')
    return PrivateKey.fromWif(decrypted as string)
  }
}
```

### 2. Transaction Security

#### Input Validation and Sanitization

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

class SecureTransactionBuilder {
  static validateAmount(satoshis: number): void {
    if (!Number.isInteger(satoshis)) {
      throw new Error('Amount must be an integer')
    }
    if (satoshis <= 0) {
      throw new Error('Amount must be positive')
    }
    if (satoshis > 21000000 * 100000000) {
      throw new Error('Amount exceeds maximum possible Bitcoin supply')
    }
  }
  
  static validateAddress(address: string): void {
    try {
      // Validate address format
      P2PKH.unlock('', 'all', {
        publicKey: address, // This will throw if invalid
        signature: { inputIndex: 0, outputs: [], inputScript: '' }
      })
    } catch (error) {
      throw new Error('Invalid Bitcoin address format')
    }
  }
  
  static async createSecureTransaction(
    privateKey: PrivateKey,
    recipientAddress: string,
    amount: number
  ): Promise<Transaction> {
    // Validate all inputs
    this.validateAmount(amount)
    this.validateAddress(recipientAddress)
    
    // Create transaction with validated inputs
    const tx = new Transaction()
    // ... transaction construction logic
    
    return tx
  }
}
```

#### Fee Calculation Security

```typescript
// ✅ Always validate fee calculations to prevent fee attacks
class SecureFeeCalculator {
  private static readonly MIN_FEE_RATE = 0.5 // satoshis per byte
  private static readonly MAX_FEE_RATE = 1000 // satoshis per byte
  
  static calculateFee(transactionSize: number, feeRate: number): number {
    // Validate fee rate is within reasonable bounds
    if (feeRate < this.MIN_FEE_RATE || feeRate > this.MAX_FEE_RATE) {
      throw new Error(`Fee rate must be between ${this.MIN_FEE_RATE} and ${this.MAX_FEE_RATE} sat/byte`)
    }
    
    const fee = Math.ceil(transactionSize * feeRate)
    
    // Additional validation to prevent excessive fees
    if (fee > 100000) { // 0.001 BSV maximum fee
      throw new Error('Calculated fee is unreasonably high')
    }
    
    return fee
  }
}
```

### 3. Cryptographic Operations Security

#### Secure Random Number Generation

```typescript
import { PrivateKey, SymmetricKey } from '@bsv/sdk'

// ✅ Always use the SDK's secure random generation
const securePrivateKey = PrivateKey.fromRandom()
const secureSymmetricKey = SymmetricKey.fromRandom()

// ❌ Never use Math.random() for cryptographic purposes
// const insecureKey = PrivateKey.fromString(Math.random().toString())
```

#### ECDH Key Exchange Security

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

class SecureECDH {
  static performKeyExchange(
    myPrivateKey: PrivateKey,
    theirPublicKey: PublicKey
  ): Buffer {
    try {
      // The SDK automatically validates the public key and prevents twist attacks
      const sharedSecret = myPrivateKey.deriveSharedSecret(theirPublicKey)
      
      // ✅ Always derive keys from the shared secret, never use it directly
      if (!sharedSecret.x) {
        throw new Error('Invalid shared secret')
      }
      const sharedSecretBuffer = Buffer.from(sharedSecret.x.toArray())
      const contextBuffer = Buffer.from('application-specific-context', 'utf8')
      const combinedBuffer = Buffer.concat([sharedSecretBuffer, contextBuffer])
      const derivedKey = Hash.sha256(Array.from(combinedBuffer))
      
      return Buffer.from(derivedKey)
    } catch (error) {
      throw new Error('Key exchange failed: Invalid public key')
    }
  }
}
```

#### AES Encryption Security

```typescript
import { SymmetricKey, Hash } from '@bsv/sdk'

class SecureEncryption {
  // ✅ Proper key derivation from passwords
  static deriveKeyFromPassword(password: string, salt: Buffer): SymmetricKey {
    if (password.length < 12) {
      throw new Error('Password must be at least 12 characters')
    }
    
    // Use multiple rounds of hashing for key derivation
    let derived = Array.from(Buffer.concat([Buffer.from(password, 'utf8'), salt]))
    for (let i = 0; i < 10000; i++) {
      derived = Hash.sha256(derived)
    }
    
    return new SymmetricKey(derived)
  }
  
  // ✅ Secure encryption with proper error handling
  static encryptData(data: string, key: SymmetricKey): string {
    try {
      const encrypted = key.encrypt(data)
      return Buffer.from(encrypted).toString('base64')
    } catch (error) {
      // Don't expose internal error details
      throw new Error('Encryption failed')
    }
  }
  
  // ✅ Secure decryption with validation
  static decryptData(encryptedData: string, key: SymmetricKey): string {
    try {
      const encrypted = Buffer.from(encryptedData, 'base64')
      const decrypted = key.decrypt(Array.from(encrypted), 'utf8')
      return decrypted as string
    } catch (error) {
      throw new Error('Decryption failed: Invalid data or key')
    }
  }
}
```

### 4. Wallet Integration Security

#### Secure WalletClient Usage

```typescript
import { WalletClient } from '@bsv/sdk'

class SecureWalletManager {
  private wallet: WalletClient | null = null
  private connectionAttempts = 0
  private readonly MAX_CONNECTION_ATTEMPTS = 3
  
  async connectWallet(): Promise<void> {
    if (this.connectionAttempts >= this.MAX_CONNECTION_ATTEMPTS) {
      throw new Error('Maximum connection attempts exceeded')
    }
    
    try {
      this.wallet = new WalletClient('auto', 'localhost')
      // Connection is established during construction
      this.connectionAttempts = 0 // Reset on successful connection
    } catch (error) {
      this.connectionAttempts++
      throw new Error('Wallet connection failed')
    }
  }
  
  async createSecureTransaction(outputs: any[]): Promise<any> {
    if (!this.wallet) {
      throw new Error('Wallet not connected')
    }
    
    // Validate all outputs before creating transaction
    for (const output of outputs) {
      if (!output.satoshis || output.satoshis <= 0) {
        throw new Error('Invalid output amount')
      }
      if (!output.lockingScript) {
        throw new Error('Missing locking script')
      }
    }
    
    try {
      return await this.wallet.createAction({
        description: 'Secure transaction',
        outputs,
        // ✅ Always include proper error handling options
        options: {
          acceptDelayedBroadcast: false, // Ensure immediate feedback
          randomizeOutputs: true // Enhance privacy
        }
      })
    } catch (error) {
      // Log error securely without exposing sensitive data
      console.error('Transaction creation failed:', error.message)
      throw new Error('Transaction creation failed')
    }
  }
}
```

When using the `WalletClient` interface, follow these security practices:

### Secure `WalletClient` Usage

The `WalletClient` provides built-in security features, but proper usage is essential:

### `WalletClient` Connection Management

When working with `WalletClient` connections:

### 5. Network Security

#### Secure Chain Tracker Usage

```typescript
import { ChainTracker, WhatsOnChain } from '@bsv/sdk'

class SecureChainTracker {
  private trackers: ChainTracker[]
  private currentTrackerIndex = 0
  
  constructor() {
    // ✅ Use multiple chain trackers for redundancy
    this.trackers = [
      new WhatsOnChain('main'),
      // Add additional trackers for failover
    ]
  }
  
  async getTransactionWithRetry(txid: string): Promise<any> {
    let lastError: Error | null = null
    
    // Try each tracker
    for (let i = 0; i < this.trackers.length; i++) {
      try {
        const tracker = this.trackers[this.currentTrackerIndex]
        const result = await tracker.getTransaction(txid)
        
        // Validate the response
        if (!result || !result.id) {
          throw new Error('Invalid transaction response')
        }
        
        return result
      } catch (error) {
        lastError = error as Error
        this.currentTrackerIndex = (this.currentTrackerIndex + 1) % this.trackers.length
      }
    }
    
    throw new Error(`All chain trackers failed: ${lastError?.message}`)
  }
}
```

### 6. SPV Verification Security

#### Secure Merkle Proof Verification

```typescript
import { Transaction, MerklePath } from '@bsv/sdk'

class SecureSPVVerifier {
  static async verifyTransaction(
    transaction: Transaction,
    merklePath: MerklePath,
    blockHeader: any
  ): Promise<boolean> {
    try {
      // ✅ Always verify the merkle proof
      const txid = Buffer.from(transaction.id()).toString('hex')
      const computedRoot = merklePath.computeRoot(txid)
      
      if (computedRoot !== blockHeader.merkleRoot) {
        throw new Error('Merkle proof verification failed')
      }
      
      // ✅ Verify the transaction itself
      const isValid = await transaction.verify()
      if (!isValid) {
        throw new Error('Transaction verification failed')
      }
      
      return true
    } catch (error) {
      console.error('SPV verification failed:', error.message)
      return false
    }
  }
}
```

### 7. Error Handling Security

#### Secure Error Reporting

```typescript
class SecureErrorHandler {
  // ✅ Sanitize error messages to prevent information leakage
  static sanitizeError(error: Error): string {
    const sensitivePatterns = [
      /private.*key/i,
      /seed/i,
      /mnemonic/i,
      /password/i,
      /secret/i
    ]
    
    let message = error.message
    
    for (const pattern of sensitivePatterns) {
      message = message.replace(pattern, '[REDACTED]')
    }
    
    return message
  }
  
  // ✅ Secure error logging
  static logError(error: Error, context: string): void {
    const sanitizedMessage = this.sanitizeError(error)
    console.error(`[${context}] ${sanitizedMessage}`)
    
    // In production, send to secure logging service
    // Never log sensitive information
  }
}
```

## Common Security Vulnerabilities

### 1. Private Key Exposure

```typescript
// ❌ Common mistakes that expose private keys
class InsecureExamples {
  // Never store keys in plain text
  private userKey = 'L1234567890abcdef...' // Exposed in source code
  
  // Never log private keys
  debugTransaction(privateKey: PrivateKey) {
    console.log('Signing with key:', privateKey.toWif()) // Logged
  }
  
  // Never send keys over insecure channels
  async sendKeyToServer(key: PrivateKey) {
    await fetch('http://api.example.com/keys', { // HTTP not HTTPS
      method: 'POST',
      body: JSON.stringify({ key: key.toWif() })
    })
  }
}
```

### 2. Insufficient Input Validation

```typescript
// ❌ Vulnerable to various attacks
class VulnerableTransaction {
  async createTransaction(amount: string, address: string) {
    // No validation - vulnerable to injection and overflow
    const tx = new Transaction()
    tx.addOutput({
      satoshis: parseInt(amount), // No validation
      lockingScript: address // No validation
    })
    return tx
  }
}
```

### 3. Weak Random Number Generation

```typescript
// ❌ Predictable and insecure
class WeakRandomness {
  generatePrivateKey(): PrivateKey {
    // Predictable seed
    const seed = Date.now().toString() + Math.random().toString()
    return PrivateKey.fromString(seed)
  }
}
```

## Security Resources

### Additional Reading

- [Bitcoin Security Best Practices](https://en.bitcoin.it/wiki/Securing_your_wallet)
- [Cryptographic Best Practices](https://cryptography.io/en/latest/faq/)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)

## Conclusion

Security in Bitcoin applications requires constant vigilance and adherence to best practices. The BSV TypeScript SDK provides many security features out of the box, but proper implementation and configuration are crucial for maintaining security.
