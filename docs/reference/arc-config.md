# ARC Configuration Reference

Complete reference for configuring ARC (Application Resource Center) connections and settings in the BSV TypeScript SDK.

## ARC Configuration Interface

### ARCConfig

```typescript
interface ARCConfig {
  url: string
  apiKey?: string
  deploymentId?: string
  timeout?: number
  retries?: number
  retryDelay?: number
  headers?: Record<string, string>
  authentication?: ARCAuthentication
  endpoints?: ARCEndpoints
  limits?: ARCLimits
  monitoring?: ARCMonitoring
}
```

### ARCAuthentication

```typescript
interface ARCAuthentication {
  type: 'apiKey' | 'bearer' | 'basic' | 'custom'
  credentials: {
    apiKey?: string
    token?: string
    username?: string
    password?: string
    custom?: Record<string, any>
  }
  refreshToken?: string
  expiresAt?: number
  autoRefresh?: boolean
}
```

### ARCEndpoints

```typescript
interface ARCEndpoints {
  submit?: string
  query?: string
  status?: string
  health?: string
  fees?: string
  utxos?: string
  balance?: string
  history?: string
  broadcast?: string
  merkleProof?: string
}
```

### ARCLimits

```typescript
interface ARCLimits {
  maxTransactionSize?: number
  maxBatchSize?: number
  maxConcurrentRequests?: number
  rateLimit?: {
    requests: number
    window: number // milliseconds
  }
  quotas?: {
    daily?: number
    monthly?: number
  }
}
```

### ARCMonitoring

```typescript
interface ARCMonitoring {
  enabled: boolean
  healthCheck?: {
    interval: number // milliseconds
    timeout: number
    retries: number
  }
  metrics?: {
    enabled: boolean
    endpoint?: string
    interval: number
  }
  alerts?: {
    enabled: boolean
    thresholds: {
      errorRate: number
      responseTime: number
      availability: number
    }
  }
}
```

## Predefined ARC Configurations

### Mainnet ARC Providers

```typescript
const MAINNET_ARC_CONFIGS: Record<string, ARCConfig> = {
  taal: {
    url: 'https://arc.taal.com',
    timeout: 30000,
    retries: 3,
    retryDelay: 1000,
    endpoints: {
      submit: '/v1/tx',
      query: '/v1/tx/{txid}',
      status: '/v1/tx/{txid}/status',
      health: '/v1/health',
      fees: '/v1/policy/fees',
      broadcast: '/v1/tx/broadcast'
    },
    limits: {
      maxTransactionSize: 10000000, // 10MB
      maxBatchSize: 100,
      maxConcurrentRequests: 10,
      rateLimit: {
        requests: 1000,
        window: 60000 // 1 minute
      }
    },
    monitoring: {
      enabled: true,
      healthCheck: {
        interval: 30000,
        timeout: 5000,
        retries: 3
      }
    }
  },
  
  gorillapool: {
    url: 'https://arc.gorillapool.io',
    timeout: 30000,
    retries: 3,
    retryDelay: 1000,
    endpoints: {
      submit: '/v1/tx',
      query: '/v1/tx/{txid}',
      status: '/v1/tx/{txid}/status',
      health: '/v1/health',
      fees: '/v1/policy/fees'
    },
    limits: {
      maxTransactionSize: 10000000,
      maxBatchSize: 50,
      maxConcurrentRequests: 5,
      rateLimit: {
        requests: 500,
        window: 60000
      }
    },
    monitoring: {
      enabled: true,
      healthCheck: {
        interval: 60000,
        timeout: 10000,
        retries: 2
      }
    }
  }
}
```

### Testnet ARC Providers

```typescript
const TESTNET_ARC_CONFIGS: Record<string, ARCConfig> = {
  taal_testnet: {
    url: 'https://arc-testnet.taal.com',
    timeout: 30000,
    retries: 3,
    retryDelay: 1000,
    endpoints: {
      submit: '/v1/tx',
      query: '/v1/tx/{txid}',
      status: '/v1/tx/{txid}/status',
      health: '/v1/health',
      fees: '/v1/policy/fees'
    },
    limits: {
      maxTransactionSize: 10000000,
      maxBatchSize: 100,
      maxConcurrentRequests: 20,
      rateLimit: {
        requests: 2000,
        window: 60000
      }
    },
    monitoring: {
      enabled: true,
      healthCheck: {
        interval: 30000,
        timeout: 5000,
        retries: 3
      }
    }
  }
}
```

