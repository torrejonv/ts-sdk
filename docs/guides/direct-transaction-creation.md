# Creating Transactions with Direct Interfaces

This guide demonstrates how to create Bitcoin SV transactions using the lower-level direct interfaces provided by the BSV TypeScript SDK. This approach gives you more control over transaction construction and is useful for specialized use cases where the WalletClient abstraction isn't suitable.

## When to Use Direct Interfaces

- When creating custom transaction types not supported by WalletClient
- When you need precise control over UTXO selection
- When building specialized applications like data storage services that require custom optimization
- When integrating with systems that require direct management of transactions
- For educational purposes to understand the underlying transaction structure

## Basic Transaction Construction

```typescript
import { PrivateKey, P2PKH, Transaction } from '@bsv/sdk'

async function createBasicTransaction() {
  // Create a private key
  const privateKey = PrivateKey.fromRandom()
  console.log(`Private key WIF: ${privateKey.toWif()}`)
  
  // Derive the address
  const address = privateKey.toAddress()
  console.log(`Address: ${address.toString()}`)
  
  // Create a new transaction
  const tx = new Transaction()
  
  // Add an output
  tx.addOutput({
    lockingScript: new P2PKH().lock(address),
    satoshis: 1000
  })
  
  // Serialize the transaction
  const txHex = tx.toHex()
  console.log(`Transaction (hex): ${txHex}`)
  
  // Get transaction ID as a hex string
  const txid = Buffer.from(tx.id()).toString('hex')
  console.log(`Transaction ID: ${txid}`)
}
```

## Complete Transaction with Inputs and Outputs

For a complete transaction that can be broadcast, you need to add inputs, calculate fees, and sign it:

```typescript
import { Transaction, PrivateKey, P2PKH, ARC } from '@bsv/sdk'

async function createCompleteTransaction() {
  // Set up your wallet
  const privateKey = PrivateKey.fromWif('your_private_key_here')
  const myAddress = privateKey.toAddress()
  const recipientAddress = 'recipient_address_here'

  // You need the hex of the source transaction
  const sourceTxHex = '...' // Hex string of the source transaction
  
  // Create a transaction
  const tx = new Transaction()
  
  // Add the input
  tx.addInput({
    sourceTransaction: Transaction.fromHex(sourceTxHex),
    sourceOutputIndex: 0, // The output index you want to spend
    unlockingScriptTemplate: new P2PKH().unlock(privateKey)
  })
  
  // Add the recipient output
  tx.addOutput({
    lockingScript: new P2PKH().lock(recipientAddress),
    satoshis: 5000 // Amount to send
  })
  
  // Add the change output back to our address
  tx.addOutput({
    lockingScript: new P2PKH().lock(myAddress),
    change: true // SDK will automatically calculate the change amount
  })
  
  // Calculate fee and sign the transaction
  await tx.fee()
  await tx.sign()
  
  // Get the transaction hex ready for broadcasting
  const signedTxHex = tx.toHex()
  console.log(`Signed transaction hex: ${signedTxHex}`)
  
  // Get the transaction ID
  const txid = Buffer.from(tx.id()).toString('hex')
  console.log(`Transaction ID: ${txid}`)
}
```

## Broadcasting the Transaction

To broadcast your signed transaction:

```typescript
import { ARC, NodejsHttpClient } from '@bsv/sdk'
import https from 'https'

async function broadcastTransaction(signedTx) {
  // Create an HTTP client
  const httpClient = new NodejsHttpClient(https)
  
  // Initialize the ARC client
  const arc = new ARC('https://api.arc.taal.com', {
    apiKey: 'your_api_key_here',
    httpClient,
    deploymentId: 'your-deployment-id'
  })
  
  // Broadcast the transaction
  const result = await signedTx.broadcast(arc)
  console.log('Broadcast result:', result)
}
```

## Key Implementation Details

When working with direct interfaces, remember these important details:

1. Use `toWif()` (lowercase 'f') not `toWIF()` for private key WIF format
2. Use `toHex()` instead of `toString()` for transaction serialization
3. Transaction IDs need to be converted from byte arrays: `Buffer.from(tx.id()).toString('hex')`
4. For script objects, use `toHex()` or `toASM()` rather than `toString()`
5. Method chaining doesn't work well with current API - use separate method calls

## Related Resources

- For simpler implementations, see the [Creating Transactions with WalletClient](../tutorials/first-transaction.md) tutorial
- Learn about [Advanced Transaction Signing](./advanced-transaction-signing.md)
- Explore [HTTP Client Configuration](./http-client-configuration.md) for optimizing API requests
