# Transaction Broadcasting

**Duration**: 25 minutes  
**Prerequisites**: Completed "Your First BSV Transaction" tutorial, Node.js, basic TypeScript knowledge  

## Learning Goals
- Understand how transaction broadcasting works in BSV
- Learn the difference between WalletClient and direct broadcasting approaches
- Configure broadcasting for testnet vs mainnet
- Implement custom broadcasters for different services (ARC, WhatsOnChain)
- Handle broadcasting errors and responses

## Introduction

Transaction broadcasting is the process of submitting your signed transaction to the Bitcoin SV network so it can be included in a block. The BSV TypeScript SDK provides multiple approaches for broadcasting transactions, each suited for different use cases and deployment scenarios.

In this tutorial, you'll learn about the two main broadcasting approaches:

1. **WalletClient Broadcasting**: Uses the configured backend Wallet as a proxy (for example, the MetaNet Desktop Wallet)
2. **Direct Broadcasting**: Connects directly to mining services and APIs

## Understanding Broadcasting Architecture

### WalletClient Broadcasting Flow

When you use `WalletClient`, the broadcasting flow looks like this:

```
Your App → WalletClient → Wallet → Mining Services → BSV Network
```

The Wallet acts as a proxy that:
- Manages your broadcasting preferences
- Handles fallback logic between different services
- Provides a consistent API regardless of the underlying service

Due to its simplicity, this is the recommended approach. 

### Direct Broadcasting Flow

With direct broadcasting, your application connects directly to mining services:

```
Your App → Custom Broadcaster → Mining Service API → BSV Network
```

This approach gives you:
- Full control over which service to use
- Direct error handling and response processing
- Ability to implement custom retry logic
- No dependency on external wallet software

Due to its complexity and need to handle the low-level details of the broadcasting process, this is the less recommended approach.

## Step 1: WalletClient Broadcasting

Let's start with the WalletClient approach, which is the simplest for most applications.

### Basic WalletClient Setup

First, create a new project for our broadcasting examples:

```bash
# Create a new directory
mkdir bsv-broadcasting-tutorial
cd bsv-broadcasting-tutorial

# Initialize project
npm init -y

# Install dependencies
npm install typescript ts-node @types/node --save-dev
npm install @bsv/sdk
```

Create a basic TypeScript configuration (`tsconfig.json`):

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

### WalletClient Broadcasting Example

Create `wallet-broadcasting.ts`:

```typescript
import { WalletClient } from '@bsv/sdk'

async function walletClientBroadcasting() {
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
    
    // For this tutorial, we'll create a transaction with a simple OP_RETURN data output
    // The wallet will handle input selection, change outputs and fees
    console.log('\n🚀 Creating transaction...')
    const actionResult = await wallet.createAction({
      description: 'Broadcasting tutorial transaction',
      outputs: [
        {
          satoshis: 100, // Amount in satoshis (very small amount)
          // For this basic example, we'll use a standard OP_RETURN script
          // Here we use a pre-defined script for simplicity (OP_RETURN with simple data)
          lockingScript: '006a0461626364', // OP_RETURN with data 'abcd'
          outputDescription: 'Broadcasting tutorial data'
        }
      ]
    })
    
    console.log('Transaction created:')
    
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
      const txReference = actionResult.signableTransaction.reference
      
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
    
  } catch (error) {
    console.error('❌ Broadcasting failed:', error)
    console.log('Note: Make sure you have a compatible wallet running and are authenticated.')
  }
}

// Run the example
walletClientBroadcasting()
```

### How WalletClient Broadcasting Works

When you use `WalletClient`:

1. **Connection**: Your app connects to the MetaNet Desktop Wallet's local HTTP API (usually `http://localhost:3321`)
2. **Transaction Creation**: The wallet helps construct the transaction using your available UTXOs
3. **Signing**: The wallet signs the transaction with your private keys
4. **Broadcasting**: The wallet submits the transaction to whatever broadcast service is configured in its settings
5. **Response**: You receive either a transaction ID (success) or an error message

The key advantage is that **you don't control the broadcasting directly** - the MetaNet Desktop Wallet handles it based on its configuration. This means:

- ✅ Easy to use - no need to manage API keys or endpoints
- ✅ Fallback logic built-in
- ✅ User can configure preferred services through the wallet UI
- ❌ Less control over broadcasting behavior
- ❌ Requires MetaNet Desktop Wallet to be running

## Step 2: Direct Broadcasting with Custom Broadcasters

For more control, you can broadcast transactions directly using custom broadcaster implementations.

### Understanding Broadcaster Interface

All broadcasters in the SDK implement the `Broadcaster` interface:

```typescript
interface Broadcaster {
  broadcast(tx: Transaction): Promise<BroadcastResponse | BroadcastFailure>
}
```

