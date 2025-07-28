# ARC Configuration Reference

Configuration for ARC (Authoritative Response Component) - a transaction submission service run by miners.

## What is ARC?

ARC is a simple API service run by miners for transaction submission. It provides:

- Transaction submission endpoint
- Transaction status queries
- Fee rate information

## Basic Configuration

The ARC broadcaster in the BSV SDK requires minimal configuration:

```typescript
import { ARC } from '@bsv/sdk'

// Basic ARC configuration
const arc = new ARC('https://arc.taal.com', { 
  apiKey: 'your-api-key' // Optional, depending on the miner
})
```

## Configuration Options

```typescript
interface ArcConfig {
  // Optional API key for authenticated endpoints
  apiKey?: string
  
  // HTTP client for making requests (optional)
  httpClient?: HttpClient
  
  // Deployment ID for tracking API calls (auto-generated if not provided)
  deploymentId?: string
  
  // Callback URL for transaction notifications
  callbackUrl?: string
  
  // Authentication token for callback endpoint
  callbackToken?: string
  
  // Additional headers for all requests
  headers?: Record<string, string>
}
```

## Common ARC Endpoints

Different miners provide ARC services at various URLs:

```typescript
// Mainnet
const mainnetARCs = {
  taal: 'https://arc.taal.com'
  // Add other confirmed ARC endpoints here
}

// Testnet
const testnetARCs = {
  taal: 'https://arc-test.taal.com'
}
```

## Using ARC

### Broadcast a Transaction

```typescript
import { Transaction, ARC } from '@bsv/sdk'

const arc = new ARC('https://arc.taal.com')

// Build and broadcast transaction
const tx = new Transaction()
// ... build your transaction

const response = await arc.broadcast(tx)
if (response.status === 'success') {
  console.log('Transaction broadcasted:', response.txid)
} else {
  console.error('Broadcast failed:', response.description)
}
```

### Broadcast Multiple Transactions

```typescript
// Broadcast multiple transactions at once
const transactions = [tx1, tx2, tx3]
const responses = await arc.broadcastMany(transactions)
responses.forEach((response, index) => {
  console.log(`Transaction ${index} result:`, response)
})
```

## Multiple ARC Services

For reliability, you can configure fallback ARC services:

```typescript
// Primary ARC
const primaryARC = new ARC('https://arc.taal.com', { apiKey: 'primary-key' })

// Fallback ARC (example - replace with actual ARC endpoint)
const fallbackARC = new ARC('https://arc.example.com', { apiKey: 'fallback-key' })

// Broadcast with fallback
try {
  const response = await primaryARC.broadcast(tx)
  if (response.status === 'success') {
    return response
  }
} catch (error) {
  console.log('Primary ARC failed, trying fallback...')
}

// Try fallback ARC
const fallbackResponse = await fallbackARC.broadcast(tx)
return fallbackResponse
```

## Environment Variables

Configure ARC using environment variables:

```typescript
const arc = new ARC(
  process.env.ARC_URL || 'https://arc.taal.com',
  { 
    apiKey: process.env.ARC_API_KEY,
    deploymentId: process.env.ARC_DEPLOYMENT_ID,
    callbackUrl: process.env.ARC_CALLBACK_URL,
    callbackToken: process.env.ARC_CALLBACK_TOKEN
  }
)
```

## Error Handling

Handle common ARC errors:

```typescript
const response = await arc.broadcast(tx)

if (response.status === 'success') {
  console.log('Transaction broadcasted successfully:', response.txid)
} else {
  // Handle broadcast failure
  console.error('Broadcast failed:', response.description)
  console.error('Error code:', response.code)
  
  // Check for specific error types
  if (response.code === 'INVALID_TRANSACTION') {
    console.error('Transaction was rejected by the network')
  } else if (response.code === 'DOUBLE_SPEND') {
    console.error('Transaction conflicts with existing transaction')
  }
}
```

## Best Practices

1. **Use HTTPS**: Always use secure connections to ARC services
2. **Handle Errors**: Implement proper error handling and retries
3. **Respect Rate Limits**: Don't overwhelm ARC services with requests
4. **Keep Keys Secure**: Never hardcode API keys in your source code

## See Also

- [Transaction Broadcasting Guide](../guides/transaction-broadcasting.md)
- [Custom Broadcasters](../guides/custom-broadcasters.md)
- [Transaction API Reference](./transaction-api.md)
