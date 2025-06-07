# Transaction Signing Methods

*This is a How-To guide for different transaction signing approaches with the BSV TypeScript SDK.*

## Overview

This guide demonstrates two different approaches to signing Bitcoin transactions with the BSV TypeScript SDK:

1. **Using WalletClient** - A high-level approach that abstracts key management and signing details
2. **Using Low-level APIs** - A direct approach with more control over the transaction signing process

Each method has its advantages depending on your use case. The WalletClient approach is recommended for production applications where security is paramount, while the low-level approach gives you more control and is useful for educational purposes or specialized applications.

## Prerequisites

- Completed the [Key Management and Cryptography tutorial](../tutorials/key-management.md)
- Familiarity with Bitcoin transaction structure
- Understanding of basic cryptographic principles

## Method 1: Transaction Signing with WalletClient

The WalletClient provides a secure, high-level interface for managing keys and signing transactions. This approach is recommended for production applications as it:

- Abstracts away complex key management
- Provides better security by isolating private keys
- Handles transaction construction and signing in a unified way

### Example Code

```typescript
import { WalletClient, Transaction } from '@bsv/sdk'

async function walletTransactionDemo() {
  console.log('\n=== Transaction Signing with WalletClient ===')
  
  try {
    // 1. WalletClient Key Management
    const wallet = new WalletClient('auto', 'localhost')
    console.log('\n1. WalletClient Key Management')
    
    // Define protocol and key identifiers for wallet operations
    // Use 1 to represent medium security level
    // Cast it to any to bypass strict type checking since we don't have the SecurityLevel enum
    const protocolID = [1, 'example'] as any
    const keyID = 'transaction-signing-key'
    
    console.log(`Protocol ID: ${protocolID[0]}-${protocolID[1]}`)
    console.log(`Key ID: ${keyID}`)
    
    // Get a public key from the wallet
    // In a real application, this would be a key securely managed by the wallet
    const publicKeyResult = await wallet.getPublicKey({ protocolID, keyID })
    const publicKeyHex = publicKeyResult.publicKey
    console.log(`Public Key: ${publicKeyHex}`)
    
    // 2. Creating a transaction with WalletClient
    console.log('\n2. Creating a transaction with WalletClient')
    
    // Set up payment details
    const recipientAddress = '1DBz6V6CmvjZTvfjvJpfnrBk9Lf8fJ8dW8' // Example recipient
    const amountSatoshis = 10000
    
    // Create a payment action using WalletClient
    // This builds a complete transaction structure internally
    const actionResult = await wallet.createAction({
      description: `Payment to ${recipientAddress}`,
      // Define outputs for the transaction
      outputs: [
        {
          // In a real application, you would create a proper P2PKH script for the recipient
          lockingScript: '76a914eb0bd5edba389198e73f8efabddfc61666969ff788ac', // Example P2PKH script
          satoshis: amountSatoshis,
          outputDescription: `Payment to ${recipientAddress}`
        }
      ],
      // Set options to ensure we get a signable transaction
      options: {
        signAndProcess: false // This ensures we get a signable transaction back
      }
    })
    
    console.log('Payment action created:')
    if (actionResult.signableTransaction) {
      console.log(`- Action Reference: ${actionResult.signableTransaction.reference}`)
    } else {
      console.log('No signable transaction returned - check wallet configuration')
      return
    }
    console.log(`- Description: Payment to ${recipientAddress}`)
    console.log(`- Amount: ${amountSatoshis} satoshis`)
    
    // 3. Sign the transaction with WalletClient
    console.log('\n3. Signing transaction with WalletClient')
    
    // Request wallet to sign the action/transaction
    const signResult = await wallet.signAction({
      // Use the reference from the createAction result
      reference: actionResult.signableTransaction.reference,
      // The spends parameter is a map of input indexes to unlocking scripts
      // For wallet-managed keys, we provide an empty map and let the wallet handle it
      spends: {}
    })
    
    console.log('Transaction signed successfully!')
    if (signResult.txid) {
      console.log(`Transaction ID: ${signResult.txid}`)
    }
    
    // 4. Examine the transaction
    console.log('\n4. Examining the transaction')
    
    // Check if we have a transaction ID from the sign result
    if (signResult.txid) {
      console.log(`Transaction ID: ${signResult.txid}`)
      console.log('Transaction was successfully signed and broadcast!')
      
      // Note: In a real application, you would fetch the transaction details
      // from the blockchain using the txid to examine its structure
      
      // In a real application, you would examine the transaction structure here
      console.log('\nTo examine the transaction structure, you would:')
      console.log('1. Fetch the transaction from the blockchain using the txid')
      console.log('2. Parse the transaction to view inputs, outputs, and scripts')
      console.log('3. Verify signatures and validate the transaction')
      
      console.log('\nExample output information you would see:')
      console.log('- Input count: typically 1 or more inputs from your wallet')
      console.log('- Output count: at least 2 (payment + change)')
      console.log('- Input scripts: Contains signatures and public keys')
      console.log('- Output scripts: Contains P2PKH or other locking scripts')
    } else {
      console.log('No transaction ID available - transaction may not have been broadcast')
    }
  } catch (error) {
    console.error('Error during wallet transaction operations:', error)
  }
}

// Run the demo
walletTransactionDemo().catch(console.error)
```

