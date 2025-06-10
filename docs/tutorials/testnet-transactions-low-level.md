# Working with Testnet Transactions

**Duration**: 30 minutes  
**Prerequisites**: Completed "Your First BSV Transaction" tutorial, Node.js, basic TypeScript knowledge  

## Learning Goals
- Set up a BSV testnet environment
- Obtain and manage testnet coins
- Create and broadcast real transactions on the testnet
- Track and verify transaction confirmations
- Configure WalletClient for different environments

## Introduction

While the previous tutorial taught you the basics of creating transaction structures, this tutorial takes you to the next level: creating and broadcasting actual transactions on the Bitcoin SV testnet. The testnet is a separate blockchain that functions almost identically to the main BSV network (mainnet), but with coins that have no real-world value, making it perfect for experimentation and learning.

## Step 1: Understanding Testnet vs Mainnet

Before we start, it's important to understand the difference between testnet and mainnet:

| Feature | Testnet | Mainnet |
|---------|---------|--------|
| Purpose | Testing and development | Real-world transactions |
| Coin value | None (free from faucets) | Real monetary value |
| Network prefix | Different from mainnet | Production network |
| Block difficulty | Lower than mainnet | Higher mining difficulty |
| Explorer URLs | Testnet-specific | Mainnet-specific |

The code you write for testnet can usually work on mainnet with minimal changes (primarily network configuration), making testnet ideal for development and testing.

## Step 2: Setting Up a Testnet Wallet

First, let's create a new project for our testnet experiments:

```bash
# Create a new directory for your testnet project
mkdir bsv-testnet-transactions
cd bsv-testnet-transactions

# Initialize a new Node.js project
npm init -y

# Install TypeScript and ts-node
npm install typescript ts-node @types/node --save-dev

# Install the BSV SDK
npm install @bsv/sdk
```

Now, create a basic TypeScript configuration file (`tsconfig.json`):

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

Let's create a file to generate a testnet wallet. Create a new file called `create-wallet.ts`:

```typescript
import { 
  PrivateKey,
  P2PKH
} from '@bsv/sdk'

// Create a new private key
const privateKey = PrivateKey.fromRandom()

// Derive the public key and create a testnet address
// Use the testnet prefix (0x6f) for the address
const address = privateKey.toAddress([0x6f])

// Display the information
console.log('===== TESTNET WALLET =====')
console.log(`Address: ${address.toString()}`)
console.log(`Private Key (WIF): ${privateKey.toWif()}`)
console.log('\nIMPORTANT: Save this information securely. You will need the private key to spend any received coins.')
console.log('=============================')
```

Run this script to generate your testnet wallet:

```bash
npx ts-node create-wallet.ts
```

You should see output similar to this:

```
===== TESTNET WALLET =====
Address: mzJR1zKcZCZvMJj87PUeQPPXRi3DEpyEDp
Private Key (WIF): cQcTnoUm6hQCSoGDQQiGfRxLEbsB6Sm3DnQPDrJdHZrP6ufuuJyp

IMPORTANT: Save this information securely. You will need the private key to spend any received coins.
=============================
```

**Important**: Save the displayed address and private key somewhere secure. You'll need them for the rest of this tutorial.

## Step 3: Getting Testnet Coins

Now that you have a testnet address, you need to get some testnet coins. Unlike mainnet, testnet coins are free and can be obtained from "faucets" - services that distribute testnet coins for development purposes.

Here are some BSV testnet faucets you can use:

