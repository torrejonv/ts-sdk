# Configuring HTTP Clients

This guide covers how to configure HTTP clients for use with the BSV TypeScript SDK, focusing on Axios and alternatives for general HTTP operations, transaction broadcasting, and SDK infrastructure.

## When to Use This Guide

**Use this guide when you need:**

- Custom HTTP client setup for SDK operations (Axios, fetch, etc.)
- Transaction broadcasting via ARC endpoints
- Environment-specific HTTP configuration (timeouts, retries, headers)
- Testing and mocking HTTP clients for SDK functionality
- Integration with existing HTTP infrastructure

**For authenticated peer-to-peer communication, use [AuthFetch Tutorial](../tutorials/authfetch-tutorial.md) instead:**

- BRC-103/104 cryptographic authentication
- Wallet-signed HTTP requests
- Certificate-based peer verification
- Secure application-to-application communication

> **📚 Related Concepts**: This guide relates to [Chain Tracking](../concepts/chain-tracking.md) and [SDK Design Philosophy](../concepts/sdk-philosophy.md) for understanding network interaction patterns.

## Using Axios with the SDK

The BSV TypeScript SDK allows you to provide your own HTTP client implementation for network requests. This is particularly useful when you need custom configuration for transaction broadcasting, network queries, or when working in specific environments.

### Basic Axios Setup

```typescript
import axios from 'axios'
import { BSV } from '@bsv/sdk'

// Create a configured Axios instance
const customAxios = axios.create({
  timeout: 10000, // 10 seconds
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json'
  }
})

// Use the custom client when broadcasting transactions
const broadcastTransaction = async (tx) => {
  try {
    // Create a simple transaction with P2PKH output
    const tx = new Transaction()
    const privateKey = PrivateKey.fromRandom()
    const publicKey = privateKey.toPublicKey()
    const address = publicKey.toAddress()
    
    // Add an output using P2PKH (instantiate the class first)
    const p2pkh = new P2PKH()
    const lockingScript = p2pkh.lock(address)
    tx.addOutput({
      satoshis: 100,
      lockingScript
    })
    
    // Create a simple transaction with P2PKH output
    const tx = new Transaction()
    const privateKey = PrivateKey.fromRandom()
    const publicKey = privateKey.toPublicKey()
    const address = publicKey.toAddress()
    
    // Add an output using P2PKH (instantiate the class first)
    const p2pkh = new P2PKH()
    const lockingScript = p2pkh.lock(address)
    tx.addOutput({
      satoshis: 100,
      lockingScript
    })
    
    // Convert the transaction to hex format
    const txHex = tx.toHex()
    
    // Use your custom axios instance for the request
    const response = await customAxios.post('https://api.example.com/v1/tx/broadcast', {
      rawTx: txHex
    })
    
    return response.data
  } catch (error) {
    console.error('Error broadcasting transaction:', error)
    throw error
  }
}
```

### Using Axios with ARC Broadcaster

The ARC (Alternative Revenue Channel) broadcaster can be configured with a custom HTTP client:

```typescript
import axios from 'axios'
import { ARC } from '@bsv/sdk'

// Configure Axios
const customAxios = axios.create({
  timeout: 15000,
  headers: {
    'Authorization': 'Bearer YOUR_API_KEY',
    'Content-Type': 'application/json'
  }
})

// Create an adapter to make Axios compatible with HttpClient interface
class AxiosAdapter {
  constructor(private axiosInstance: any) {}
  
  async request(url: string, options: any = {}) {
    const response = await this.axiosInstance({
      url,
      method: options.method || 'GET',
      data: options.body,
      headers: options.headers
    })
    return response.data
  }
}

// Create an adapter to make Axios compatible with HttpClient interface
class AxiosAdapter {
  constructor(private axiosInstance: any) {}
  
  async request(url: string, options: any = {}) {
    const response = await this.axiosInstance({
      url,
      method: options.method || 'GET',
      data: options.body,
      headers: options.headers
    })
    return response.data
  }
}

// Create an ARC instance with custom HTTP client
const httpClient = new AxiosAdapter(customAxios)
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'YOUR_API_KEY',
  httpClient
const httpClient = new AxiosAdapter(customAxios)
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'YOUR_API_KEY',
  httpClient
})

// Use the configured ARC instance to broadcast a transaction
const broadcastWithARC = async (tx) => {
  try {
    // ARC expects hex format
    const txid = await arc.broadcast(tx.toHex())
    return txid
  } catch (error) {
    console.error('ARC broadcast error:', error)
    throw error
  }
}
```