### Key Benefits of the WalletClient Approach

1. **Security**: Private keys are managed by the wallet service, reducing exposure
2. **Abstraction**: Complex transaction construction details are handled internally
3. **Integration**: Designed for integration with secure key management systems
4. **Consistency**: Provides a standardized approach to transaction creation and signing

## Method 2: Transaction Signing with Low-level APIs

The low-level approach gives you direct control over the transaction signing process. This is useful for:

- Educational purposes to understand the underlying mechanics
- Specialized applications requiring custom transaction structures
- Situations where you need fine-grained control over the signing process

### Example Code

```typescript
import { PrivateKey, PublicKey, Transaction, P2PKH } from '@bsv/sdk'

async function transactionSigningDemo() {
  // Generate keys for our demo
  const privateKey = PrivateKey.fromRandom()
  const address = privateKey.toAddress()
  
  console.log('\n=== Keys for Transaction Signing ===')  
  console.log(`Private Key (WIF): ${privateKey.toWif()}`)
  console.log(`Address: ${address.toString()}`)
  
  // Create a new transaction
  const tx = new Transaction()
  
  // For demonstration, we'll add a dummy input
  // In a real scenario, this would be a reference to a UTXO
  // For our example, we'll create a simple transaction structure
  // In a real scenario, you would use actual UTXOs
  
  // First, create a dummy transaction that will serve as our input source
  const dummyTx = new Transaction()
  dummyTx.addOutput({
    lockingScript: new P2PKH().lock(address),
    satoshis: 10000
  })
  
  // Now add an input that references our dummy transaction
  tx.addInput({
    sourceTransaction: dummyTx,  // Reference to the dummy transaction
    sourceOutputIndex: 0,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey)
  })
  
  // Add an output
  tx.addOutput({
    lockingScript: new P2PKH().lock(address),
    satoshis: 10000
  })
  
  console.log('\n=== Transaction Before Signing ===')  
  console.log(`Input Count: ${tx.inputs.length}`)
  console.log(`Output Count: ${tx.outputs.length}`)
  console.log(`First input has unlocking script: ${tx.inputs[0].unlockingScript ? 'Yes' : 'No'}`)
  console.log(`First input has unlocking script template: ${tx.inputs[0].unlockingScriptTemplate ? 'Yes' : 'No'}`)
  
  // Now, sign the transaction
  await tx.sign()
  
  console.log('\n=== Transaction After Signing ===')  
  console.log(`Transaction ID: ${Buffer.from(tx.id()).toString('hex')}`)
  console.log(`First input has unlocking script: ${tx.inputs[0].unlockingScript ? 'Yes' : 'No'}`)
  
  // Let's look at the unlocking script (scriptSig) that contains the signature
  if (tx.inputs[0].unlockingScript) {
    console.log(`\nUnlocking Script (ASM): ${tx.inputs[0].unlockingScript.toASM()}`)
  }
  
  // Serialize the transaction to hex
  const txHex = tx.toHex()
  console.log(`\nSigned Transaction (hex, first 64 chars): ${txHex.substring(0, 64)}...`)
  
  // Verify the signature(s) in the transaction
  const isValid = await tx.verify()
  console.log(`\nTransaction signature verification: ${isValid ? 'Valid ✓' : 'Invalid ✗'}`)
  console.log('\nNote: The verification shows as invalid because this is a simplified example.')
  console.log('In real transactions, proper UTXOs and transaction validation would be required.')
}

// Run our transaction signing demo
transactionSigningDemo().catch(console.error)
```

### Key Benefits of the Low-level Approach

1. **Control**: Direct control over every aspect of the transaction
2. **Transparency**: Clear visibility into the transaction structure and signing process
3. **Flexibility**: Ability to customize transaction construction for specialized use cases
4. **Educational Value**: Better understanding of the underlying Bitcoin transaction mechanics

## Choosing the Right Approach

Consider the following factors when deciding which approach to use:

| Factor | WalletClient Approach | Low-level Approach |
|--------|----------------------|-------------------|
| Security | Higher (keys managed by wallet) | Lower (direct key handling) |
| Complexity | Lower (abstracted API) | Higher (manual transaction construction) |
| Control | Limited (managed by wallet) | Complete (direct access) |
| Use Case | Production applications | Educational, specialized applications |
| Integration | Better for enterprise systems | Better for custom implementations |

## Best Practices

Regardless of which approach you choose, follow these best practices:

1. **Never expose private keys**: Keep private keys secure and never expose them in logs or user interfaces
2. **Test thoroughly**: Always test transaction signing in a test environment before production
3. **Verify signatures**: Always verify signatures after signing to ensure transaction validity
4. **Handle errors gracefully**: Implement proper error handling for signing failures
5. **Consider SIGHASH flags**: Use appropriate signature hash types for your use case
6. **Document key management**: Maintain clear documentation of your key management approach

## Related Resources

- [Key Management and Cryptography Tutorial](../tutorials/key-management.md)
- [Advanced Transaction Signing](./advanced-transaction-signing.md)
- [Transaction Signatures Reference](../reference/transaction-signatures.md)
- [Bitcoin Transaction Specification](https://reference.cash/protocol/blockchain/transaction)
