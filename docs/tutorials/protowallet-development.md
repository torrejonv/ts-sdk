# Working with ProtoWallet for Development

**Duration**: 45 minutes  
**Prerequisites**: Node.js, basic TypeScript knowledge, completed "Your First BSV Transaction" tutorial  
**Learning Goals**:

- Understand ProtoWallet's role in development and testing
- Implement cryptographic operations without blockchain interaction
- Use ProtoWallet for key derivation and signing
- Build development tools and testing frameworks

## Introduction

ProtoWallet is a lightweight wallet implementation designed for development and testing scenarios. Unlike full wallets, ProtoWallet focuses purely on cryptographic operations without blockchain interaction, making it perfect for:

- Development and testing environments
- Cryptographic operation prototyping
- Key derivation and signing operations
- Building wallet-like functionality without full wallet complexity

## What You'll Build

In this tutorial, you'll create a development toolkit using ProtoWallet that includes:

- Key generation and management
- Message signing and verification
- Symmetric encryption/decryption
- HMAC creation and verification

## Setting Up ProtoWallet

### Basic ProtoWallet Creation

```typescript
import { ProtoWallet, PrivateKey } from '@bsv/sdk'

async function createProtoWallet() {
  // Create with a random private key
  const randomWallet = new ProtoWallet(PrivateKey.fromRandom())
  
  // Create with a specific private key
  const privateKey = PrivateKey.fromRandom()
  const specificWallet = new ProtoWallet(privateKey)
  
  // Create with 'anyone' key (for testing)
  const anyoneWallet = new ProtoWallet('anyone')
  
  console.log('ProtoWallet instances created successfully')
  return { randomWallet, specificWallet, anyoneWallet }
}

createProtoWallet().catch(console.error)
```

### Getting Public Keys

```typescript
import { ProtoWallet } from '@bsv/sdk'

async function demonstratePublicKeys() {
  const wallet = new ProtoWallet(PrivateKey.fromRandom())
  
  // Get identity public key
  const { publicKey: identityKey } = await wallet.getPublicKey({
    identityKey: true
  })
  console.log('Identity Key:', identityKey)
  
  // Get derived public key for a protocol
  const { publicKey: protocolKey } = await wallet.getPublicKey({
    protocolID: [1, 'my-app'],
    keyID: 'user-signing-key'
  })
  console.log('Protocol Key:', protocolKey)
  
  // Get public key for counterparty communication
  const { publicKey: counterpartyKey } = await wallet.getPublicKey({
    protocolID: [1, 'messaging'],
    keyID: 'chat-key',
    counterparty: identityKey
  })
  console.log('Counterparty Key:', counterpartyKey)
}

demonstratePublicKeys().catch(console.error)
```

## Digital Signatures with ProtoWallet

### Creating and Verifying Signatures

```typescript
import { ProtoWallet, Utils } from '@bsv/sdk'

async function demonstrateSignatures() {
  const wallet = new ProtoWallet(PrivateKey.fromRandom())
  
  // Message to sign
  const message = 'Hello, BSV development!'
  const messageBytes = Utils.toArray(message, 'utf8')
  
  // Create signature
  const { signature } = await wallet.createSignature({
    data: messageBytes,
    protocolID: [1, 'document signing'],
    keyID: 'doc-key',
    counterparty: 'self'
  })
  
  console.log('Message:', message)
  console.log('Signature:', Utils.toBase64(signature))
  
  // Verify signature
  const { valid } = await wallet.verifySignature({
    data: messageBytes,
    signature,
    protocolID: [1, 'document signing'],
    keyID: 'doc-key',
    counterparty: 'self'
  })
  
  console.log('Signature valid:', valid)
  
  return { message, signature, valid }
}

demonstrateSignatures().catch(console.error)
```

### Advanced Signature Scenarios

