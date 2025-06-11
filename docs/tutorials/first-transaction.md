# Your First BSV Transaction

**Duration**: 15 minutes  
**Prerequisites**: Node.js, basic TypeScript knowledge  

## Learning Goals
- Install and configure the BSV TypeScript SDK
- Create a simple transaction using WalletClient
- Understand BSV transaction anatomy

> **📚 Related Concepts**: Before starting, you may want to review [Transaction Structure](../concepts/transaction-structure.md) and [Wallet Integration](../concepts/wallet-integration.md) to understand the fundamentals.

## Introduction

In this tutorial, you'll learn how to create your first Bitcoin SV transaction using the TypeScript SDK's WalletClient interface on the mainnet network. This approach makes it easy to build transactions by abstracting away many of the low-level details. By the end, you'll understand the basic components of a BSV transaction and how to construct, sign, and broadcast one on the BSV blockchain.

## Precondition 

Install a BRC-100 compliant wallet such as the [MetaNet Desktop Wallet](https://metanet.bsvb.tech/). When you install it, you'll receive a small amount of funds to play with.

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

For a deeper understanding of Bitcoin transactions, check out these resources:

- [Bitcoin SV Protocol Specifications](https://github.com/bitcoin-sv-specs/protocol) - Official Bitcoin SV protocol specifications
- [WhatsOnChain Explorer](https://whatsonchain.com/) - Standard block explorer for viewing BSV transactions
- [Bitcoin Developer Reference](https://developer.bitcoin.org/reference/transactions.html) - Comprehensive explanation of transaction structure

## Step 3: Creating Your First Transaction With WalletClient

Create a new file called `first-transaction.ts`:

```typescript
import { WalletClient } from '@bsv/sdk'

async function createTransaction() {
  try {
    // Initialize the WalletClient, using localhost as wallet substrate
    const wallet = new WalletClient('auto', 'localhost')
    
    // Check if we're authenticated with the wallet
    const { authenticated } = await wallet.isAuthenticated()
    if (!authenticated) {
      console.log('Please authenticate with your wallet')
      await wallet.waitForAuthentication()
      console.log('Successfully authenticated!')
    }
    
    // Get wallet version
    const { version } = await wallet.getVersion()
    console.log(`Wallet version: ${version}`)
    
    // Get our identity public key
    const { publicKey } = await wallet.getPublicKey({ identityKey: true })
    console.log(`Your identity public key: ${publicKey}`)
    
    // For this tutorial, we'll create a transaction with a simple OP_RETURN data output
    // The wallet will handle input selection, change outputs and fees
    const actionResult = await wallet.createAction({
      description: 'My first BSV transaction', 
      outputs: [
        {
          satoshis: 100, // Amount in satoshis (very small amount)
          // For this basic example, we'll use a standard P2PKH script template
          // In a real application, you would create a proper script from the public key
          // Here we use a pre-defined script for simplicity (OP_RETURN with simple data)
          lockingScript: '006a0461626364', // OP_RETURN with data 'abcd'
          outputDescription: 'Simple data output'
        }
      ],
      // You can add options here if needed
      // options: { ... }
    })
    
    console.log('Transaction created:')
    
    let txReference
    
    if (actionResult.txid) {
      // If the wallet auto-signed and broadcast the transaction
      console.log('Full action result:', JSON.stringify(actionResult, null, 2))

      console.log(`Transaction ID: ${actionResult.txid}`)
      console.log(`View on explorer: https://whatsonchain.com/tx/${actionResult.txid}`)
      console.log('Transaction was automatically signed and broadcast!')
    } 
    else if (actionResult.signableTransaction) {
      console.log('Created transaction that needs signing')
      // Get the reference needed for signing
      txReference = actionResult.signableTransaction.reference
      
      // Now sign the transaction
      // Note: In a real application, you might prompt the user to sign
      const signResult = await wallet.signAction({
        // The wallet knows which inputs need to be spent based on the reference
        spends: {},
        // Use the reference from the createAction result
        reference: txReference,
        options: {
          acceptDelayedBroadcast: true
        }
      })

      console.log(`Transaction signed and broadcast!`)
      console.log(`Transaction ID: ${signResult.txid}`)
      console.log(`View on explorer: https://whatsonchain.com/tx/${signResult.txid}`)
    }
    else {
      console.log('Transaction created but no actionable result returned')
    }
    
    console.log('Transaction successfully created and broadcast to the BSV mainnet!')
    console.log('**Note**: This transaction uses real BSV, though only a very small amount (100 satoshis + fees)')
    
  } catch (error) {
    console.error('Error:', error)
  }
}

// Call the function to create the transaction
createTransaction().catch(console.error)
```

## Step 4: Run Your Code

Execute your code with the following command:

```bash
npx ts-node first-transaction.ts
```

When you run this code:

1. If you don't have a wallet connected, you'll be prompted to authenticate with one
2. Make sure your wallet is properly connected and authenticated
3. The wallet will handle input selection, change, and fee calculation
4. Upon successful completion, you'll see your transaction ID in the console

### Viewing Your Transaction

Once your transaction is broadcast, you can view it on the BSV mainnet block explorer:

1. Copy the transaction ID from your console output
2. Visit https://whatsonchain.com/
3. Paste the transaction ID in the search box and click search
4. You'll see details of your transaction including:
   - Inputs (where the BSV came from)
   - Outputs (where the BSV was sent)
   - Fees paid
   - Block confirmation status

## Step 5: Understanding How WalletClient Works

The WalletClient interface simplifies transaction creation by:

1. **Abstracting away transaction details**: You don't need to manually construct inputs and outputs
2. **Managing UTXOs automatically**: The wallet selects appropriate UTXOs for spending
3. **Calculating fees**: No need to worry about transaction fee calculations
4. **Handling change**: Automatically creates change outputs when needed
5. **Taking care of signing**: Manages the cryptographic operations needed to sign transactions

### Key Methods for Transaction Management

- `createAction()`: Prepares a transaction with specified outputs
- `signAction()`: Signs and optionally broadcasts a transaction
- `listActions()`: Retrieves transaction history
- `abortAction()`: Cancels an unsigned transaction

## Step 6: Understanding Network Environments

### Mainnet and Testnet

In this tutorial, we used the Bitcoin SV mainnet. For future development, you might also want to use testnet:

- **Mainnet**: The main Bitcoin SV blockchain where coins have real monetary value
- **Testnet**: A separate blockchain used for testing where coins have no real value

### Working with Testnet

*** TODO ***

If you want to experiment without using real BSV, you can use testnet:

1. Ensure your wallet is connected to testnet (most wallets have a network selector)
2. No code changes are needed when using the WalletClient approach - the wallet handles the network selection
3. You'll need testnet coins, which you can obtain from faucets
4. Use https://test.whatsonchain.com/ to view your testnet transactions
5. This is ideal for development and testing before deploying to mainnet

> **Important:** Always test thoroughly on testnet before working with substantial real funds on mainnet.

### Next Steps

In the next tutorial, "[Transaction Types and Data](./transaction-types.md)", you'll learn how to:

1. Create more complex transaction types
2. Work with multiple outputs
3. Add data to transactions
4. Use advanced WalletClient features

## Conclusion

Congratulations! You've learned the basics of creating a BSV transaction using the TypeScript SDK's WalletClient. In this tutorial, you've:

- Set up your development environment
- Learned about transaction components
- Created a simple transaction using WalletClient
- Understood how WalletClient simplifies transaction management

## Next Steps

- Ready to try test transactions? Continue with [Working with Testnet Transactions](./testnet-transactions.md)
- Learn about [Key Management and Cryptography](./key-management.md)
- Prefer lower-level control? Check out [First Transaction (Low-level API)](./first-transaction-low-level.md)

## Additional Resources

- [WalletClient API Reference](../reference/wallet.md#class-walletclient)
- [BSV Wallet Protocols](https://projectbabbage.com/docs/guides/wallet/transactions)
- [Transaction Broadcasting](https://projectbabbage.com/docs/guides/wallet/signing)