The response types are:

```typescript
interface BroadcastResponse {
  status: 'success'
  txid: string
  message?: string
}

interface BroadcastFailure {
  status: 'error'
  code: string
  description: string
}
```

### ARC Broadcasting

ARC (Application Resource Component) is TAAL's enterprise-grade transaction processing service. Create `arc-broadcasting.ts`:

```typescript
import { Transaction, PrivateKey, P2PKH, ARC, NodejsHttpClient, Script } from '@bsv/sdk'
import https from 'https'

async function arcBroadcasting() {
  try {
    // 1. Set up your wallet (using testnet for this example)
    const privateKey = PrivateKey.fromRandom()
    const myAddress = privateKey.toAddress([0x6f]) // 0x6f is testnet prefix
    console.log(`Generated address: ${myAddress}`)
    console.log('Note: You would need to fund this address with testnet coins first')
    
    // 2. For this example, we'll create a simple transaction with an OP_RETURN output
    // In practice, you would add inputs from your funded UTXOs
    const tx = new Transaction()
    
    // Add a simple OP_RETURN output (doesn't require inputs for demonstration)
    tx.addOutput({
      lockingScript: Script.fromHex('006a0c42726f6164636173742054585821'), // OP_RETURN with "Broadcast TX!"
      satoshis: 0 // OP_RETURN outputs typically have 0 satoshis
    })
    
    // 3. Set up ARC broadcaster for testnet
    const apiKey = process.env.TAAL_API_KEY || 'your_taal_api_key_here'
    const httpClient = new NodejsHttpClient(https)
    
    const arc = new ARC('https://arc-test.taal.com', {
      apiKey,
      httpClient,
      deploymentId: 'broadcasting-tutorial'
    })
    
    // 4. Broadcast the transaction
    console.log('Broadcasting transaction with ARC...')
    const result = await tx.broadcast(arc)
    
    if (result.status === 'success') {
      console.log('✅ Transaction broadcast successful!')
      console.log(`Transaction ID: ${result.txid}`)
      console.log(`View on explorer: https://test.whatsonchain.com/tx/${result.txid}`)
    } else {
      console.log('❌ Broadcasting failed:', result)
    }
    
  } catch (error) {
    console.error('❌ Error during ARC broadcasting:', error)
  }
}

// Run the example
arcBroadcasting()
```

### WhatsOnChain Broadcasting

WhatsOnChain provides a free broadcasting service. Create `whatsonchain-broadcasting.ts`:

```typescript
import { Transaction, PrivateKey, P2PKH, WhatsOnChainBroadcaster, NodejsHttpClient, Script } from '@bsv/sdk'
import https from 'https'

async function whatsOnChainBroadcasting() {
  try {
    // 1. Create a simple transaction with OP_RETURN data
    const tx = new Transaction()
    
    // Add OP_RETURN output for demonstration
    tx.addOutput({
      lockingScript: Script.fromHex('006a0c57686174734f6e436861696e21'), // OP_RETURN with "WhatsOnChain!"
      satoshis: 0
    })
    
    // 2. Set up WhatsOnChain broadcaster for testnet
    const httpClient = new NodejsHttpClient(https)
    const broadcaster = new WhatsOnChainBroadcaster('test', httpClient)
    
    // 3. Broadcast the transaction
    console.log('Broadcasting transaction with WhatsOnChain...')
    const result = await tx.broadcast(broadcaster)
    
    if (result.status === 'success') {
      console.log('✅ Transaction broadcast successful!')
      console.log(`Transaction ID: ${result.txid}`)
      console.log(`View on explorer: https://test.whatsonchain.com/tx/${result.txid}`)
    } else {
      console.log('❌ Broadcasting failed:', result)
    }
    
  } catch (error) {
    console.error('❌ Error during WhatsOnChain broadcasting:', error)
  }
}

// Run the example
whatsOnChainBroadcasting()
```

## Step 3: Building a Custom Broadcaster

You can also create your own broadcaster for services not included in the SDK. Here's an example for a generic HTTP-based broadcaster:

Create `custom-broadcaster.ts`:

```typescript
import { Transaction, Broadcaster, BroadcastResponse, BroadcastFailure } from '@bsv/sdk'

class CustomHTTPBroadcaster implements Broadcaster {
  private url: string
  private headers: Record<string, string>
  
  constructor(url: string, headers: Record<string, string> = {}) {
    this.url = url
    this.headers = {
      'Content-Type': 'application/json',
      ...headers
    }
  }
  