```typescript
import { ProtoWallet, Utils } from '@bsv/sdk'

class DocumentSigner {
  private wallet: ProtoWallet
  
  constructor() {
    this.wallet = new ProtoWallet(PrivateKey.fromRandom())
  }
  
  async signDocument(content: string, documentId: string): Promise<{
    signature: number[]
    publicKey: string
    timestamp: number
  }> {
    const timestamp = Date.now()
    const documentData = {
      content,
      documentId,
      timestamp
    }
    
    const dataToSign = Utils.toArray(JSON.stringify(documentData), 'utf8')
    
    const { signature } = await this.wallet.createSignature({
      data: dataToSign,
      protocolID: [1, 'document system'],
      keyID: `doc-${documentId}`,
      counterparty: 'self'
    })
    
    const { publicKey } = await this.wallet.getPublicKey({
      protocolID: [1, 'document system'],
      keyID: `doc-${documentId}`
    })
    
    return { signature, publicKey, timestamp }
  }
  
  async verifyDocument(
    content: string,
    documentId: string,
    signature: number[],
    timestamp: number
  ): Promise<boolean> {
    const documentData = {
      content,
      documentId,
      timestamp
    }
    
    const dataToVerify = Utils.toArray(JSON.stringify(documentData), 'utf8')
    
    const { valid } = await this.wallet.verifySignature({
      data: dataToVerify,
      signature,
      protocolID: [1, 'document system'],
      keyID: `doc-${documentId}`,
      counterparty: 'self'
    })
    
    return valid
  }
}

async function demonstrateDocumentSigning() {
  const signer = new DocumentSigner()
  
  const document = {
    content: 'This is a confidential document requiring digital signature.',
    id: 'contract-2024-001'
  }
  
  // Sign document
  const signatureData = await signer.signDocument(document.content, document.id)
  console.log('Document signed:', {
    documentId: document.id,
    publicKey: signatureData.publicKey,
    timestamp: new Date(signatureData.timestamp).toISOString()
  })
  
  // Verify document
  const isValid = await signer.verifyDocument(
    document.content,
    document.id,
    signatureData.signature,
    signatureData.timestamp
  )
  
  console.log('Document verification:', isValid ? 'VALID' : 'INVALID')
  
  return { signatureData, isValid }
}

demonstrateDocumentSigning().catch(console.error)
```

## Encryption and Decryption

### Symmetric Encryption

```typescript
import { ProtoWallet, Utils } from '@bsv/sdk'

async function demonstrateEncryption() {
  const wallet = new ProtoWallet(PrivateKey.fromRandom())
  
  // Data to encrypt
  const secretMessage = 'This is confidential development data'
  const plaintext = Utils.toArray(secretMessage, 'utf8')
  
  // Encrypt data
  const { ciphertext } = await wallet.encrypt({
    plaintext,
    protocolID: [1, 'secure storage'],
    keyID: 'data-encryption-key'
  })
  
  console.log('Original message:', secretMessage)
  console.log('Encrypted (base64):', Utils.toBase64(ciphertext))
  
  // Decrypt data
  const { plaintext: decrypted } = await wallet.decrypt({
    ciphertext,
    protocolID: [1, 'secure storage'],
    keyID: 'data-encryption-key'
  })
  
  const decryptedMessage = Utils.toUTF8(decrypted)
  console.log('Decrypted message:', decryptedMessage)
  
  return { original: secretMessage, decrypted: decryptedMessage }
}

demonstrateEncryption().catch(console.error)
```

### Counterparty Encryption

