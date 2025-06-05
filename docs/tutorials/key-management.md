# Key Management and Cryptography

**Duration**: 45 minutes  
**Prerequisites**: Completed "Your First BSV Transaction" tutorial, Node.js, basic TypeScript knowledge  

## Learning Goals
- Generate and manage private/public keys
- Understand ECDSA signatures
- Create and verify digital signatures
- Apply secure key management practices

## Introduction

Bitcoin is built on cryptographic principles, with keys and signatures forming the foundation of its security model. In this tutorial, you'll learn how to generate, manage, and use cryptographic keys with the BSV TypeScript SDK. You'll also learn how to create and verify digital signatures, which are essential for authorizing transactions and proving ownership.

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

// Derive a testnet address from a private key
function deriveTestnetAddress(privateKey: PrivateKey) {
  // Use testnet prefix (0x6f) instead of mainnet (0x00)
  const testnetAddress = privateKey.toAddress([0x6f])
  
  console.log('\n=== Testnet Address ===')  
  console.log(`Testnet Address: ${testnetAddress.toString()}`)
  return testnetAddress
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
  
  // Create testnet address from the key
  deriveTestnetAddress(newKey)
  
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

Let's create a new file called `signatures.ts` to explore how signatures work:

```typescript
import { PrivateKey, PublicKey, Signature } from '@bsv/sdk'

async function signatureExamples() {
  // Generate a key to use for signing
  const privateKey = PrivateKey.fromRandom()
  const publicKey = privateKey.toPublicKey()
  
  console.log('\n=== Key for Signing ===')  
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
signatureExamples().catch(console.error)
```

Run the script:

```bash
npx ts-node signatures.ts
```

You should see the results of signing and verifying messages, including what happens when messages are tampered with or wrong keys are used.

## Step 5: Practical Application: Signing and Verifying a Transaction

Let's put our knowledge to practical use by creating a simple transaction and manually signing it. This will help you understand how signatures are used in Bitcoin transactions.

Create a file called `transaction-signing.ts`:

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

Run the script:

```bash
npx ts-node transaction-signing.ts
```

This demonstrates how digital signatures are used in actual Bitcoin transactions.

## Step 6: Best Practices for Key Management

Now that you understand how to work with keys and signatures, let's discuss best practices for managing keys securely.

### Security Considerations

1. **Never share private keys**: Private keys should never be shared, posted online, or stored in plain text

2. **Use hardware wallets for significant funds**: Hardware wallets provide strong security by keeping private keys offline

3. **Implement proper backup procedures**: Create secure backups of private keys or seed phrases

4. **Use encryption**: Encrypt private keys when they must be stored

5. **Consider multi-signature setups**: For high-security applications, consider requiring multiple signatures

## Conclusion

Congratulations! You've learned the fundamentals of key management and cryptography with the BSV TypeScript SDK. In this tutorial, you've:

- Generated and managed private/public keys
- Created and verified digital signatures
- Understood how signatures are used in Bitcoin transactions
- Explored best practices for secure key management

These cryptographic principles form the foundation of Bitcoin's security model and will be essential in your journey to build robust Bitcoin applications.

## See Also

For more advanced information about transaction signatures, including:
- Different signature hash types (SIGHASH flags)
- Manual signature creation and verification
- Advanced transaction signing techniques
- Low-level cryptographic operations

See the following resources:
- [Advanced Transaction Signing](../guides/advanced-transaction-signing.md) (How-To Guide)
- [Transaction Signatures Reference](../reference/transaction-signatures.md) (Technical Reference)

## Next Steps

- Learn about [Transaction Broadcasting and ARC](./transaction-broadcasting.md)
- Explore [Advanced Transaction Construction](./advanced-transaction.md)
- Dive deeper into [Script Construction and Custom Logic](./script-construction.md)

