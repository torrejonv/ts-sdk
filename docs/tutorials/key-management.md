# Key Management and Cryptography

**Duration**: 45 minutes  
**Prerequisites**: Completed "Your First BSV Transaction" tutorial, Node.js, basic TypeScript knowledge  

## Learning Goals

- Generate and manage private/public keys
- Understand ECDSA signatures
- Create and verify digital signatures
- Apply secure key management practices
- Use `WalletClient` for advanced key operations

> **📚 Related Concepts**: Review [Key Management](../concepts/key-management.md) and [Digital Signatures](../concepts/signatures.md) for essential background.

## Introduction

Bitcoin is built on cryptographic principles, with keys and signatures forming the foundation of its security model. In this tutorial, you'll learn how to generate, manage, and use cryptographic keys with the BSV TypeScript SDK. You'll also learn how to create and verify digital signatures, which are essential for authorizing transactions and proving ownership.

> **💡 Try It Interactive**: Experiment with key generation and cryptographic operations in our [Interactive BSV Coding Environment](https://fast.brc.dev/) - perfect for testing the concepts covered in this tutorial!

## Step 1: Setting Up Your Environment

First, let's create a project for our key management exercises:

```bash
# Create a new directory for the project
mkdir bsv-key-management
cd bsv-key-management

# Initialize a new Node.js project
npm init -y

# Install TypeScript and ts-node
npm install typescript ts-node @types/node --save-dev

# Install the BSV SDK
npm install @bsv/sdk
```

Create a basic TypeScript configuration file (`tsconfig.json`):

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "commonjs",
    "esModuleInterop": true,
    "strict": true,
    "outDir": "./dist"
  }
}
```

## Step 2: Understanding Bitcoin Keys

Before diving into code, let's understand the key concepts:

### Key Hierarchy

Bitcoin uses a hierarchical key system:

- **Private Key**: A randomly generated number that must be kept secret
- **Public Key**: Derived from the private key, can be shared safely
- **Bitcoin Address**: Derived from the public key, used to receive funds

### Key Formats

Private keys can be represented in several formats:

- **Raw**: A 32-byte number
- **WIF (Wallet Import Format)**: A Base58Check encoded string, making keys easier to handle
- **Extended Keys**: Used in HD wallets (covered in advanced tutorials)

Public keys can be represented as:

- **Compressed**: 33 bytes (more efficient, preferred format)
- **Uncompressed**: 65 bytes (legacy format)

Addresses can be in various formats:

- **P2PKH**: Standard "Pay to Public Key Hash" addresses
- **P2SH**: "Pay to Script Hash" addresses for more complex scripts
- **Others**: Various formats exist for specific use cases

## Step 3: Generating and Managing Keys

Let's create a file called `key-management.ts` to experiment with key generation and management:

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

// Generate a new random private key
function generateNewKey() {
  const privateKey = PrivateKey.fromRandom()
  const publicKey = privateKey.toPublicKey()
  const address = privateKey.toAddress()
  
  console.log('\n=== Newly Generated Key ===')  
  console.log(`Private Key (WIF): ${privateKey.toWif()}`)
  console.log(`Public Key (DER Hex): ${publicKey.toDER('hex')}`)
  console.log(`Bitcoin Address: ${address.toString()}`)
  
  return privateKey
}

// Import an existing private key from WIF format
function importFromWIF(wifString: string) {
  try {
    const privateKey = PrivateKey.fromWif(wifString)
    const publicKey = privateKey.toPublicKey()
    const address = privateKey.toAddress()
    
    console.log('\n=== Imported Key ===')  
    console.log(`Private Key (WIF): ${privateKey.toWif()}`)
    console.log(`Public Key (DER Hex): ${publicKey.toDER('hex')}`)
    console.log(`Bitcoin Address: ${address.toString()}`)
    
    return privateKey
  } catch (error) {
    console.error('Error importing key:', error)
    return null
  }
}

// Derive different address types from a private key
function deriveAddressTypes(privateKey: PrivateKey) {
  // Standard P2PKH mainnet address (prefix 0x00)
  const mainnetAddress = privateKey.toAddress()
  
  console.log('\n=== Bitcoin Address Types ===')  
  console.log(`Mainnet Address: ${mainnetAddress.toString()}`)
  
  // Get the public key and hash it to show the process
  const publicKey = privateKey.toPublicKey()
  // For P2PKH addresses, we use HASH160 (RIPEMD160(SHA256(pubKey)))
  const pubKeyHash = publicKey.toHash()
  console.log(`Public Key Hash: ${Buffer.from(pubKeyHash).toString('hex')}`)
  
  return mainnetAddress
}

// Advanced: Check if a public key corresponds to a private key
function verifyKeyPair(privateKey: PrivateKey, publicKeyHex: string) {
  // Convert the provided public key hex to a PublicKey object
  // The toDER('hex') method provides a hex string that can be parsed by fromString
  const providedPubKey = PublicKey.fromString(publicKeyHex)
  
  // Derive the public key from the private key
  const derivedPubKey = privateKey.toPublicKey()
  
  // Compare the hex representations
  // Make sure we cast to string to ensure proper comparison
  const isMatch = (providedPubKey.toDER('hex') as string) === (derivedPubKey.toDER('hex') as string)
  
  console.log('\n=== Key Pair Verification ===')  
  console.log(`Public keys match: ${isMatch}`)
  
  return isMatch
}

// Execute our key management examples
async function runKeyManagementExamples() {
  // Generate a new key
  const newKey = generateNewKey()
  
  // Derive different address types from the key
  deriveAddressTypes(newKey)
  
  // Import a key from WIF (using the one we just generated as an example)
  const wif = newKey.toWif()
  const importedKey = importFromWIF(wif)
  
  if (importedKey) {
    // Verify the key pair
    // Make sure we're using a string type for the public key hex
    const pubKeyHex = newKey.toPublicKey().toDER('hex') as string
    verifyKeyPair(importedKey, pubKeyHex)
  }
  
  console.log('\n=== Key Management Demo Complete ===')  
}

// Run our examples
runKeyManagementExamples().catch(console.error)
```