```typescript
import { ProtoWallet, Utils } from '@bsv/sdk'

class SecureMessaging {
  private aliceWallet: ProtoWallet
  private bobWallet: ProtoWallet
  private aliceIdentity: string
  private bobIdentity: string
  
  constructor() {
    this.aliceWallet = new ProtoWallet(PrivateKey.fromRandom())
    this.bobWallet = new ProtoWallet(PrivateKey.fromRandom())
  }
  
  async initialize() {
    // Get identity keys for both parties
    const aliceKey = await this.aliceWallet.getPublicKey({ identityKey: true })
    const bobKey = await this.bobWallet.getPublicKey({ identityKey: true })
    
    this.aliceIdentity = aliceKey.publicKey
    this.bobIdentity = bobKey.publicKey
    
    console.log('Alice Identity:', this.aliceIdentity.substring(0, 20) + '...')
    console.log('Bob Identity:', this.bobIdentity.substring(0, 20) + '...')
  }
  
  async aliceSendsToBob(message: string): Promise<string> {
    const plaintext = Utils.toArray(message, 'utf8')
    
    const { ciphertext } = await this.aliceWallet.encrypt({
      plaintext,
      protocolID: [1, 'secure chat'],
      keyID: 'message-key',
      counterparty: this.bobIdentity
    })
    
    console.log('Alice encrypts message for Bob')
    return Utils.toBase64(ciphertext)
  }
  
  async bobReceivesFromAlice(ciphertext: string): Promise<string> {
    const ciphertextBytes = Utils.toArray(ciphertext, 'base64')
    
    const { plaintext } = await this.bobWallet.decrypt({
      ciphertext: ciphertextBytes,
      protocolID: [1, 'secure chat'],
      keyID: 'message-key',
      counterparty: this.aliceIdentity
    })
    
    const message = Utils.toUTF8(plaintext)
    console.log('Bob decrypts message from Alice')
    return message
  }
}

async function demonstrateSecureMessaging() {
  const messaging = new SecureMessaging()
  await messaging.initialize()
  
  const originalMessage = 'Hello Bob, this is a secure message from Alice!'
  
  // Alice encrypts and sends
  const encryptedMessage = await messaging.aliceSendsToBob(originalMessage)
  console.log('Encrypted message length:', encryptedMessage.length, 'bytes')
  
  // Bob receives and decrypts
  const decryptedMessage = await messaging.bobReceivesFromAlice(encryptedMessage)
  
  console.log('Original:', originalMessage)
  console.log('Decrypted:', decryptedMessage)
  console.log('Messages match:', originalMessage === decryptedMessage)
  
  return { originalMessage, decryptedMessage }
}

demonstrateSecureMessaging().catch(console.error)
```

## HMAC Operations

### Creating and Verifying HMACs

```typescript
import { ProtoWallet, Utils } from '@bsv/sdk'

async function demonstrateHMAC() {
  const wallet = new ProtoWallet(PrivateKey.fromRandom())
  
  // Data to authenticate
  const data = Utils.toArray('Important data requiring integrity verification', 'utf8')
  
  // Create HMAC
  const { hmac } = await wallet.createHmac({
    data,
    protocolID: [1, 'data integrity'],
    keyID: 'hmac-key'
  })
  
  console.log('Data:', Utils.toUTF8(data))
  console.log('HMAC:', Utils.toHex(hmac))
  
  // Verify HMAC
  const { valid } = await wallet.verifyHmac({
    data,
    hmac,
    protocolID: [1, 'data integrity'],
    keyID: 'hmac-key'
  })
  
  console.log('HMAC valid:', valid)
  
  // Test with tampered data
  const tamperedData = Utils.toArray('Tampered data requiring integrity verification', 'utf8')
  const { valid: tamperedValid } = await wallet.verifyHmac({
    data: tamperedData,
    hmac,
    protocolID: [1, 'data integrity'],
    keyID: 'hmac-key'
  })
  
  console.log('Tampered data HMAC valid:', tamperedValid)
  
  return { valid, tamperedValid }
}

demonstrateHMAC().catch(console.error)
```

## Building a Development Toolkit

### Complete ProtoWallet Utility Class

