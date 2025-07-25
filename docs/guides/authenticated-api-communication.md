# Setting up Authenticated API Communication

Learn how to implement secure, authenticated API communication using AuthFetch and BRC-103/104 protocols.

## Problem

You need to build secure API communication between services in the BSV ecosystem using cryptographic authentication instead of traditional token-based systems.

## Solution

Use AuthFetch with proper certificate management and session handling for authenticated HTTP requests.

### Basic Authenticated API Setup

```typescript
import { AuthFetch, WalletClient } from '@bsv/sdk'

class AuthenticatedAPIClient {
  private authFetch: AuthFetch
  private baseURL: string
  
  constructor(baseURL: string, wallet?: WalletClient) {
    this.baseURL = baseURL
    this.authFetch = new AuthFetch(wallet || new WalletClient('auto', 'localhost'))
  }
  
  async makeAuthenticatedRequest(endpoint: string, options: any = {}) {
    const url = `${this.baseURL}${endpoint}`
    
    try {
      const response = await this.authFetch.fetch(url, {
        method: options.method || 'GET',
        headers: {
          'Content-Type': 'application/json',
          ...options.headers
        },
        body: options.body ? JSON.stringify(options.body) : undefined
      })
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }
      
      return await response.json()
    } catch (error) {
      console.error('Authenticated request failed:', error)
      throw error
    }
  }
}
```

### Certificate-Based Authentication

```typescript
import { AuthFetch, WalletClient } from '@bsv/sdk'

class CertificateAuthAPI {
  private authFetch: AuthFetch
  
  constructor(requiredCertificates: any) {
    const wallet = new WalletClient('auto', 'localhost')
    this.authFetch = new AuthFetch(wallet, requiredCertificates)
  }
  
  async authenticatedCall(url: string, data: any) {
    const response = await this.authFetch.fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    })
    
    // Access received certificates
    const certificates = this.authFetch.consumeReceivedCertificates()
    
    return {
      data: await response.json(),
      certificates
    }
  }
}
```

## Best Practices

1. **Always verify peer certificates** before processing responses
2. **Implement proper retry logic** for authentication failures
3. **Cache authentication sessions** appropriately
4. **Use secure certificate storage** and validation

## Common Issues

- **Certificate validation failures**: Ensure proper certificate chain validation
- **Session expiration**: Implement automatic session renewal
- **Network timeouts**: Add appropriate timeout and retry mechanisms

## Related

- [AuthFetch Tutorial](../tutorials/authfetch-tutorial.md)
- [Security Best Practices](./security-best-practices.md)
