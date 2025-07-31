# Custom Transaction Broadcasters

This guide shows how to create custom broadcaster implementations for services not included in the BSV TypeScript SDK. This is an advanced topic for developers who need to integrate with specific broadcasting services or implement custom broadcasting logic.

## Prerequisites

- Understanding of the [Transaction Broadcasting tutorial](../tutorials/transaction-broadcasting.md)
- Familiarity with HTTP APIs and error handling
- Knowledge of TypeScript interfaces and classes

## Understanding the Broadcaster Interface

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

## Basic HTTP Broadcaster Implementation

Here's a generic HTTP-based broadcaster that can be adapted for various services:

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
```

## Advanced Broadcaster with Retry Logic

For production use, you'll want more robust error handling and retry logic:

```typescript
class RobustHTTPBroadcaster implements Broadcaster {
  private url: string
  private headers: Record<string, string>
  private maxRetries: number
  private retryDelay: number
  
  constructor(
    url: string, 
    headers: Record<string, string> = {},
    maxRetries: number = 3,
    retryDelay: number = 1000
  ) {
    this.url = url
    this.headers = {
      'Content-Type': 'application/json',
      ...headers
    }
    this.maxRetries = maxRetries
    this.retryDelay = retryDelay
  }
  
  async broadcast(tx: Transaction): Promise<BroadcastResponse | BroadcastFailure> {
    let lastError: BroadcastFailure | null = null
    
    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        const result = await this.attemptBroadcast(tx)
        
        if (result.status === 'success') {
          return result
        }
        
        // Don't retry on certain error codes
        if (this.isNonRetryableError(result.code)) {
          return result
        }
        
        lastError = result
        
        // Wait before retry (except on last attempt)
        if (attempt < this.maxRetries) {
          await this.delay(this.retryDelay * Math.pow(2, attempt)) // Exponential backoff
        }
        
      } catch (error) {
        lastError = {
          status: 'error',
          code: '500',
          description: error instanceof Error ? error.message : 'Network error'
        }
        
        if (attempt < this.maxRetries) {
          await this.delay(this.retryDelay * Math.pow(2, attempt))
        }
      }
    }
    
    return lastError || {
      status: 'error',
      code: '500',
      description: 'Max retries exceeded'
    }
  }
  
  private async attemptBroadcast(tx: Transaction): Promise<BroadcastResponse | BroadcastFailure> {
    const txHex = tx.toHex()
    
    const response = await fetch(this.url, {
      method: 'POST',
      headers: this.headers,
      body: JSON.stringify({ txhex: txHex })
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
  }
  
  private isNonRetryableError(code: string): boolean {
    // Don't retry on client errors (4xx) except for rate limiting
    const numCode = parseInt(code)
    return numCode >= 400 && numCode < 500 && numCode !== 429
  }
  
  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms))
  }
}
```

## Multi-Service Broadcaster with Failover

For maximum reliability, you can create a broadcaster that tries multiple services:

```typescript
class FailoverBroadcaster implements Broadcaster {
  private broadcasters: Broadcaster[]
  
  constructor(broadcasters: Broadcaster[]) {
    if (broadcasters.length === 0) {
      throw new Error('At least one broadcaster is required')
    }
    this.broadcasters = broadcasters
  }
  
  async broadcast(tx: Transaction): Promise<BroadcastResponse | BroadcastFailure> {
    const errors: BroadcastFailure[] = []
    
    for (const broadcaster of this.broadcasters) {
      try {
        const result = await broadcaster.broadcast(tx)
        
        if (result.status === 'success') {
          return result
        }
        
        errors.push(result)
        
      } catch (error) {
        errors.push({
          status: 'error',
          code: '500',
          description: error instanceof Error ? error.message : 'Unknown error'
        })
      }
    }
    
    // All broadcasters failed
    return {
      status: 'error',
      code: '503',
      description: `All broadcasters failed. Errors: ${errors.map(e => e.description).join('; ')}`
    }
  }
}
```

## Usage Examples

### Basic Custom Broadcaster

```typescript
import { WalletClient, Transaction } from '@bsv/sdk'