## ARC Configuration Builder

### ARCConfigBuilder Class

```typescript
class ARCConfigBuilder {
  private config: Partial<ARCConfig> = {}
  
  url(url: string): ARCConfigBuilder {
    this.config.url = url
    return this
  }
  
  apiKey(apiKey: string): ARCConfigBuilder {
    this.config.apiKey = apiKey
    return this
  }
  
  timeout(timeout: number): ARCConfigBuilder {
    this.config.timeout = timeout
    return this
  }
  
  retries(retries: number, delay?: number): ARCConfigBuilder {
    this.config.retries = retries
    if (delay !== undefined) {
      this.config.retryDelay = delay
    }
    return this
  }
  
  authentication(auth: ARCAuthentication): ARCConfigBuilder {
    this.config.authentication = auth
    return this
  }
  
  endpoints(endpoints: Partial<ARCEndpoints>): ARCConfigBuilder {
    this.config.endpoints = { ...this.config.endpoints, ...endpoints }
    return this
  }
  
  limits(limits: Partial<ARCLimits>): ARCConfigBuilder {
    this.config.limits = { ...this.config.limits, ...limits }
    return this
  }
  
  monitoring(monitoring: Partial<ARCMonitoring>): ARCConfigBuilder {
    this.config.monitoring = { ...this.config.monitoring, ...monitoring }
    return this
  }
  
  headers(headers: Record<string, string>): ARCConfigBuilder {
    this.config.headers = { ...this.config.headers, ...headers }
    return this
  }
  
  build(): ARCConfig {
    if (!this.config.url) {
      throw new Error('ARC URL is required')
    }
    
    return {
      url: this.config.url,
      timeout: this.config.timeout || 30000,
      retries: this.config.retries || 3,
      retryDelay: this.config.retryDelay || 1000,
      ...this.config
    } as ARCConfig
  }
}
```

### Usage Examples

```typescript
// Basic ARC configuration
const basicARC = new ARCConfigBuilder()
  .url('https://arc.example.com')
  .apiKey('your-api-key')
  .timeout(15000)
  .build()

// Advanced ARC configuration with authentication
const advancedARC = new ARCConfigBuilder()
  .url('https://arc.example.com')
  .authentication({
    type: 'bearer',
    credentials: {
      token: 'your-bearer-token'
    },
    autoRefresh: true
  })
  .endpoints({
    submit: '/api/v2/transactions',
    query: '/api/v2/transactions/{txid}',
    status: '/api/v2/status/{txid}'
  })
  .limits({
    maxTransactionSize: 5000000,
    maxBatchSize: 50,
    rateLimit: {
      requests: 100,
      window: 60000
    }
  })
  .monitoring({
    enabled: true,
    healthCheck: {
      interval: 30000,
      timeout: 5000,
      retries: 3
    },
    metrics: {
      enabled: true,
      interval: 60000
    }
  })
  .build()
```

## ARC Client Configuration

### ARCClient Interface

```typescript
interface ARCClient {
  config: ARCConfig
  submit(transaction: Transaction): Promise<ARCResponse>
  query(txid: string): Promise<ARCTransactionStatus>
  getStatus(txid: string): Promise<ARCStatus>
  getFees(): Promise<ARCFeeQuote>
  getHealth(): Promise<ARCHealthStatus>
  broadcast(rawTx: string): Promise<ARCBroadcastResponse>
}
```

### ARCResponse Types

```typescript
interface ARCResponse {
  txid: string
  status: 'success' | 'error' | 'pending'
  message?: string
  timestamp: number
  blockHash?: string
  blockHeight?: number
  merklePath?: string
}

interface ARCTransactionStatus {
  txid: string
  status: 'seen' | 'mined' | 'confirmed' | 'rejected'
  blockHash?: string
  blockHeight?: number
  timestamp: number
  confirmations?: number
}

interface ARCFeeQuote {
  standard: number
  data: number
  timestamp: number
  expiresAt: number
}

interface ARCHealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy'
  version: string
  timestamp: number
  checks: {
    database: boolean
    network: boolean
    mempool: boolean
  }
}
```

## Multi-ARC Configuration