Run the script with:

```bash
npx ts-node key-management.ts
```

You should see output showing the generated keys, addresses, and verification results.

## Step 4: Creating and Verifying Digital Signatures

Digital signatures are fundamental to Bitcoin. They prove that the owner of a private key has authorized a specific action, like spending coins in a transaction.

### Approach 1: Using `WalletClient`

`WalletClient` is the recommended interface for these actions, providing enhanced security as private keys remain isolated within the wallet environment. Let's create a file called `signatures-wallet.ts`:

```typescript
import { WalletClient } from '@bsv/sdk'

async function signatureWalletExamples() {
  // Initialize a WalletClient with default settings
  const wallet = new WalletClient('auto', 'localhost')
  
  console.log('\n=== WalletClient Signature Example ===')
  
  try {
    // Connect to the wallet substrate
    await wallet.connectToSubstrate()
    
    // 1. Define protocol and key identifiers for wallet operations
    // In a real app, these would be specific to your application
    // Using 1 to represent medium security level
    // Using 'any' type to bypass type checking since we don't have access to the SecurityLevel enum values
    const protocolID = [1, 'bsv tutorial'] as any
    const keyID = 'tutorial signing key'
    
    // 2. Get a public key from the wallet for verification (for demo purposes)
    const keyResult = await wallet.getPublicKey({
      protocolID,
      keyID,
      counterparty: 'self' // Get our own public key
    })
    
    console.log(`Public Key from Wallet: ${keyResult.publicKey}`)
    
    // 3. Create a message to sign
    const message = 'Hello, Bitcoin SV!'
    console.log(`\nMessage to sign: "${message}"`)
    const messageBytes = new TextEncoder().encode(message)
    
    // 4. Create a signature using WalletClient
    console.log('\nCreating signature with parameters:')
    console.log('- Protocol ID:', JSON.stringify(protocolID))
    console.log('- Key ID:', keyID)
    console.log('- Message bytes:', JSON.stringify(Array.from(messageBytes)))
    
    // When creating signatures with counterparty='self', we must explicitly set it
    // This ensures we can verify the signature with the default parameters
    const sigResult = await wallet.createSignature({
      data: Array.from(messageBytes),
      protocolID,
      keyID,
      counterparty: 'self' // Explicitly use 'self' as counterparty, as the default counterparty is 'anyone' (implicit)
    })
    
    console.log(`\nSignature created with WalletClient: ${Buffer.from(sigResult.signature).toString('hex').substring(0, 64)}...`)
    
    // 5. Verify the signature using WalletClient
    // Note: WalletClient throws an error when verification fails
    try {
      console.log('\nVerifying signature with parameters:')
      console.log('- Protocol ID:', JSON.stringify(protocolID))
      console.log('- Key ID:', keyID)
      console.log('- Message bytes:', JSON.stringify(Array.from(messageBytes)))
      
      // When verifying signatures, the default counterparty is 'self'
      // Since we created the signature with counterparty='self', we can use the default
      const verifyResult = await wallet.verifySignature({
        data: Array.from(messageBytes),
        signature: sigResult.signature,
        protocolID,
        keyID
        // Using default counterparty: 'self' (implicit)
        // counterparty: ourPublicKey 
      })
      
      console.log(`\nSignature verification result: ${verifyResult.valid ? 'Valid ✓' : 'Invalid ✗'}`)
    } catch (error) {
      // The wallet throws an error when verification fails instead of returning { valid: false }
      console.log('\nSignature verification result: Invalid ✗')
      console.log(`Verification error: ${error instanceof Error ? error.message : 'Unknown error'}`)
    }
    
    // 6. Try verifying with tampered data
    const tamperedMessage = 'Hello, Bitcoin SV! [tampered]'
    const tamperedBytes = new TextEncoder().encode(tamperedMessage)
    
    try {
      // For tampered message verification, we use the same parameters
      const tamperedVerifyResult = await wallet.verifySignature({
        data: Array.from(tamperedBytes),
        signature: sigResult.signature,
        protocolID,
        keyID
        // Using default counterparty: 'self' (implicit)
        // counterparty: ourPublicKey 
      })
      
      console.log(`\nTampered message verification: ${tamperedVerifyResult.valid ? 'Valid ✓' : 'Invalid ✗'}`)
    } catch (error) {
      // Expected behavior: verification should fail with tampered data
      console.log('\nTampered message verification: Invalid ✗')
      console.log('This is the expected behavior - tampered data should fail verification')
    }
    
  } catch (error) {
    console.error('\nError during WalletClient operations:', error)
    console.log('Note: To use WalletClient, you need a compatible wallet connection.')
  }
}

// Run our wallet signature examples
signatureWalletExamples().catch(console.error)
```

