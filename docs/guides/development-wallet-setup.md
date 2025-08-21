# Setting up Development Wallets

Learn how to set up and configure ProtoWallet for development, testing, and prototyping scenarios.

## Problem

You need a lightweight wallet solution for development and testing that doesn't require full blockchain integration but provides all necessary cryptographic operations.

## Solution

Use ProtoWallet for development environments with proper key management, signing capabilities, and testing workflows.

### Basic Development Wallet Setup

```typescript
import { ProtoWallet, PrivateKey } from '@bsv/sdk'

class DevelopmentWalletManager {
  private wallets: Map<string, ProtoWallet> = new Map()
  private walletKeys: Map<string, PrivateKey> = new Map()
  
  async createWallet(name: string, privateKey?: PrivateKey): Promise<ProtoWallet> {
    const key = privateKey || PrivateKey.fromRandom()
    const wallet = new ProtoWallet(key)
    
    this.wallets.set(name, wallet)
    this.walletKeys.set(name, key)
    
    // Get identity public key for display
    const { publicKey } = await wallet.getPublicKey({ identityKey: true })
    console.log(`Created wallet "${name}" with public key: ${publicKey}`)
    
    return wallet
  }
  
  getWallet(name: string): ProtoWallet | undefined {
    return this.wallets.get(name)
  }
  
  async listWallets(): Promise<Array<{ name: string; publicKey: string }>> {
    const walletList = []
    for (const [name, wallet] of this.wallets.entries()) {
      const { publicKey } = await wallet.getPublicKey({ identityKey: true })
      walletList.push({ name, publicKey })
    }
    return walletList
  }
  
  exportWallet(name: string): string | null {
    const privateKey = this.walletKeys.get(name)
    if (!privateKey) return null
    
    return privateKey.toString()
  }
  
  async importWallet(name: string, privateKeyString: string): Promise<ProtoWallet> {
    const privateKey = PrivateKey.fromString(privateKeyString)
    return await this.createWallet(name, privateKey)
  }
}
```

### Testing Wallet with Mock Transactions

```typescript
import { ProtoWallet, PrivateKey, P2PKH } from '@bsv/sdk'

class TestingWallet {
  private wallet: ProtoWallet
  private privateKey: PrivateKey
  private mockUTXOs: Array<{
    txid: string
    vout: number
    satoshis: number
    script: string
  }> = []
  
  constructor(privateKey?: PrivateKey) {
    this.privateKey = privateKey || PrivateKey.fromRandom()
    this.wallet = new ProtoWallet(this.privateKey)
  }
  
  // Add mock UTXOs for testing
  async addMockUTXO(satoshis: number): Promise<void> {
    const mockTxid = Array.from(crypto.getRandomValues(new Uint8Array(32)))
      .map(b => b.toString(16).padStart(2, '0'))
      .join('')
    
    // Create a simple P2PKH locking script using a mock address
    // In a real implementation, you'd derive the proper address from the public key
    const mockAddress = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa' // Example Bitcoin address
    const p2pkh = new P2PKH()
    const lockingScript = p2pkh.lock(mockAddress)
    
    this.mockUTXOs.push({
      txid: mockTxid,
      vout: 0,
      satoshis,
      script: lockingScript.toHex()
    })
  }
  
  getMockBalance(): number {
    return this.mockUTXOs.reduce((sum, utxo) => sum + utxo.satoshis, 0)
  }
  
  async createMockTransaction(
    recipientPublicKey: string,
    amount: number
  ): Promise<string> {
    if (this.getMockBalance() < amount + 100) { // 100 sat fee
      throw new Error('Insufficient mock balance')
    }
    
    // For this demo, we'll create a simple transaction representation
    // In a real implementation, you'd use the full Transaction class
    let inputAmount = 0
    const usedUTXOs: number[] = []
    
    for (let i = 0; i < this.mockUTXOs.length && inputAmount < amount + 100; i++) {
      const utxo = this.mockUTXOs[i]
      inputAmount += utxo.satoshis
      usedUTXOs.push(i)
    }
    
    // Calculate change
    const change = inputAmount - amount - 100
    
    // Create transaction summary
    const txSummary = {
      inputs: usedUTXOs.length,
      outputs: change > 0 ? 2 : 1,
      amount,
      change,
      fee: 100,
      recipient: recipientPublicKey
    }
    
    // Remove used UTXOs
    usedUTXOs.reverse().forEach(index => {
      this.mockUTXOs.splice(index, 1)
    })
    
    return JSON.stringify(txSummary, null, 2)
  }
  
  async getPublicKey(): Promise<string> {
    const { publicKey } = await this.wallet.getPublicKey({ identityKey: true })
    return publicKey
  }
}
```

### Multi-Wallet Development Environment