### ARC Pool Configuration

```typescript
interface ARCPoolConfig {
  primary: ARCConfig
  fallbacks: ARCConfig[]
  strategy: 'failover' | 'round-robin' | 'load-balance'
  healthCheck: {
    enabled: boolean
    interval: number
    timeout: number
  }
  failover: {
    maxRetries: number
    retryDelay: number
    backoffMultiplier: number
  }
}

class ARCPool {
  private configs: ARCConfig[]
  private currentIndex = 0
  private healthStatus = new Map<string, boolean>()
  
  constructor(private poolConfig: ARCPoolConfig) {
    this.configs = [poolConfig.primary, ...poolConfig.fallbacks]
    this.startHealthChecks()
  }
  
  async submit(transaction: Transaction): Promise<ARCResponse> {
    const config = this.selectARC()
    const client = new ARCClient(config)
    
    try {
      return await client.submit(transaction)
    } catch (error) {
      this.markUnhealthy(config.url)
      
      if (this.hasHealthyFallbacks()) {
        return this.submit(transaction) // Retry with fallback
      }
      
      throw error
    }
  }
  
  private selectARC(): ARCConfig {
    switch (this.poolConfig.strategy) {
      case 'failover':
        return this.selectFailover()
      case 'round-robin':
        return this.selectRoundRobin()
      case 'load-balance':
        return this.selectLoadBalanced()
      default:
        return this.configs[0]
    }
  }
  
  private selectFailover(): ARCConfig {
    for (const config of this.configs) {
      if (this.healthStatus.get(config.url) !== false) {
        return config
      }
    }
    return this.configs[0] // Fallback to primary
  }
  
  private selectRoundRobin(): ARCConfig {
    const healthyConfigs = this.configs.filter(
      config => this.healthStatus.get(config.url) !== false
    )
    
    if (healthyConfigs.length === 0) {
      return this.configs[0]
    }
    
    const config = healthyConfigs[this.currentIndex % healthyConfigs.length]
    this.currentIndex++
    return config
  }
  
  private selectLoadBalanced(): ARCConfig {
    // Simple load balancing based on response times
    const healthyConfigs = this.configs.filter(
      config => this.healthStatus.get(config.url) !== false
    )
    
    if (healthyConfigs.length === 0) {
      return this.configs[0]
    }
    
    // Return random healthy config (can be enhanced with actual load metrics)
    return healthyConfigs[Math.floor(Math.random() * healthyConfigs.length)]
  }
  
  private startHealthChecks(): void {
    if (!this.poolConfig.healthCheck.enabled) return
    
    setInterval(async () => {
      for (const config of this.configs) {
        try {
          const client = new ARCClient(config)
          await client.getHealth()
          this.healthStatus.set(config.url, true)
        } catch (error) {
          this.healthStatus.set(config.url, false)
        }
      }
    }, this.poolConfig.healthCheck.interval)
  }
  
  private markUnhealthy(url: string): void {
    this.healthStatus.set(url, false)
  }
  
  private hasHealthyFallbacks(): boolean {
    return this.configs.some(config => 
      this.healthStatus.get(config.url) !== false
    )
  }
}
```

## Environment-Based ARC Configuration

### Environment Configuration