Run the example:

```bash
npx ts-node signatures-wallet.ts
```

### Approach 2: Using Low-level Cryptography APIs

Alternatively, you could perform the same using direct cryptography APIs. Let's create a file called `signatures-low-level.ts`:

```typescript
import { PrivateKey, PublicKey, Signature } from '@bsv/sdk'

async function signatureLowLevelExamples() {
  // Generate a key to use for signing
  const privateKey = PrivateKey.fromRandom()
  const publicKey = privateKey.toPublicKey()
  
  console.log('\n=== Key for Signing (Low-level API) ===')  
  console.log(`Private Key (WIF): ${privateKey.toWif()}`)
  console.log(`Public Key (DER Hex): ${publicKey.toDER('hex')}`)
  
  // 1. Create a message to sign
  const message = 'Hello, Bitcoin SV!'
  console.log(`\nMessage to sign: "${message}"`)
  
  // 2. Sign the message - using the message string directly
  const signature = await privateKey.sign(message)
  
  // Get the signature in DER format (hex string)
  const derSignatureHex = signature.toDER('hex') as string
  console.log(`\nSignature (DER format): ${derSignatureHex}`)
  
  // 3. Verify the signature using the public key
  const isValid = await publicKey.verify(message, signature)
  console.log(`\nSignature verification result: ${isValid ? 'Valid ✓' : 'Invalid ✗'}`)
  
  // 4. Try verifying with a modified message (should fail)
  const tamperedMessage = message + ' [tampered]'
  const isTamperedValid = await publicKey.verify(tamperedMessage, signature)
  console.log(`\nTampered message verification: ${isTamperedValid ? 'Valid ✓' : 'Invalid ✗'}`)
  
  // 5. Try verifying with a different public key (should fail)
  const differentKey = PrivateKey.fromRandom().toPublicKey()
  const isDifferentKeyValid = await differentKey.verify(message, signature)
  console.log(`\nWrong key verification: ${isDifferentKeyValid ? 'Valid ✓' : 'Invalid ✗'}`)
  
  // 6. Importing a signature from DER format (as number array)
  const derSignature = signature.toDER() as number[]
  const importedSignature = Signature.fromDER(derSignature)
  const isImportedValid = await publicKey.verify(message, importedSignature)
  console.log(`\nImported signature verification: ${isImportedValid ? 'Valid ✓' : 'Invalid ✗'}`)
}

// Run our signature examples
signatureLowLevelExamples().catch(console.error)
```