```typescript
import { ProtoWallet, PrivateKey, Utils } from '@bsv/sdk'

export class DevelopmentWallet {
  private wallet: ProtoWallet
  private identityKey: string | null = null
  
  constructor(privateKey?: PrivateKey) {
    this.wallet = new ProtoWallet(privateKey)
  }
  
  async getIdentity(): Promise<string> {
    if (!this.identityKey) {
      const { publicKey } = await this.wallet.getPublicKey({ identityKey: true })
      this.identityKey = publicKey
    }
    return this.identityKey
  }
  
  async signData(data: string, protocolId: string, keyId: string): Promise<{
    signature: string
    publicKey: string
    data: string
  }> {
    const dataBytes = Utils.toArray(data, 'utf8')
    
    const { signature } = await this.wallet.createSignature({
      data: dataBytes,
      protocolID: [1, protocolId],
      keyID: keyId,
      counterparty: 'self'
    })
    
    const { publicKey } = await this.wallet.getPublicKey({
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    return {
      signature: Utils.toBase64(signature),
      publicKey,
      data
    }
  }
  
  async verifyData(
    data: string,
    signature: string,
    protocolId: string,
    keyId: string
  ): Promise<boolean> {
    const dataBytes = Utils.toArray(data, 'utf8')
    const signatureBytes = Utils.toArray(signature, 'base64')
    
    const { valid } = await this.wallet.verifySignature({
      data: dataBytes,
      signature: signatureBytes,
      protocolID: [1, protocolId],
      keyID: keyId,
      counterparty: 'self'
    })
    
    return valid
  }
  
  async encryptForSelf(data: string, protocolId: string, keyId: string): Promise<string> {
    const plaintext = Utils.toArray(data, 'utf8')
    
    const { ciphertext } = await this.wallet.encrypt({
      plaintext,
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    return Utils.toBase64(ciphertext)
  }
  
  async decryptFromSelf(
    encryptedData: string,
    protocolId: string,
    keyId: string
  ): Promise<string> {
    const ciphertext = Utils.toArray(encryptedData, 'base64')
    
    const { plaintext } = await this.wallet.decrypt({
      ciphertext,
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    return Utils.toUTF8(plaintext)
  }
  
  async createDataIntegrityTag(data: string, protocolId: string, keyId: string): Promise<string> {
    const dataBytes = Utils.toArray(data, 'utf8')
    
    const { hmac } = await this.wallet.createHmac({
      data: dataBytes,
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    return Utils.toHex(hmac)
  }
  
  async verifyDataIntegrity(
    data: string,
    integrityTag: string,
    protocolId: string,
    keyId: string
  ): Promise<boolean> {
    const dataBytes = Utils.toArray(data, 'utf8')
    const hmac = Utils.toArray(integrityTag, 'hex')
    
    const { valid } = await this.wallet.verifyHmac({
      data: dataBytes,
      hmac,
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    return valid
  }
}

async function demonstrateDevelopmentToolkit() {
  const devWallet = new DevelopmentWallet()
  
  console.log('=== Development Wallet Toolkit Demo ===')
  
  // Get identity
  const identity = await devWallet.getIdentity()
  console.log('Wallet Identity:', identity.substring(0, 20) + '...')
  
  // Sign and verify data
  const testData = 'Development test data for signing'
  const signatureResult = await devWallet.signData(testData, 'dev-tools', 'test-key')
  console.log('Data signed successfully')
  
  const isValid = await devWallet.verifyData(
    testData,
    signatureResult.signature,
    'dev-tools',
    'test-key'
  )
  console.log('Signature verification:', isValid ? 'PASSED' : 'FAILED')
  
  // Encrypt and decrypt data
  const secretData = 'Confidential development information'
  const encrypted = await devWallet.encryptForSelf(secretData, 'dev-storage', 'secret-key')
  console.log('Data encrypted successfully')
  
  const decrypted = await devWallet.decryptFromSelf(encrypted, 'dev-storage', 'secret-key')
  console.log('Decryption result:', secretData === decrypted ? 'PASSED' : 'FAILED')
  
  // Create and verify integrity tag
  const importantData = 'Critical development configuration'
  const integrityTag = await devWallet.createDataIntegrityTag(
    importantData,
    'dev-integrity',
    'config-key'
  )
  console.log('Integrity tag created')
  
  const integrityValid = await devWallet.verifyDataIntegrity(
    importantData,
    integrityTag,
    'dev-integrity',
    'config-key'
  )
  console.log('Integrity verification:', integrityValid ? 'PASSED' : 'FAILED')
  
  return {
    identity,
    signatureValid: isValid,
    decryptionValid: secretData === decrypted,
    integrityValid
  }
}

demonstrateDevelopmentToolkit().catch(console.error)
```

## Testing Framework Integration

### ProtoWallet Test Utilities