  async broadcast(tx: Transaction): Promise<BroadcastResponse | BroadcastFailure> {
    try {
      const txHex = tx.toHex()
      
      const response = await fetch(this.url, {
        method: 'POST',
        headers: this.headers,
        body: JSON.stringify({ 
          txhex: txHex 
        })
      })
      
      const data = await response.json()
      
      if (response.ok && data.txid) {
        return {
          status: 'success',
          txid: data.txid,
          message: data.message || 'Broadcast successful'
        }
      } else {
        return {
          status: 'error',
          code: response.status.toString(),
          description: data.error || data.message || 'Unknown error'
        }
      }
      
    } catch (error) {
      return {
        status: 'error',
        code: '500',
        description: error instanceof Error ? error.message : 'Network error'
      }
    }
  }
}

// Example usage
async function customBroadcasterExample() {
  const customBroadcaster = new CustomHTTPBroadcaster(
    'https://api.example.com/broadcast',
    {
      'Authorization': 'Bearer your-api-key',
      'X-Custom-Header': 'value'
    }
  )
  
  // Use with any transaction
  const tx = new Transaction() // Your transaction here
  const result = await tx.broadcast(customBroadcaster)
  
  console.log('Custom broadcast result:', result)
}
```

## Step 4: Network Configuration (Testnet vs Mainnet)

Different networks require different broadcaster configurations:

Create `network-config.ts`:

```typescript
import { ARC, WhatsOnChainBroadcaster, Broadcaster } from '@bsv/sdk'

interface NetworkConfig {
  name: string
  arc: Broadcaster
  whatsOnChain: Broadcaster
}

// Network configurations
const networks: Record<string, NetworkConfig> = {
  testnet: {
    name: 'BSV Testnet',
    arc: new ARC('https://arc-test.taal.com'),
    whatsOnChain: new WhatsOnChainBroadcaster('test')
  },
  
  mainnet: {
    name: 'BSV Mainnet',
    arc: new ARC('https://arc.taal.com', {
      apiKey: process.env.TAAL_API_KEY // Use environment variable for production
    }),
    whatsOnChain: new WhatsOnChainBroadcaster('main')
  }
}

async function networkAwareBroadcasting(networkName: 'testnet' | 'mainnet') {
  const config = networks[networkName]
  
  console.log(`Broadcasting on ${config.name}`)
  
  // Your transaction here
  const tx = new Transaction()
  
  // Try ARC first, fallback to WhatsOnChain
  try {
    console.log('Trying ARC...')
    const result = await tx.broadcast(config.arc)
    
    if (result.status === 'success') {
      console.log('✅ ARC broadcast successful:', result.txid)
      return result
    } else {
      console.log('ARC failed, trying WhatsOnChain...')
      throw new Error(result.description)
    }
    
  } catch (error) {
    console.log('ARC failed, trying WhatsOnChain...')
    
    try {
      const result = await tx.broadcast(config.whatsOnChain)
      
      if (result.status === 'success') {
        console.log('✅ WhatsOnChain broadcast successful:', result.txid)
        return result
      } else {
        console.log('❌ All broadcasters failed')
        throw new Error(result.description)
      }
      
    } catch (wocError) {
      console.error('❌ All broadcasting attempts failed')
      throw wocError
    }
  }
}

// Example usage
networkAwareBroadcasting('testnet')
  .then(result => console.log('Final result:', result))
  .catch(error => console.error('Broadcasting failed:', error))
```

## Step 5: Error Handling and Retry Logic

Robust applications need proper error handling for broadcasting:

Create `robust-broadcasting.ts`:

```typescript
import { Transaction, Broadcaster, BroadcastResponse, BroadcastFailure } from '@bsv/sdk'

class RobustBroadcaster {
  private broadcasters: Broadcaster[]
  private maxRetries: number
  private retryDelay: number
  
  constructor(
    broadcasters: Broadcaster[], 
    maxRetries: number = 3, 
    retryDelay: number = 1000
  ) {
    this.broadcasters = broadcasters
    this.maxRetries = maxRetries
    this.retryDelay = retryDelay
  }
  
  private async delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms))
  }
  
  async broadcast(tx: Transaction): Promise<BroadcastResponse> {
    let lastError: Error | null = null
    
    for (const broadcaster of this.broadcasters) {
      for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
        try {
          console.log(`Attempt ${attempt} with broadcaster ${broadcaster.constructor.name}`)
          
          const result = await broadcaster.broadcast(tx)
          
          if (result.status === 'success') {
            console.log(`✅ Success with ${broadcaster.constructor.name}`)
            return result
          } else {
            console.log(`❌ Failed with ${broadcaster.constructor.name}: ${result.description}`)
            lastError = new Error(`${result.code}: ${result.description}`)
            
            // Don't retry on certain errors
            if (this.isNonRetryableError(result.code)) {
              break
            }
          }
          
        } catch (error) {
          console.log(`❌ Exception with ${broadcaster.constructor.name}:`, error)
          lastError = error instanceof Error ? error : new Error(String(error))
        }
        
        // Wait before retry (except on last attempt)
        if (attempt < this.maxRetries) {
          await this.delay(this.retryDelay * attempt) // Exponential backoff
        }
      }
    }
    
    throw lastError || new Error('All broadcasting attempts failed')
  }
  
  private isNonRetryableError(code: string): boolean {
    // Don't retry on these error codes
    const nonRetryableCodes = [
      '400', // Bad request - transaction is invalid
      '409', // Conflict - transaction already exists
      '422'  // Unprocessable entity - transaction validation failed
    ]
    
    return nonRetryableCodes.includes(code)
  }
}