- [Scrypt.io Testnet Faucet](https://scrypt.io/faucet)
- [BSV Testnet Faucet](https://bsvfaucet.org)

Visit one of these faucets, enter your testnet address, and request some coins. The faucet will send a small amount of testnet BSV to your address.

## Step 4: Verifying Your Balance and Finding UTXO Information

After requesting coins from a faucet, you'll need to verify that you received them and gather information about the UTXO (Unspent Transaction Output) you're going to spend. You can use a testnet block explorer for this purpose:

- [WhatsOnChain Testnet](https://test.whatsonchain.com/)
- [BitcoinSV Testnet Explorer](https://testnet.bitcoincloud.net/)

### Using WhatsOnChain Testnet Explorer

1. Go to [WhatsOnChain Testnet Explorer](https://test.whatsonchain.com/)

2. Enter your testnet address in the search bar at the top and click the search icon

3. You'll see your address details page with any transactions listed. Verify that you have received coins from the faucet. It may take a few minutes for the transaction to be confirmed.

4. Once you confirm that you've received coins with at least one confirmed transaction (unspent output/UTXO), you can proceed to gather the information needed for your transaction.

5. Look for the incoming transaction from the faucet

6. Click on the transaction ID (txid) to view the full transaction details

7. On the transaction details page, find the following information:
   - **Transaction ID (txid)**: This is the long hexadecimal string at the top of the page (e.g., `7f4e6ea49a847f557fccd9bf99d4a07ac103e5e8cb3464abb852af552516317e`)
   - **Output Index**: In the "Outputs" section, find your address and note its index number (0-based). If your address is the first output, the index is 0.
   - **Output Amount**: Note the amount sent to your address in this specific output. WhatsOnChain displays amounts in BSV (e.g., 0.00010000 BSV), but our code needs satoshis. To convert:
     * 1 BSV = 100,000,000 satoshis
     * Example: 0.00010000 BSV = 10,000 satoshis (multiply by 100,000,000)
     * You can use a calculator or simply move the decimal point 8 places to the right

8. Write down or copy these three pieces of information:
   ```
   Transaction ID (txid): [your transaction id]
   Output Index: [your output index, usually 0]
   Output Amount: [amount in BSV shown on WhatsOnChain] = [converted amount in satoshis]
                 Example: 0.00010000 BSV = 10000 satoshis
   ```

> **Important**: Make sure you're looking at an *unspent* output. If the coins have already been spent, you won't be able to use them in your transaction. WhatsOnChain typically shows if an output has been spent.

Write down this information as you'll need it for the next step.

## Step 5: Creating a Transaction with UTXOs

### Understanding UTXO Consumption

When you spend a UTXO in Bitcoin, the **entire UTXO** must be consumed. You can think of a UTXO like a bill in your wallet - if you want to spend a $20 bill, you must use the entire bill, not just part of it. If you only want to spend $5, you'll spend the entire $20 bill and get $15 back as change.

In the same way, when creating a Bitcoin transaction:

1. **You spend the entire UTXO** (even if you only want to send a portion of it)
2. **You specify how to distribute those funds**:
   - Some goes to the recipient (the payment)
   - Some goes to the miners (the transaction fee)
   - The remainder comes back to you (the change)

### Prerequisites

#### Taal API Key for Broadcasting

To broadcast transactions via the ARC API, you'll need a Taal API key:

1. Sign up for an account at [https://console.taal.com](https://console.taal.com)
2. Once logged in, navigate to the API keys section
3. Create a new API key for your application
4. Copy the generated API key - you'll need it for the transaction broadcasting step

### Implementation

Create a file `send-transaction.ts`:

```typescript
import { Transaction, PrivateKey, P2PKH, ARC, NodejsHttpClient } from '@bsv/sdk'
import https from 'https'

async function main() {
  try {
    // 1. Set up your wallet
    const privateKey = PrivateKey.fromWif('your_testnet_private_key_here')
    const myAddress = privateKey.toAddress([0x6f]) // 0x6f is the testnet prefix
    const recipientAddress = 'testnet_address_to_send_coins_to'

    // 2. Fetch the full transaction hex
    const txid = 'source_transaction_id_here'
    const response = await fetch(`https://api.whatsonchain.com/v1/bsv/test/tx/${txid}/hex`)
    const sourceTxHex = await response.text()
    console.log(`Retrieved transaction hex (first 50 chars): ${sourceTxHex.substring(0, 50)}...`)
    
    // 3. Create a transaction
    const tx = new Transaction()
    
    // 4. Add the input
    // For testnet, we need the hex of the transaction that contains our UTXO
    tx.addInput({
      sourceTransaction: Transaction.fromHex(sourceTxHex),
      sourceOutputIndex: 0, // The output index from Step 4 (typically 0)
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    })
    
    // 5. Add the recipient output
    tx.addOutput({
      lockingScript: new P2PKH().lock(recipientAddress),
      satoshis: 100 // Amount to send (must be less than input amount)
    })
    
    // 6. Add the change output back to our address
      tx.addOutput({
        lockingScript: new P2PKH().lock(myAddress),
      change: true // SDK will automatically calculate the change amount
      })
    
    // 7. Calculate fee and sign the transaction
    await tx.fee()
    await tx.sign()
    
    // 8. Broadcast the transaction to the testnet using ARC
    // You need to provide your Taal API key here
    // Get it by signing up at https://console.taal.com
    const apiKey = 'your_taal_api_key_here' // Replace with your actual API key
    
    // Create an HTTP client to ensure the transaction can be broadcast
    const httpClient = new NodejsHttpClient(https)
    
    // The SDK automatically appends '/v1/tx' to the base URL when broadcasting
    // so we need to use the base URL without the '/arc' suffix
    const arc = new ARC('https://arc-test.taal.com', {
      apiKey,
      httpClient, // Provide an HTTP client to avoid connectivity issues
      deploymentId: 'testnet-tutorial-deployment' // Provide a fixed deployment ID to avoid random generation
    })
    const result = await tx.broadcast(arc)
    console.log('ARC Response:', JSON.stringify(result, null, 2)) // Log the full response for debugging
    
    // 9. Display the transaction ID
    console.log(`Transaction ID: ${Buffer.from(tx.id()).toString('hex')}`)
    console.log(`View on explorer: https://test.whatsonchain.com/tx/${Buffer.from(tx.id()).toString('hex')}`)
    console.log('Transaction broadcast successfully!')
  } catch (error) {
    console.error('Error:', error)
  }
}

// Run the main function
main()
```

Before running this script, make sure to replace these values in the code:

1. `your_testnet_private_key_here`: Your testnet private key (WIF format) from Step 2
2. `testnet_address_to_send_coins_to`: The recipient's testnet address - for the purpose of this tutorial, use your own address
3. `source_transaction_id_here`: The ID of the transaction containing your UTXO
4. Update the `sourceOutputIndex` value if your output index is not 0
5. Adjust the recipient output `satoshis` value (currently 100) to be less than your input amount

Once you've made these changes, run the script:

```bash
npx ts-node send-transaction.ts
```

## Step 6: Tracking Your Transaction

After broadcasting your transaction, you can track its progress using a testnet explorer (note - it can take a few seconds for the transaction to show up). Our code already outputs the transaction ID and a direct link to view it on WhatsOnChain:

```typescript
console.log(`Transaction ID: ${Buffer.from(tx.id()).toString('hex')}`)
console.log(`View on explorer: https://test.whatsonchain.com/tx/${Buffer.from(tx.id()).toString('hex')}`)
```

Simply click the link or copy the transaction ID to search for it on [WhatsOnChain Testnet](https://test.whatsonchain.com/) or another testnet explorer.

Initially, your transaction will be unconfirmed. Once it's included in a block, it will show as confirmed. On testnet, confirmations typically happen faster than on mainnet due to lower mining difficulty.

## Step 7: Understanding Transaction Fees

Transaction fees are paid to miners to include your transaction in a block. In our simplified code example, we're using the BSV SDK's automatic fee calculation with:

```typescript
await tx.fee()
```

This method automatically calculates the appropriate fee based on:
- The size of your transaction (in bytes)
- Current network fee rates
- Number of inputs and outputs

The `change: true` parameter in our change output works with the fee calculation to:
1. Calculate the appropriate fee amount
2. Subtract that fee from the total input amount
3. Allocate the remaining balance to the change address

This approach ensures your transaction pays an appropriate fee based on its size and current network conditions, without needing to manually calculate fees and change amounts.

## Conclusion

Congratulations! You've successfully created, signed, and broadcast a real transaction on the Bitcoin SV testnet. In this tutorial, you've learned:

- How to set up a testnet wallet
- How to obtain testnet coins from a faucet
- How to find and use UTXO information
- How to create, sign, and broadcast a real transaction
- How to track transaction confirmations
- How to work with transaction fees

This practical experience with testnet transactions provides a solid foundation for working with real BSV transactions on the mainnet. With these skills, you're ready to move on to more advanced topics in the BSV TypeScript SDK.

## Next Steps

- Learn about [Key Management and Cryptography](./key-management.md)
- Explore [Advanced Transaction Construction](./advanced-transaction.md)
- Discover how to implement [Multi-signature Transactions](../guides/multisig-transactions.md)

## Additional Resources

- [BSV Testnet Faucet List](https://wiki.bitcoinsv.io/index.php/Testnet)
- [WhatsOnChain Testnet API Documentation](https://developers.whatsonchain.com/#introduction)
- [BSV Transaction Reference](../reference/transaction.md)