## Advanced HTTP Client Configuration

### Handling Timeouts and Retries

```typescript
import axios from 'axios'
import axiosRetry from 'axios-retry'

// Create a custom Axios instance
const client = axios.create({
  timeout: 30000 // 30 seconds
})

// Configure automatic retries
axiosRetry(client, {
  retries: 3,
  retryDelay: axiosRetry.exponentialDelay,
  retryCondition: (error: any) => {
  retryCondition: (error: any) => {
    // Retry on network errors or 5xx responses
    return axiosRetry.isNetworkOrIdempotentRequestError(error) || 
           (error.response && error.response.status >= 500)
  }
})

// Add request interceptor for logging
client.interceptors.request.use((request: any) => {
client.interceptors.request.use((request: any) => {
  console.log('Starting request:', request.url)
  return request
})

// Add response interceptor for error handling
client.interceptors.response.use(
  response => response,
  error => {
    if (error.response) {
      console.error('Server responded with error:', error.response.status, error.response.data)
    } else if (error.request) {
      console.error('No response received:', error.request)
    } else {
      console.error('Error setting up request:', error.message)
    }
    return Promise.reject(error)
  }
)
```

### Environment-specific Configuration

```typescript
import axios from 'axios'

const getConfiguredClient = (environment: 'production' | 'staging' | 'development' = 'production') => {
  const baseURLs: Record<string, string> = {
const getConfiguredClient = (environment: 'production' | 'staging' | 'development' = 'production') => {
  const baseURLs: Record<string, string> = {
    production: 'https://api.taal.com',
    staging: 'https://api-staging.taal.com',
    development: 'http://localhost:3000'
  }

  const timeouts: Record<string, number> = {
  const timeouts: Record<string, number> = {
    production: 10000,
    staging: 15000,
    development: 30000
  }

  return axios.create({
    baseURL: baseURLs[environment],
    timeout: timeouts[environment],
    headers: {
      'Content-Type': 'application/json'
    }
  })
}

const productionClient = getConfiguredClient('production')
const developmentClient = getConfiguredClient('development')
```

## SDK Built-in HTTP Clients

The BSV TypeScript SDK comes with built-in HTTP client implementations that you can use directly. The SDK automatically selects the appropriate client based on your environment through the `defaultHttpClient()` function.

### Using NodejsHttpClient

The SDK includes a Node.js-specific HTTP client implementation that uses the Node.js `https` module:

```typescript
import { NodejsHttpClient } from '@bsv/sdk'
import https from 'https'

// Create a NodejsHttpClient instance
const nodeClient = new NodejsHttpClient(https)

// Use with ARC
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'your-api-key',
  httpClient: nodeClient
})

// Example of broadcasting a transaction
const broadcastTx = async (tx) => {
  try {
    const txid = await arc.broadcast(tx)
    console.log('Transaction broadcast successful. TXID:', txid)
    return txid
  } catch (error) {
    console.error('Error broadcasting transaction:', error)
    throw error
  }
}
```

### Using Built-in FetchHttpClient

In browser environments, the SDK provides a `FetchHttpClient` implementation that uses the Fetch API:

```typescript
import { FetchHttpClient, ARC } from '@bsv/sdk'

// Create a FetchHttpClient instance with custom fetch options
const fetchClient = new FetchHttpClient(window.fetch.bind(window))

// Use with ARC
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'your-api-key',
  httpClient: fetchClient
})
```

### Using defaultHttpClient

The SDK provides a `defaultHttpClient()` function that automatically selects the appropriate HTTP client based on the environment:

```typescript
import { defaultHttpClient, ARC } from '@bsv/sdk'

// Get the default HTTP client for the current environment
const client = defaultHttpClient()

// Use with ARC
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'your-api-key',
  httpClient: client
})
```

## Alternative HTTP Clients

While the SDK provides built-in HTTP clients and Axios is commonly used, you can implement your own HTTP clients with the BSV TypeScript SDK:

### Using Fetch API

```typescript
import { ARC } from '@bsv/sdk'

// Create a fetch-based HTTP client that implements HttpClient interface
class CustomFetchClient {
  async request(url: string, options: any = {}) {
// Create a fetch-based HTTP client that implements HttpClient interface
class CustomFetchClient {
  async request(url: string, options: any = {}) {
    const response = await fetch(url, {
      method: options.method || 'GET',
      method: options.method || 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        ...options.headers
      },
      body: options.body ? JSON.stringify(options.body) : undefined
      },
      body: options.body ? JSON.stringify(options.body) : undefined
    })
    
    if (!response.ok) {
      const errorText = await response.text()
      throw new Error(`HTTP error ${response.status}: ${errorText}`)
    }
    
    return await response.json()
  }
}

// Use with ARC
const fetchClient = new CustomFetchClient()
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'your-api-key',
const fetchClient = new CustomFetchClient()
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'your-api-key',
  httpClient: fetchClient
})
```

## Testing and Mocking HTTP Clients

When testing your application, you may want to mock HTTP responses:

```typescript
import { ARC } from '@bsv/sdk'

// Create a mock HTTP client for testing that implements HttpClient interface
class MockHttpClient {
  request = jest.fn().mockImplementation(async (url: string, options: any = {}) => {
    if (options.method === 'POST' && url.includes('/tx')) {
      return { txid: '1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef' }
    }
    return { status: 'confirmed' }
  })
// Create a mock HTTP client for testing that implements HttpClient interface
class MockHttpClient {
  request = jest.fn().mockImplementation(async (url: string, options: any = {}) => {
    if (options.method === 'POST' && url.includes('/tx')) {
      return { txid: '1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef' }
    }
    return { status: 'confirmed' }
  })
}

// Create an ARC instance with the mock client
const mockClient = new MockHttpClient()
const arc = new ARC('https://api.example.com/arc', {
  apiKey: 'test-api-key',
  httpClient: mockClient
const mockClient = new MockHttpClient()
const arc = new ARC('https://api.example.com/arc', {
  apiKey: 'test-api-key',
  httpClient: mockClient
})

// Test transaction broadcasting
const testBroadcast = async () => {
  const mockTxHex = '0100000001...'
  const result = await arc.broadcast(mockTxHex)
  
  // Verify the mock was called correctly
  expect(mockClient.request).toHaveBeenCalledWith(
    expect.stringContaining('/tx'),
    expect.objectContaining({
      method: 'POST',
      body: expect.objectContaining({ rawTx: mockTxHex })
    })
  expect(mockClient.request).toHaveBeenCalledWith(
    expect.stringContaining('/tx'),
    expect.objectContaining({
      method: 'POST',
      body: expect.objectContaining({ rawTx: mockTxHex })
    })
  )
  
  return result
}
```

## Implementing a Custom HTTP Client

You can create your own HTTP client implementation by implementing the `HttpClient` interface from the SDK. This gives you complete control over how HTTP requests are handled:

```typescript
import { HttpClient, HttpClientResponse, HttpClientRequestOptions, ARC, Transaction, PrivateKey, P2PKH } from '@bsv/sdk'
import { HttpClient, HttpClientResponse, HttpClientRequestOptions, ARC, Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

// Implement the HttpClient interface
class CustomHttpClient implements HttpClient {
  constructor(private readonly options: { timeout?: number } = {}) {}
  
  async request<T = any>(
    url: string,
    options: HttpClientRequestOptions
  ): Promise<HttpClientResponse<T>> {
    console.log(`Making ${options.method} request to ${url}`)
    
    try {
      // Set up timeout
      const timeout = this.options.timeout || 10000
      const controller = new AbortController()
      const timeoutId = setTimeout(() => controller.abort(), timeout)
      
      // Prepare fetch options
      const fetchOptions: RequestInit = {
        method: options.method || 'GET',
        headers: options.headers,
        signal: controller.signal,
        body: options.data ? JSON.stringify(options.data) : undefined
      }
      
      // Make the request
      const response = await fetch(url, fetchOptions)
      clearTimeout(timeoutId)
      
      // Parse response
      let data: any
      const contentType = response.headers.get('content-type')
      if (contentType && contentType.includes('application/json')) {
        data = await response.json()
      } else {
        data = await response.text()
      }
      
      // Return formatted response
      return {
        status: response.status,
        statusText: response.statusText,
        ok: response.ok,
        data
      }
    } catch (error) {
      console.error('Request failed:', error)
      return {
        status: 500,
        statusText: error.message || 'Request failed',
        ok: false,
        data: { error: error.message }
      }
    }
  }
}

// Use the custom client with ARC
const customClient = new CustomHttpClient({ timeout: 15000 })
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'your-api-key',
  httpClient: customClient
})

// Example broadcasting a transaction with the custom client
const broadcastTx = async () => {
const broadcastTx = async () => {
  try {
    // Create a simple transaction with P2PKH output
    const tx = new Transaction()
    const privateKey = PrivateKey.fromRandom()
    const publicKey = privateKey.toPublicKey()
    const address = publicKey.toAddress()
    
    // Add an output using P2PKH (instantiate the class first)
    const p2pkh = new P2PKH()
    const lockingScript = p2pkh.lock(address)
    tx.addOutput({
      satoshis: 100,
      lockingScript
    })
    
    // Broadcast the transaction
    // Create a simple transaction with P2PKH output
    const tx = new Transaction()
    const privateKey = PrivateKey.fromRandom()
    const publicKey = privateKey.toPublicKey()
    const address = publicKey.toAddress()
    
    // Add an output using P2PKH (instantiate the class first)
    const p2pkh = new P2PKH()
    const lockingScript = p2pkh.lock(address)
    tx.addOutput({
      satoshis: 100,
      lockingScript
    })
    
    // Broadcast the transaction
    const result = await arc.broadcast(tx)
    
    // Transaction ID needs specific handling
    console.log('Transaction broadcast successful. TXID:', result.txid)
    return result
  } catch (error) {
    console.error('Error broadcasting transaction:', error)
    throw error
  }
}
```

## Best Practices

1. **Always set timeouts** - Network requests can hang indefinitely without proper timeouts
2. **Implement retries** - Especially for transaction broadcasting, retries can improve reliability
3. **Add proper error handling** - Parse and handle HTTP errors appropriately
4. **Configure request logging** - Log requests and responses for debugging purposes
5. **Use environment variables** - Store API keys and endpoints in environment variables
6. **Consider rate limiting** - Implement backoff strategies for rate-limited APIs
7. **Use the built-in clients** - The SDK's `defaultHttpClient()` handles environment detection automatically

## Related Documentation

### For Authenticated Communication

- **[AuthFetch Tutorial](../tutorials/authfetch-tutorial.md)** - Use for BRC-103/104 cryptographic authentication, wallet-signed requests, and secure peer-to-peer communication

### For Advanced HTTP Scenarios

- **[Error Handling Guide](error-handling.md)** - Comprehensive error handling patterns for HTTP operations
- **[Chain Tracking](../concepts/chain-tracking.md)** - Understanding network interaction patterns
- **[SDK Design Philosophy](../concepts/sdk-philosophy.md)** - Core principles behind SDK HTTP client design

### For Transaction Broadcasting

- **[Transaction Broadcasting Tutorial](../tutorials/transaction-broadcasting.md)** - Step-by-step transaction broadcasting examples

---

**Summary**: This guide covers infrastructure-level HTTP client configuration for SDK operations. For application-level authenticated communication using BSV cryptographic protocols, see the [AuthFetch Tutorial](../tutorials/authfetch-tutorial.md).

## Related Resources

- [Axios Documentation](https://axios-http.com/docs/intro)
- [ARC API Reference](../reference/arc.md)
- [Transaction Broadcasting Guide](./transaction-monitoring.md)