async function useCustomBroadcaster() {
  // Create custom broadcaster
  const customBroadcaster = new CustomHTTPBroadcaster(
    'https://api.example.com/broadcast',
    {
      'Authorization': 'Bearer your-api-key',
      'X-Custom-Header': 'value'
    }
  )
  
  // Create transaction (using wallet for simplicity)
  const wallet = new WalletClient('auto', 'localhost')
  const actionResult = await wallet.createAction({
    description: 'Custom broadcaster test',
    outputs: [{
      satoshis: 100,
      lockingScript: '76a914f1c075a01882ae0972f95d3a4177c86c852b7d9188ac',
      outputDescription: 'Test payment'
    }],
    options: { noSend: true }
  })
  
  if (!actionResult.tx) {
    throw new Error('Transaction creation failed')
  }
  
  const tx = Transaction.fromAtomicBEEF(actionResult.tx)
  
  // Broadcast with custom broadcaster
  const result = await tx.broadcast(customBroadcaster)
  
  if (result.status === 'success') {
    console.log('✅ Custom broadcast successful:', result.txid)
  } else {
    console.log('❌ Custom broadcast failed:', result.description)
  }
}
```

### Robust Broadcaster with Failover

```typescript
import { ARC, WhatsOnChainBroadcaster, NodejsHttpClient } from '@bsv/sdk'
import https from 'https'

async function useFailoverBroadcaster() {
  const httpClient = new NodejsHttpClient(https)
  
  // Create multiple broadcasters
  const arc = new ARC('https://arc.taal.com', {
    apiKey: process.env.TAAL_API_KEY || 'your-api-key',
    httpClient
  })
  
  const whatsOnChain = new WhatsOnChainBroadcaster('main', httpClient)
  
  const customBroadcaster = new RobustHTTPBroadcaster(
    'https://api.example.com/broadcast',
    { 'Authorization': 'Bearer your-api-key' },
    3, // max retries
    1000 // initial delay
  )
  
  // Create failover broadcaster
  const failoverBroadcaster = new FailoverBroadcaster([
    arc,                // Try ARC first
    whatsOnChain,       // Fallback to WhatsOnChain
    customBroadcaster   // Final fallback to custom service
  ])
  
  // Use with any transaction
  const tx = new Transaction() // Your transaction here
  const result = await tx.broadcast(failoverBroadcaster)
  
  console.log('Failover broadcast result:', result)
}
```

## Testing Custom Broadcasters

Always test your custom broadcasters thoroughly:

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

async function testCustomBroadcaster() {
  // Create a simple test transaction
  const privateKey = PrivateKey.fromRandom()
  const publicKey = privateKey.toPublicKey()
  const address = publicKey.toAddress()
  
  console.log('Test address:', address)
  console.log('Note: Fund this address with small amounts for testing')
  
  // Test your broadcaster
  const broadcaster = new CustomHTTPBroadcaster('https://api.test.com/broadcast')
  
  // Test error handling
  try {
    const tx = new Transaction() // Empty transaction for testing error handling
    const result = await broadcaster.broadcast(tx)
    console.log('Test result:', result)
  } catch (error) {
    console.log('Test error (expected):', error)
  }
}
```

## Best Practices

### 1. Error Handling

- Always implement proper error handling
- Use specific error codes for different failure types
- Provide meaningful error messages

### 2. Retry Logic

- Implement exponential backoff for retries
- Don't retry on client errors (4xx) except rate limiting (429)
- Set reasonable retry limits

### 3. Logging

- Log all broadcast attempts for debugging
- Include transaction IDs and error details
- Use structured logging for production

### 4. Security

- Never log sensitive data (API keys, private keys)
- Use environment variables for configuration
- Validate all inputs

### 5. Testing

- Test with small amounts first
- Test error scenarios
- Use testnet for development

## Common Broadcasting Service APIs

Different services have different API formats. Here are common patterns:

### JSON-RPC Style

```typescript
body: JSON.stringify({
  method: 'sendrawtransaction',
  params: [txHex]
})
```

### REST Style

```typescript
body: JSON.stringify({
  txhex: txHex
})
```

### Form Data Style

```typescript
body: new URLSearchParams({
  tx: txHex
})
```

Adapt the `CustomHTTPBroadcaster` implementation based on your target service's API format.

## Related Guides

- [Transaction Broadcasting Tutorial](../tutorials/transaction-broadcasting.md) - Basic broadcasting concepts
- [Error Handling and Edge Cases](../tutorials/error-handling.md) - Comprehensive error handling