```typescript
import { ProtoWallet, PrivateKey, Utils } from '@bsv/sdk'

export class ProtoWalletTestUtils {
  static createTestWallet(seed?: string): ProtoWallet {
    if (seed) {
      // Create deterministic wallet for testing
      const hash = Utils.toArray(seed, 'utf8')
      const privateKey = PrivateKey.fromString(Utils.toHex(hash).padEnd(64, '0'))
      return new ProtoWallet(privateKey)
    }
    return new ProtoWallet(PrivateKey.fromRandom())
  }
  
  static async createTestIdentities(count: number): Promise<{
    wallets: ProtoWallet[]
    identities: string[]
  }> {
    const wallets: ProtoWallet[] = []
    const identities: string[] = []
    
    for (let i = 0; i < count; i++) {
      const wallet = this.createTestWallet(`test-identity-${i}`)
      const { publicKey } = await wallet.getPublicKey({ identityKey: true })
      
      wallets.push(wallet)
      identities.push(publicKey)
    }
    
    return { wallets, identities }
  }
  
  static async testCryptographicRoundTrip(
    wallet: ProtoWallet,
    data: string,
    protocolId: string,
    keyId: string
  ): Promise<{
    signatureValid: boolean
    encryptionValid: boolean
    hmacValid: boolean
  }> {
    const dataBytes = Utils.toArray(data, 'utf8')
    
    // Test signature round trip
    const { signature } = await wallet.createSignature({
      data: dataBytes,
      protocolID: [1, protocolId],
      keyID: keyId,
      counterparty: 'self'
    })
    
    const { valid: signatureValid } = await wallet.verifySignature({
      data: dataBytes,
      signature,
      protocolID: [1, protocolId],
      keyID: keyId,
      counterparty: 'self'
    })
    
    // Test encryption round trip
    const { ciphertext } = await wallet.encrypt({
      plaintext: dataBytes,
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    const { plaintext } = await wallet.decrypt({
      ciphertext,
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    const encryptionValid = Utils.toUTF8(plaintext) === data
    
    // Test HMAC round trip
    const { hmac } = await wallet.createHmac({
      data: dataBytes,
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    const { valid: hmacValid } = await wallet.verifyHmac({
      data: dataBytes,
      hmac,
      protocolID: [1, protocolId],
      keyID: keyId
    })
    
    return { signatureValid, encryptionValid, hmacValid }
  }
}

async function runTestSuite() {
  console.log('=== ProtoWallet Test Suite ===')
  
  // Test deterministic wallet creation
  const wallet1 = ProtoWalletTestUtils.createTestWallet('test-seed-123')
  const wallet2 = ProtoWalletTestUtils.createTestWallet('test-seed-123')
  
  const identity1 = await wallet1.getPublicKey({ identityKey: true })
  const identity2 = await wallet2.getPublicKey({ identityKey: true })
  
  console.log('Deterministic wallet test:', 
    identity1.publicKey === identity2.publicKey ? 'PASSED' : 'FAILED')
  
  // Test multiple identities
  const { wallets, identities } = await ProtoWalletTestUtils.createTestIdentities(3)
  console.log('Created test identities:', identities.length)
  
  // Test cryptographic operations
  const testData = 'Test data for cryptographic operations'
  const results = await ProtoWalletTestUtils.testCryptographicRoundTrip(
    wallets[0],
    testData,
    'test-protocol',
    'test-key'
  )
  
  console.log('Cryptographic tests:')
  console.log('  Signature:', results.signatureValid ? 'PASSED' : 'FAILED')
  console.log('  Encryption:', results.encryptionValid ? 'PASSED' : 'FAILED')
  console.log('  HMAC:', results.hmacValid ? 'PASSED' : 'FAILED')
  
  return results
}

runTestSuite().catch(console.error)
```

## Conclusion

Congratulations! You've successfully built a comprehensive ProtoWallet development framework using the BSV TypeScript SDK. In this tutorial, you've learned how to create, test, and manage prototype wallet implementations for rapid development and testing.

### Core Concepts Mastered

1. **ProtoWallet Architecture**: Implemented lightweight wallet prototypes for development and testing
2. **Key Management**: Created deterministic key generation and management systems
3. **Cryptographic Operations**: Implemented signing, encryption, and HMAC operations
4. **Protocol Integration**: Built protocol-specific wallet functionality with proper key derivation
5. **Testing Framework**: Developed comprehensive testing utilities for wallet validation

## Next Steps

- Learn about [Development Wallet Setup](../guides/development-wallet-setup.md) for production-ready wallet implementation
- Explore [Key Management](./key-management.md) for advanced cryptographic key handling

## Additional Resources

- [Wallet API Reference](../reference/wallet.md)
- [Key Management Concepts](../concepts/key-management.md)
- [BSV Wallet Standards](https://projectbabbage.com/docs/guides/wallet/)