Run the script:

```bash
npx ts-node signatures-low-level.ts
```

### Key Benefits of `WalletClient` for Signatures

1. **Enhanced Security**: Private keys never leave the wallet environment
2. **Key Management**: No need to handle raw private keys in your code
3. **Standardized API**: Consistent interface for all cryptographic operations
4. **Protocol-based**: Keys are managed within specific protocol contexts

## Step 5: Practical Application: Signing Transactions with `WalletClient`

Let's put our knowledge to practical use by creating and signing a Bitcoin transaction using the `WalletClient`.

Create a file called `wallet-transaction-signing.ts`:

```typescript
import { WalletClient, Transaction } from '@bsv/sdk'

async function walletTransactionDemo() {
  console.log('\n=== Transaction Signing with WalletClient ===')
  
  try {
    // 1. WalletClient Key Management
    // Note: This tutorial requires a BSV wallet to be installed and available
    // If you get connection errors, you may need to install a compatible BSV wallet
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
    const amountSatoshis = 100
    
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
      console.log(`- Transaction available: ${!!actionResult.signableTransaction.tx}`)
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
      // For wallet-managed transactions, we can let the wallet handle unlocking scripts
      spends: {},
      // Add options to ensure proper handling
      options: {
        acceptDelayedBroadcast: true,
        returnTXIDOnly: false,
        noSend: true // Don't broadcast automatically for this tutorial
      }
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
      console.log('Transaction was successfully signed!')
    } else {
      console.log('No transaction ID available - transaction may not have been completed')
    }
    
  } catch (error) {
    console.error('Error during wallet transaction operations:', error)
  }
}

// Run the demo
walletTransactionDemo().catch(console.error)
```

Run the script:

```bash
npx ts-node wallet-transaction-signing.ts
```

This example demonstrates:

1. Creating a transaction with inputs and outputs
2. Getting the transaction hash that needs to be signed
3. How the `WalletClient` would sign this hash securely
4. Verifying the transaction signature
5. The complete `WalletClient` workflow for real applications

For a detailed comparison between `WalletClient` transaction signing and low-level transaction signing approaches, see the [Transaction Signing Methods guide](../guides/transaction-signing-methods.md).

### Advanced Transaction Signing

For more advanced transaction signing techniques like using different SIGHASH flags, manual signature creation, and multi-signature transactions, please refer to the [Advanced Transaction Signing guide](../guides/advanced-transaction-signing.md).

## Conclusion

Congratulations! You've learned the fundamentals of key management and cryptography with the BSV TypeScript SDK. In this tutorial, you've:

- Generated and managed private/public keys
- Created and verified digital signatures using both direct cryptography APIs and `WalletClient`
- Applied signatures in a Bitcoin transaction context using `WalletClient`
- Learned best practices for secure key management

These cryptographic concepts form the foundation of Bitcoin and blockchain technology. By understanding how keys and signatures work, you're well-equipped to build secure and robust applications using the BSV TypeScript SDK.

For more advanced techniques like different signature hash types (SIGHASH flags), manual signature creation, and multi-signature transactions, refer to the following documents:

- [Advanced Transaction Signing](../guides/advanced-transaction-signing.md) (How-To Guide)
- [Transaction Signatures Reference](../reference/transaction-signatures.md) (Technical Reference)

## Next Steps

- Learn about [Transaction Broadcasting and ARC](./transaction-broadcasting.md)
- Explore [Advanced Transaction Construction](./advanced-transaction.md)
- Dive deeper into [Script Construction and Custom Logic](./script-construction.md)
