# Your First BSV Transaction (Low level)

**Duration**: 15 minutes  
**Prerequisites**: Node.js, basic TypeScript knowledge  

## Learning Goals
- Install and configure the BSV TypeScript SDK
- Create a simple P2PKH transaction
- Understand BSV transaction anatomy

## Introduction

In this tutorial, you'll learn how to create your first Bitcoin SV transaction using the TypeScript SDK. By the end, you'll understand the basic components of a BSV transaction and how to construct, sign, and broadcast one.

## Step 1: Setting Up Your Environment

First, create a new Node.js project and install the BSV SDK:

```bash
# Create a new directory for your project
mkdir my-first-bsv-tx
cd my-first-bsv-tx

# Initialize a new Node.js project
npm init -y

# Install TypeScript and ts-node (TypeScript execution engine)
npm install typescript ts-node @types/node --save-dev
# ts-node allows you to run TypeScript files directly without compiling them first

# Install the BSV SDK
npm install @bsv/sdk
```

Create a basic TypeScript configuration file (`tsconfig.json`):

```json
{
  "compilerOptions": {
    "target": "es2020",
    "module": "commonjs",
    "esModuleInterop": true,
    "strict": true,
    "outDir": "./dist"
  }
}
```

## Step 2: Understanding Transaction Components

Before we write any code, let's understand the basic components of a Bitcoin transaction:

- **Inputs**: References to previous transaction outputs that you're spending
- **Outputs**: New UTXOs (Unspent Transaction Outputs) that define who receives the bitcoins
- **Locking Scripts**: Scripts that determine the conditions for spending outputs
- **Unlocking Scripts**: Scripts that satisfy the conditions in locking scripts

### Additional Resources

For a deeper understanding of Bitcoin transactions, check out these official resources:

- [BSV Wiki: Bitcoin Transaction](https://wiki.bitcoinsv.io/index.php/Bitcoin_Transaction) - Comprehensive explanation of transaction structure
- [BSV Wiki: Script](https://wiki.bitcoinsv.io/index.php/Script) - Detailed information about Bitcoin Script language
- [Bitcoin Whitepaper](https://craigwright.net/bitcoin-white-paper.pdf) - Section 5 covers the original design of transactions
- [BSV Academy: Transactions Course](https://bitcoinsv.academy/course/transactions) - In-depth course on Bitcoin transactions

## Step 3: Creating Your First Transaction

Create a new file called `first-transaction.ts`:

```typescript
import { PrivateKey, P2PKH, Transaction } from '@bsv/sdk'

async function createTransaction() {
  // Create a private key (in production, you'd use proper key management)
  const privateKey = PrivateKey.fromRandom()
  console.log(`Private key WIF: ${privateKey.toWif()}`)
  
  // Derive the public key and address
  const address = privateKey.toAddress()
  console.log(`Address: ${address.toString()}`)
  
  // For a real transaction, you would have a source UTXO
  // For this example, we'll create a transaction without inputs (cannot be broadcast)
  const tx = new Transaction()
  
  // Add an output
  tx.addOutput({
    lockingScript: new P2PKH().lock(address),
    satoshis: 100
  })
  
  // Serialize the transaction
  const txHex = tx.toHex() // Use toHex() instead of toString()
  console.log(`Transaction (hex): ${txHex}`)
  
  // Get transaction ID as a hex string
  const txid = Buffer.from(tx.id()).toString('hex') // Convert the byte array to hex string
  console.log(`Transaction ID: ${txid}`)
  
  // Display the transaction structure
  console.log('\nTransaction Structure:')
  console.log(`Version: ${tx.version}`)
  console.log(`Input Count: ${tx.inputs.length}`)
  console.log(`Output Count: ${tx.outputs.length}`)
  
  for (let i = 0; i < tx.outputs.length; i++) {
    const output = tx.outputs[i]
    console.log(`\nOutput #${i}:`)
    console.log(`  Satoshis: ${output.satoshis}`)
    console.log(`  Locking Script: ${output.lockingScript.toHex()}`)
    console.log(`  Locking Script (ASM): ${output.lockingScript.toASM()}`)
  }
}

createTransaction().catch(console.error)
```

## Step 4: Run Your Code

Execute your code with the following command:

```bash
npx ts-node first-transaction.ts
```

You should see output showing your private key, address, and transaction details.

## Step 5: Next Steps for Real Transactions

The transaction we created in the previous step doesn't have any inputs, so it can't be broadcast to the network. It serves as a conceptual introduction to transaction structure.

In the next tutorial, "[Working with Testnet Transactions](./testnet-transactions.md)", you'll learn how to:

1. Create a wallet specifically for the BSV testnet
2. Obtain free testnet coins from a faucet
3. Create transactions with real inputs and outputs
4. Calculate transaction fees automatically
5. Sign and broadcast real transactions to the testnet network

### Preview: Transaction Structure for Real Transactions

Here's a simplified preview of what a complete transaction looks like. Don't worry about understanding all the details yet - this is just to give you a sense of the structure you'll work with in the next tutorial:

```typescript
// A complete transaction typically follows this structure:
const tx = new Transaction()

// Add an input (where the money comes from)
tx.addInput({
  sourceTransaction: /* previous transaction containing your funds */,
  sourceOutputIndex: /* which output from that transaction */,
  unlockingScriptTemplate: /* script to unlock those funds */
})

// Add an output (where the money goes)
tx.addOutput({
  lockingScript: /* script that locks funds to recipient */,
  satoshis: /* amount to send */
})

// Add change output (remaining funds returned to you)
tx.addOutput({
  lockingScript: /* script that locks funds to you */,
  change: true // Automatically handles change amount
})

// Calculate fee and sign
await tx.fee()
await tx.sign()

// Broadcast to the network
const result = await tx.broadcast()
```

### Understanding Transaction Fees

Bitcoin transactions require fees to be included in a block by miners. The BSV SDK simplifies fee calculation with the `fee()` method, which:

- Calculates the appropriate fee based on transaction size
- Works with the `change: true` parameter to automatically handle change outputs

You'll get hands-on experience with transaction fees in the testnet tutorial.

## Conclusion

Congratulations! You've learned the basics of creating a BSV transaction using the TypeScript SDK. In this tutorial, you've:

- Set up your development environment
- Learned about transaction components
- Created a simple transaction
- Understood how to structure a complete transaction

## Next Steps

- Ready to try real transactions? Continue with [Working with Testnet Transactions](./testnet-transactions-low-level.md)
- Learn about [Key Management and Cryptography](./key-management.md)
- Explore how to [Broadcast Transactions with ARC](./transaction-broadcasting.md)

## Additional Resources

- [BSV Transaction Reference](../reference/transaction.md)
- [P2PKH Script Reference](../reference/script.md#p2pkh)
- [Bitcoin Transaction Explorer](https://whatsonchain.com) - To view your transactions on the blockchain