```typescript
class ARCEnvironmentConfig {
  static fromEnvironment(): ARCConfig {
    const config: ARCConfig = {
      url: process.env.BSV_ARC_URL || 'https://arc.taal.com',
      apiKey: process.env.BSV_ARC_API_KEY,
      timeout: parseInt(process.env.BSV_ARC_TIMEOUT || '30000'),
      retries: parseInt(process.env.BSV_ARC_RETRIES || '3'),
      retryDelay: parseInt(process.env.BSV_ARC_RETRY_DELAY || '1000')
    }
    
    // Add authentication if provided
    if (process.env.BSV_ARC_AUTH_TYPE) {
      config.authentication = {
        type: process.env.BSV_ARC_AUTH_TYPE as any,
        credentials: {
          apiKey: process.env.BSV_ARC_API_KEY,
          token: process.env.BSV_ARC_TOKEN,
          username: process.env.BSV_ARC_USERNAME,
          password: process.env.BSV_ARC_PASSWORD
        }
      }
    }
    
    // Add custom headers
    const headerPrefix = 'BSV_ARC_HEADER_'
    const headers: Record<string, string> = {}
    
    Object.keys(process.env).forEach(key => {
      if (key.startsWith(headerPrefix)) {
        const headerName = key.substring(headerPrefix.length).toLowerCase()
        headers[headerName] = process.env[key]!
      }
    })
    
    if (Object.keys(headers).length > 0) {
      config.headers = headers
    }
    
    return config
  }
  
  static validate(config: ARCConfig): void {
    if (!config.url) {
      throw new Error('ARC URL is required')
    }
    
    if (!config.url.startsWith('http')) {
      throw new Error('ARC URL must be a valid HTTP/HTTPS URL')
    }
    
    if (config.timeout && config.timeout < 1000) {
      throw new Error('ARC timeout must be at least 1000ms')
    }
    
    if (config.retries && config.retries < 0) {
      throw new Error('ARC retries must be non-negative')
    }
    
    if (config.authentication?.type === 'apiKey' && !config.authentication.credentials.apiKey) {
      throw new Error('API key is required for apiKey authentication')
    }
    
    if (config.authentication?.type === 'bearer' && !config.authentication.credentials.token) {
      throw new Error('Bearer token is required for bearer authentication')
    }
  }
}
```

## ARC Configuration Best Practices

### Security Best Practices

```typescript
// 1. Never hardcode API keys
const secureConfig = new ARCConfigBuilder()
  .url(process.env.ARC_URL!)
  .apiKey(process.env.ARC_API_KEY!) // Use environment variables
  .build()

// 2. Use HTTPS URLs only
const httpsConfig = new ARCConfigBuilder()
  .url('https://arc.example.com') // Always use HTTPS
  .build()

// 3. Implement proper authentication rotation
const rotatingAuthConfig = new ARCConfigBuilder()
  .url('https://arc.example.com')
  .authentication({
    type: 'bearer',
    credentials: { token: getCurrentToken() },
    autoRefresh: true,
    expiresAt: getTokenExpiry()
  })
  .build()
```

### Performance Best Practices

```typescript
// 1. Configure appropriate timeouts
const performantConfig = new ARCConfigBuilder()
  .url('https://arc.example.com')
  .timeout(15000) // 15 seconds for most operations
  .retries(3)
  .build()

// 2. Use connection pooling for high-throughput applications
const pooledConfig: ARCPoolConfig = {
  primary: MAINNET_ARC_CONFIGS.taal,
  fallbacks: [MAINNET_ARC_CONFIGS.gorillapool],
  strategy: 'load-balance',
  healthCheck: {
    enabled: true,
    interval: 30000,
    timeout: 5000
  },
  failover: {
    maxRetries: 3,
    retryDelay: 1000,
    backoffMultiplier: 2
  }
}

// 3. Monitor and optimize rate limits
const rateLimitedConfig = new ARCConfigBuilder()
  .url('https://arc.example.com')
  .limits({
    rateLimit: {
      requests: 100,
      window: 60000 // 100 requests per minute
    },
    maxConcurrentRequests: 5
  })
  .build()
```

### Reliability Best Practices

```typescript
// 1. Always configure fallbacks
const reliableConfig: ARCPoolConfig = {
  primary: MAINNET_ARC_CONFIGS.taal,
  fallbacks: [
    MAINNET_ARC_CONFIGS.gorillapool,
    // Add more fallbacks as needed
  ],
  strategy: 'failover',
  healthCheck: { enabled: true, interval: 30000, timeout: 5000 },
  failover: { maxRetries: 3, retryDelay: 1000, backoffMultiplier: 2 }
}

// 2. Enable comprehensive monitoring
const monitoredConfig = new ARCConfigBuilder()
  .url('https://arc.example.com')
  .monitoring({
    enabled: true,
    healthCheck: {
      interval: 30000,
      timeout: 5000,
      retries: 3
    },
    metrics: {
      enabled: true,
      interval: 60000
    },
    alerts: {
      enabled: true,
      thresholds: {
        errorRate: 0.05, // 5% error rate
        responseTime: 10000, // 10 seconds
        availability: 0.99 // 99% availability
      }
    }
  })
  .build()
```

This comprehensive ARC configuration reference provides developers with all the tools and patterns needed to effectively configure and manage ARC connections in their BSV TypeScript SDK applications.