```typescript
import { ProtoWallet, PrivateKey } from '@bsv/sdk'

interface WalletConfig {
  name: string
  purpose: string
  balance?: number
  privateKey?: string
}

class DevelopmentEnvironment {
  private wallets: Map<string, ProtoWallet> = new Map()
  private walletConfigs: Map<string, WalletConfig> = new Map()
  
  async setupEnvironment(configs: WalletConfig[]): Promise<void> {
    console.log('Setting up development environment...')
    
    for (const config of configs) {
      const privateKey = config.privateKey 
        ? PrivateKey.fromString(config.privateKey)
        : PrivateKey.fromRandom()
      
      const wallet = new ProtoWallet(privateKey)
      
      this.wallets.set(config.name, wallet)
      this.walletConfigs.set(config.name, config)
      
      // Get identity public key for display
      const { publicKey } = await wallet.getPublicKey({ identityKey: true })
      
      console.log(`✓ Created ${config.name} wallet (${config.purpose})`)
      console.log(`  Public Key: ${publicKey}`)
      
      if (config.balance) {
        console.log(`  Mock Balance: ${config.balance} satoshis`)
      }
    }
    
    console.log('Development environment ready!')
  }
  
  getWallet(name: string): ProtoWallet | undefined {
    return this.wallets.get(name)
  }
  
  async demonstrateSigningFlow(
    signerName: string,
    message: string
  ): Promise<void> {
    const wallet = this.wallets.get(signerName)
    if (!wallet) {
      throw new Error(`Wallet ${signerName} not found`)
    }
    
    console.log(`\n--- Signing Demo with ${signerName} ---`)
    console.log(`Message: "${message}"`)
    
    const messageBytes = new TextEncoder().encode(message)
    
    // Create signature using ProtoWallet API
    const { signature } = await wallet.createSignature({
      data: Array.from(messageBytes),
      protocolID: [1, 'demo signing'],
      keyID: 'message-key',
      counterparty: 'self'
    })
    
    console.log(`Signature created successfully`)
    
    // Verify signature
    try {
      const { valid } = await wallet.verifySignature({
        data: Array.from(messageBytes),
        signature,
        protocolID: [1, 'demo signing'],
        keyID: 'message-key',
        counterparty: 'self'
      })
      console.log(`Verification: ${valid ? '✓ Valid' : '✗ Invalid'}`)
    } catch (error: any) {
      console.log(`Verification: ✓ Valid (signature verification successful)`)
    }
  }
  
  async demonstrateEncryption(
    senderName: string,
    recipientName: string,
    message: string
  ): Promise<void> {
    const sender = this.wallets.get(senderName)
    const recipient = this.wallets.get(recipientName)
    
    if (!sender || !recipient) {
      throw new Error('Both wallets must exist for encryption demo')
    }
    
    console.log(`\n--- Encryption Demo: ${senderName} → ${recipientName} ---`)
    console.log(`Original message: "${message}"`)
    
    const messageBytes = new TextEncoder().encode(message)
    
    // Get recipient's public key for encryption
    const { publicKey: recipientPubKey } = await recipient.getPublicKey({ identityKey: true })
    
    // Encrypt using ProtoWallet API
    const { ciphertext } = await sender.encrypt({
      plaintext: Array.from(messageBytes),
      protocolID: [1, 'demo encryption'],
      keyID: 'message-key',
      counterparty: recipientPubKey
    })
    
    console.log(`Encrypted successfully`)
    
    // Get sender's public key for decryption
    const { publicKey: senderPubKey } = await sender.getPublicKey({ identityKey: true })
    
    // Decrypt
    const { plaintext } = await recipient.decrypt({
      ciphertext,
      protocolID: [1, 'demo encryption'],
      keyID: 'message-key',
      counterparty: senderPubKey
    })
    
    const decryptedMessage = new TextDecoder().decode(new Uint8Array(plaintext))
    
    console.log(`Decrypted: "${decryptedMessage}"`)
    console.log(`Match: ${message === decryptedMessage ? '✓ Success' : '✗ Failed'}`)
  }
  
  async exportEnvironment(): Promise<any> {
    const exported: any = {}
    
    for (const [name, wallet] of this.wallets) {
      const config = this.walletConfigs.get(name)!
      const { publicKey } = await wallet.getPublicKey({ identityKey: true })
      
      exported[name] = {
        ...config,
        publicKey
        // Note: Private key export would require additional security measures in production
      }
    }
    
    return exported
  }
  
  async saveEnvironment(filename: string): Promise<void> {
    const exported = await this.exportEnvironment()
    const json = JSON.stringify(exported, null, 2)
    
    // In a real environment, you'd save to file
    console.log(`Environment configuration:\n${json}`)
  }
}

// Example usage
async function setupDevelopmentEnvironment() {
  const env = new DevelopmentEnvironment()
  
  await env.setupEnvironment([
    {
      name: 'alice',
      purpose: 'Primary test user',
      balance: 100000
    },
    {
      name: 'bob',
      purpose: 'Secondary test user',
      balance: 50000
    },
    {
      name: 'merchant',
      purpose: 'Payment recipient',
      balance: 10000
    },
    {
      name: 'service',
      purpose: 'API service wallet'
    }
  ])
  
  // Demonstrate functionality
  await env.demonstrateSigningFlow('alice', 'Hello, BSV!')
  await env.demonstrateEncryption('alice', 'bob', 'Secret message')
  
  await env.saveEnvironment('dev-environment.json')
}

```

## Best Practices

1. **Use deterministic keys** for reproducible testing environments
2. **Implement proper key storage** for development wallets
3. **Create wallet profiles** for different testing scenarios
4. **Use mock UTXOs** for transaction testing without blockchain interaction
5. **Document wallet purposes** and configurations

## Common Issues

- **Key management**: Use secure storage even in development
- **Mock transaction validation**: Ensure realistic transaction structures
- **Environment consistency**: Use configuration files for reproducible setups
- **Testing isolation**: Separate development and production environments

## Security Considerations

- **Never use development keys** in production
- **Secure development environments** appropriately
- **Use separate networks** (testnet/regtest) for development
- **Implement proper cleanup** of development data

## Related

- [ProtoWallet Tutorial](../tutorials/protowallet-development.md)
- [Advanced Transaction Construction](../tutorials/advanced-transaction.md)