// Example usage
async function robustBroadcastingExample() {
  const robustBroadcaster = new RobustBroadcaster([
    networks.testnet.arc,
    networks.testnet.whatsOnChain
  ])
  
  try {
    const tx = new Transaction() // Your transaction
    const result = await robustBroadcaster.broadcast(tx)
    
    console.log('✅ Transaction broadcast successfully:', result.txid)
    
  } catch (error) {
    console.error('❌ Failed to broadcast transaction:', error)
  }
}
```

## Step 6: Monitoring and Verification

After broadcasting, you should verify that your transaction was accepted:

Create `transaction-monitoring.ts`:

```typescript
import { Transaction } from '@bsv/sdk'

async function verifyTransaction(txid: string, network: 'test' | 'main' = 'test') {
  const baseUrl = network === 'test' 
    ? 'https://api.whatsonchain.com/v1/bsv/test'
    : 'https://api.whatsonchain.com/v1/bsv/main'
  
  try {
    // Check if transaction exists
    const response = await fetch(`${baseUrl}/tx/${txid}`)
    
    if (response.ok) {
      const txData = await response.json()
      
      console.log('✅ Transaction found on network')
      console.log('Transaction ID:', txData.txid)
      console.log('Block height:', txData.blockheight || 'Unconfirmed')
      console.log('Confirmations:', txData.confirmations || 0)
      
      return txData
    } else if (response.status === 404) {
      console.log('⏳ Transaction not yet visible on network')
      return null
    } else {
      throw new Error(`API error: ${response.status}`)
    }
    
  } catch (error) {
    console.error('❌ Error verifying transaction:', error)
    throw error
  }
}

async function waitForTransaction(
  txid: string, 
  network: 'test' | 'main' = 'test',
  timeoutMs: number = 30000
): Promise<any> {
  const startTime = Date.now()
  
  while (Date.now() - startTime < timeoutMs) {
    const txData = await verifyTransaction(txid, network)
    
    if (txData) {
      return txData
    }
    
    // Wait 2 seconds before checking again
    await new Promise(resolve => setTimeout(resolve, 2000))
  }
  
  throw new Error(`Transaction ${txid} not found within ${timeoutMs}ms`)
}

// Example usage
async function monitoringExample() {
  // After broadcasting a transaction
  const txid = 'your-transaction-id-here'
  
  try {
    console.log('Waiting for transaction to appear on network...')
    const txData = await waitForTransaction(txid, 'test')
    
    console.log('✅ Transaction confirmed:', txData)
    
  } catch (error) {
    console.error('❌ Transaction monitoring failed:', error)
  }
}
```

## Summary

In this tutorial, you learned about the two main approaches to transaction broadcasting in BSV:

### WalletClient Approach
- ✅ **Simple**: Easy to use with MetaNet Desktop Wallet
- ✅ **Managed**: Wallet handles service selection and fallbacks
- ✅ **User Control**: Users can configure preferred services
- ❌ **Dependency**: Requires wallet software running
- ❌ **Limited Control**: Less control over broadcasting behavior

### Direct Broadcasting Approach
- ✅ **Full Control**: Choose exactly which service to use
- ✅ **No Dependencies**: Works without external wallet software
- ✅ **Custom Logic**: Implement your own retry and fallback logic
- ✅ **Error Handling**: Direct access to service responses
- ❌ **More Complex**: Requires more setup and configuration

### Key Takeaways

1. **For Development**: Use WalletClient with MetaNet Desktop Wallet for quick prototyping
2. **For Production**: Consider direct broadcasting for better control and reliability
3. **Network Configuration**: Always use appropriate endpoints for testnet vs mainnet
4. **Error Handling**: Implement robust retry logic and fallback mechanisms
5. **Monitoring**: Verify transactions were accepted by the network

### Next Steps

- Experiment with different broadcaster configurations
- Implement custom broadcasters for other services
- Build monitoring dashboards for your applications
- Explore advanced features like batch broadcasting

The broadcasting approach you choose depends on your application's requirements, deployment environment, and control needs. Both approaches are valid and can be used effectively in different scenarios.
