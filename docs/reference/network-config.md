# Network Configuration Reference

Complete reference for configuring network settings, chain trackers, and blockchain connectivity in the BSV TypeScript SDK.

## Network Configuration Interface

### NetworkConfig

```typescript
interface NetworkConfig {
  name: string
  network: 'mainnet' | 'testnet' | 'regtest'
  chainTracker: ChainTrackerConfig
  feeModel: FeeModelConfig
  limits: NetworkLimits
  endpoints: NetworkEndpoints
  security: NetworkSecurity
  monitoring: NetworkMonitoring
}
```

### ChainTrackerConfig

```typescript
interface ChainTrackerConfig {
  provider: 'whatsonchain' | 'blockchair' | 'custom'
  url?: string
  apiKey?: string
  timeout: number
  retries: number
  retryDelay: number
  fallbacks: ChainTrackerFallback[]
  caching: ChainTrackerCaching
  spv: SPVConfig
}

interface ChainTrackerFallback {
  provider: string
  url: string
  apiKey?: string
  priority: number
  healthCheck: boolean
}

interface ChainTrackerCaching {
  enabled: boolean
  ttl: number // Time to live in milliseconds
  maxSize: number // Maximum cache entries
  storage: 'memory' | 'localStorage' | 'custom'
}

interface SPVConfig {
  enabled: boolean
  maxDepth: number
  requiredConfirmations: number
  merkleProofValidation: boolean
  headerValidation: boolean
}
```

### NetworkEndpoints

```typescript
interface NetworkEndpoints {
  // Block and transaction data
  blocks: string
  transactions: string
  utxos: string
  balance: string
  history: string
  
  // Mempool and broadcasting
  mempool: string
  broadcast: string
  fees: string
  
  // Chain information
  chainInfo: string
  blockHeight: string
  difficulty: string
  
  // WebSocket endpoints
  websocket?: {
    blocks: string
    transactions: string
    mempool: string
  }
}
```

### NetworkLimits

```typescript
interface NetworkLimits {
  maxBlockSize: number
  maxTransactionSize: number
  maxScriptSize: number
  maxInputs: number
  maxOutputs: number
  maxSigOps: number
  dustThreshold: number
  
  // Rate limiting
  requestsPerSecond: number
  requestsPerMinute: number
  requestsPerHour: number
  
  // Connection limits
  maxConcurrentRequests: number
  maxRetries: number
  backoffMultiplier: number
}
```

### NetworkSecurity

```typescript
interface NetworkSecurity {
  ssl: {
    enabled: boolean
    validateCertificates: boolean
    allowSelfSigned: boolean
  }
  
  authentication: {
    required: boolean
    type: 'apiKey' | 'bearer' | 'basic'
    credentials: Record<string, string>
  }
  
  rateLimit: {
    enabled: boolean
    windowMs: number
    maxRequests: number
    skipSuccessfulRequests: boolean
  }
  
  cors: {
    enabled: boolean
    origins: string[]
    methods: string[]
    headers: string[]
  }
}
```

## Predefined Network Configurations

### Mainnet Configuration

```typescript
const MAINNET_CONFIG: NetworkConfig = {
  name: 'Bitcoin SV Mainnet',
  network: 'mainnet',
  chainTracker: {
    provider: 'whatsonchain',
    url: 'https://api.whatsonchain.com/v1/bsv/main',
    timeout: 30000,
    retries: 3,
    retryDelay: 1000,
    fallbacks: [
      {
        provider: 'blockchair',
        url: 'https://api.blockchair.com/bitcoin-sv',
        priority: 1,
        healthCheck: true
      }
    ],
    caching: {
      enabled: true,
      ttl: 300000, // 5 minutes
      maxSize: 1000,
      storage: 'memory'
    },
    spv: {
      enabled: true,
      maxDepth: 6,
      requiredConfirmations: 6,
      merkleProofValidation: true,
      headerValidation: true
    }
  },
  feeModel: {
    standard: 0.5, // satoshis per byte
    data: 0.25,
    priority: 1.0
  },
  limits: {
    maxBlockSize: 4000000000, // 4GB
    maxTransactionSize: 100000000, // 100MB
    maxScriptSize: 10000,
    maxInputs: 100000,
    maxOutputs: 100000,
    maxSigOps: 4000000,
    dustThreshold: 546,
    requestsPerSecond: 10,
    requestsPerMinute: 600,
    requestsPerHour: 36000,
    maxConcurrentRequests: 5,
    maxRetries: 3,
    backoffMultiplier: 2
  },
  endpoints: {
    blocks: '/block',
    transactions: '/tx',
    utxos: '/address/{address}/unspent',
    balance: '/address/{address}/balance',
    history: '/address/{address}/history',
    mempool: '/mempool',
    broadcast: '/tx/raw',
    fees: '/fees',
    chainInfo: '/chain/info',
    blockHeight: '/chain/info',
    difficulty: '/chain/info'
  },
  security: {
    ssl: {
      enabled: true,
      validateCertificates: true,
      allowSelfSigned: false
    },
    authentication: {
      required: false,
      type: 'apiKey',
      credentials: {}
    },
    rateLimit: {
      enabled: true,
      windowMs: 60000,
      maxRequests: 100,
      skipSuccessfulRequests: false
    },
    cors: {
      enabled: false,
      origins: [],
      methods: ['GET', 'POST'],
      headers: ['Content-Type', 'Authorization']
    }
  },
  monitoring: {
    enabled: true,
    healthCheck: {
      interval: 60000,
      timeout: 10000,
      endpoints: ['chainInfo', 'fees']
    },
    metrics: {
      enabled: true,
      collectInterval: 30000,
      retentionPeriod: 86400000 // 24 hours
    }
  }
}
```

### Testnet Configuration

```typescript
const TESTNET_CONFIG: NetworkConfig = {
  name: 'Bitcoin SV Testnet',
  network: 'testnet',
  chainTracker: {
    provider: 'whatsonchain',
    url: 'https://api.whatsonchain.com/v1/bsv/test',
    timeout: 30000,
    retries: 3,
    retryDelay: 1000,
    fallbacks: [],
    caching: {
      enabled: true,
      ttl: 300000,
      maxSize: 500,
      storage: 'memory'
    },
    spv: {
      enabled: true,
      maxDepth: 3, // Shorter for testnet
      requiredConfirmations: 3,
      merkleProofValidation: true,
      headerValidation: true
    }
  },
  feeModel: {
    standard: 0.1, // Lower fees for testnet
    data: 0.05,
    priority: 0.2
  },
  limits: {
    maxBlockSize: 4000000000,
    maxTransactionSize: 100000000,
    maxScriptSize: 10000,
    maxInputs: 100000,
    maxOutputs: 100000,
    maxSigOps: 4000000,
    dustThreshold: 546,
    requestsPerSecond: 20, // Higher limits for testing
    requestsPerMinute: 1200,
    requestsPerHour: 72000,
    maxConcurrentRequests: 10,
    maxRetries: 5,
    backoffMultiplier: 1.5
  },
  endpoints: {
    blocks: '/block',
    transactions: '/tx',
    utxos: '/address/{address}/unspent',
    balance: '/address/{address}/balance',
    history: '/address/{address}/history',
    mempool: '/mempool',
    broadcast: '/tx/raw',
    fees: '/fees',
    chainInfo: '/chain/info',
    blockHeight: '/chain/info',
    difficulty: '/chain/info'
  },
  security: {
    ssl: {
      enabled: true,
      validateCertificates: true,
      allowSelfSigned: false
    },
    authentication: {
      required: false,
      type: 'apiKey',
      credentials: {}
    },
    rateLimit: {
      enabled: false, // Disabled for testing
      windowMs: 60000,
      maxRequests: 1000,
      skipSuccessfulRequests: true
    },
    cors: {
      enabled: true, // Enabled for development
      origins: ['*'],
      methods: ['GET', 'POST', 'OPTIONS'],
      headers: ['*']
    }
  },
  monitoring: {
    enabled: true,
    healthCheck: {
      interval: 30000,
      timeout: 5000,
      endpoints: ['chainInfo']
    },
    metrics: {
      enabled: true,
      collectInterval: 60000,
      retentionPeriod: 43200000 // 12 hours
    }
  }
}
```

### RegTest Configuration

```typescript
const REGTEST_CONFIG: NetworkConfig = {
  name: 'Bitcoin SV RegTest',
  network: 'regtest',
  chainTracker: {
    provider: 'custom',
    url: 'http://localhost:18332',
    timeout: 5000,
    retries: 1,
    retryDelay: 500,
    fallbacks: [],
    caching: {
      enabled: false, // Disabled for local testing
      ttl: 0,
      maxSize: 0,
      storage: 'memory'
    },
    spv: {
      enabled: false, // Disabled for regtest
      maxDepth: 1,
      requiredConfirmations: 1,
      merkleProofValidation: false,
      headerValidation: false
    }
  },
  feeModel: {
    standard: 0.01, // Minimal fees for regtest
    data: 0.01,
    priority: 0.01
  },
  limits: {
    maxBlockSize: 4000000000,
    maxTransactionSize: 100000000,
    maxScriptSize: 10000,
    maxInputs: 100000,
    maxOutputs: 100000,
    maxSigOps: 4000000,
    dustThreshold: 546,
    requestsPerSecond: 1000, // No limits for local testing
    requestsPerMinute: 60000,
    requestsPerHour: 3600000,
    maxConcurrentRequests: 100,
    maxRetries: 1,
    backoffMultiplier: 1
  },
  endpoints: {
    blocks: '/rest/block',
    transactions: '/rest/tx',
    utxos: '/rest/getutxos',
    balance: '/rest/balance',
    history: '/rest/history',
    mempool: '/rest/mempool',
    broadcast: '/rest/tx',
    fees: '/rest/fees',
    chainInfo: '/rest/chaininfo',
    blockHeight: '/rest/chaininfo',
    difficulty: '/rest/chaininfo'
  },
  security: {
    ssl: {
      enabled: false, // HTTP for local development
      validateCertificates: false,
      allowSelfSigned: true
    },
    authentication: {
      required: true,
      type: 'basic',
      credentials: {
        username: 'rpcuser',
        password: 'rpcpassword'
      }
    },
    rateLimit: {
      enabled: false,
      windowMs: 0,
      maxRequests: 0,
      skipSuccessfulRequests: true
    },
    cors: {
      enabled: true,
      origins: ['*'],
      methods: ['*'],
      headers: ['*']
    }
  },
  monitoring: {
    enabled: false, // Disabled for local testing
    healthCheck: {
      interval: 0,
      timeout: 0,
      endpoints: []
    },
    metrics: {
      enabled: false,
      collectInterval: 0,
      retentionPeriod: 0
    }
  }
}
```

## Network Configuration Builder

### NetworkConfigBuilder Class

```typescript
class NetworkConfigBuilder {
  private config: Partial<NetworkConfig> = {}
  
  name(name: string): NetworkConfigBuilder {
    this.config.name = name
    return this
  }
  
  network(network: 'mainnet' | 'testnet' | 'regtest'): NetworkConfigBuilder {
    this.config.network = network
    return this
  }
  
  chainTracker(config: Partial<ChainTrackerConfig>): NetworkConfigBuilder {
    this.config.chainTracker = { ...this.config.chainTracker, ...config }
    return this
  }
  
  feeModel(config: FeeModelConfig): NetworkConfigBuilder {
    this.config.feeModel = config
    return this
  }
  
  limits(config: Partial<NetworkLimits>): NetworkConfigBuilder {
    this.config.limits = { ...this.config.limits, ...config }
    return this
  }
  
  endpoints(config: Partial<NetworkEndpoints>): NetworkConfigBuilder {
    this.config.endpoints = { ...this.config.endpoints, ...config }
    return this
  }
  
  security(config: Partial<NetworkSecurity>): NetworkConfigBuilder {
    this.config.security = { ...this.config.security, ...config }
    return this
  }
  
  monitoring(config: Partial<NetworkMonitoring>): NetworkConfigBuilder {
    this.config.monitoring = { ...this.config.monitoring, ...config }
    return this
  }
  
  build(): NetworkConfig {
    // Apply defaults based on network type
    const defaults = this.getNetworkDefaults(this.config.network || 'mainnet')
    
    return {
      ...defaults,
      ...this.config,
      chainTracker: { ...defaults.chainTracker, ...this.config.chainTracker },
      limits: { ...defaults.limits, ...this.config.limits },
      endpoints: { ...defaults.endpoints, ...this.config.endpoints },
      security: { ...defaults.security, ...this.config.security },
      monitoring: { ...defaults.monitoring, ...this.config.monitoring }
    } as NetworkConfig
  }
  
  private getNetworkDefaults(network: string): NetworkConfig {
    switch (network) {
      case 'mainnet':
        return MAINNET_CONFIG
      case 'testnet':
        return TESTNET_CONFIG
      case 'regtest':
        return REGTEST_CONFIG
      default:
        return MAINNET_CONFIG
    }
  }
}
```

### Usage Examples

```typescript
// Basic network configuration
const basicNetwork = new NetworkConfigBuilder()
  .name('Custom Mainnet')
  .network('mainnet')
  .chainTracker({
    provider: 'whatsonchain',
    timeout: 15000
  })
  .build()

// Custom network with fallbacks
const robustNetwork = new NetworkConfigBuilder()
  .name('Production Mainnet')
  .network('mainnet')
  .chainTracker({
    provider: 'whatsonchain',
    url: 'https://api.whatsonchain.com/v1/bsv/main',
    timeout: 30000,
    retries: 3,
    fallbacks: [
      {
        provider: 'blockchair',
        url: 'https://api.blockchair.com/bitcoin-sv',
        priority: 1,
        healthCheck: true
      },
      {
        provider: 'custom',
        url: 'https://custom-api.example.com',
        apiKey: 'your-api-key',
        priority: 2,
        healthCheck: true
      }
    ]
  })
  .security({
    ssl: { enabled: true, validateCertificates: true },
    rateLimit: { enabled: true, windowMs: 60000, maxRequests: 100 }
  })
  .monitoring({
    enabled: true,
    healthCheck: { interval: 30000, timeout: 5000 }
  })
  .build()
```

## Chain Tracker Management

### ChainTracker Interface

```typescript
interface ChainTracker {
  config: ChainTrackerConfig
  
  // Block operations
  getBlock(hash: string): Promise<Block>
  getBlockHeader(hash: string): Promise<BlockHeader>
  getBlockHeight(): Promise<number>
  
  // Transaction operations
  getTransaction(txid: string): Promise<Transaction>
  getTransactionStatus(txid: string): Promise<TransactionStatus>
  getRawTransaction(txid: string): Promise<string>
  
  // UTXO operations
  getUTXOs(address: string): Promise<UTXO[]>
  getBalance(address: string): Promise<number>
  getHistory(address: string): Promise<TransactionHistory[]>
  
  // Broadcasting
  broadcast(rawTx: string): Promise<BroadcastResult>
  
  // Mempool
  getMempoolInfo(): Promise<MempoolInfo>
  getMempoolTransactions(): Promise<string[]>
  
  // Health and status
  getHealth(): Promise<HealthStatus>
  isConnected(): boolean
}
```

### Multi-Provider Chain Tracker

```typescript
class MultiProviderChainTracker implements ChainTracker {
  private providers: ChainTracker[]
  private currentProvider = 0
  private healthStatus = new Map<string, boolean>()
  
  constructor(
    public config: ChainTrackerConfig,
    providers: ChainTracker[]
  ) {
    this.providers = providers
    this.startHealthChecks()
  }
  
  async getTransaction(txid: string): Promise<Transaction> {
    const provider = this.selectProvider()
    
    try {
      return await provider.getTransaction(txid)
    } catch (error) {
      this.markProviderUnhealthy(provider)
      
      if (this.hasHealthyProviders()) {
        return this.getTransaction(txid) // Retry with different provider
      }
      
      throw error
    }
  }
  
  async broadcast(rawTx: string): Promise<BroadcastResult> {
    const errors: Error[] = []
    
    // Try all healthy providers for broadcasting
    for (const provider of this.getHealthyProviders()) {
      try {
        return await provider.broadcast(rawTx)
      } catch (error) {
        errors.push(error as Error)
        this.markProviderUnhealthy(provider)
      }
    }
    
    throw new Error(`Broadcast failed on all providers: ${errors.map(e => e.message).join(', ')}`)
  }
  
  private selectProvider(): ChainTracker {
    const healthyProviders = this.getHealthyProviders()
    
    if (healthyProviders.length === 0) {
      return this.providers[0] // Fallback to first provider
    }
    
    // Round-robin selection
    const provider = healthyProviders[this.currentProvider % healthyProviders.length]
    this.currentProvider++
    return provider
  }
  
  private getHealthyProviders(): ChainTracker[] {
    return this.providers.filter(provider => 
      this.healthStatus.get(this.getProviderId(provider)) !== false
    )
  }
  
  private markProviderUnhealthy(provider: ChainTracker): void {
    this.healthStatus.set(this.getProviderId(provider), false)
  }
  
  private getProviderId(provider: ChainTracker): string {
    return provider.config.url || provider.config.provider
  }
  
  private startHealthChecks(): void {
    setInterval(async () => {
      for (const provider of this.providers) {
        try {
          await provider.getHealth()
          this.healthStatus.set(this.getProviderId(provider), true)
        } catch (error) {
          this.healthStatus.set(this.getProviderId(provider), false)
        }
      }
    }, 60000) // Check every minute
  }
  
  // Implement other ChainTracker methods...
  async getBlock(hash: string): Promise<Block> {
    const provider = this.selectProvider()
    return provider.getBlock(hash)
  }
  
  async getUTXOs(address: string): Promise<UTXO[]> {
    const provider = this.selectProvider()
    return provider.getUTXOs(address)
  }
  
  async getHealth(): Promise<HealthStatus> {
    const healthyCount = this.getHealthyProviders().length
    const totalCount = this.providers.length
    
    return {
      status: healthyCount > 0 ? 'healthy' : 'unhealthy',
      providers: {
        healthy: healthyCount,
        total: totalCount,
        percentage: (healthyCount / totalCount) * 100
      },
      timestamp: Date.now()
    }
  }
  
  isConnected(): boolean {
    return this.getHealthyProviders().length > 0
  }
}
```

## Environment-Based Network Configuration

### Environment Configuration Loader

```typescript
class NetworkEnvironmentConfig {
  static fromEnvironment(): NetworkConfig {
    const network = (process.env.BSV_NETWORK || 'mainnet') as 'mainnet' | 'testnet' | 'regtest'
    const baseConfig = NetworkEnvironmentConfig.getBaseConfig(network)
    
    // Override with environment variables
    if (process.env.BSV_CHAIN_TRACKER_URL) {
      baseConfig.chainTracker.url = process.env.BSV_CHAIN_TRACKER_URL
    }
    
    if (process.env.BSV_CHAIN_TRACKER_API_KEY) {
      baseConfig.chainTracker.apiKey = process.env.BSV_CHAIN_TRACKER_API_KEY
    }
    
    if (process.env.BSV_CHAIN_TRACKER_TIMEOUT) {
      baseConfig.chainTracker.timeout = parseInt(process.env.BSV_CHAIN_TRACKER_TIMEOUT)
    }
    
    // Fee model overrides
    if (process.env.BSV_FEE_STANDARD) {
      baseConfig.feeModel.standard = parseFloat(process.env.BSV_FEE_STANDARD)
    }
    
    if (process.env.BSV_FEE_DATA) {
      baseConfig.feeModel.data = parseFloat(process.env.BSV_FEE_DATA)
    }
    
    // Security overrides
    if (process.env.BSV_SSL_ENABLED) {
      baseConfig.security.ssl.enabled = process.env.BSV_SSL_ENABLED === 'true'
    }
    
    if (process.env.BSV_RATE_LIMIT_ENABLED) {
      baseConfig.security.rateLimit.enabled = process.env.BSV_RATE_LIMIT_ENABLED === 'true'
    }
    
    return baseConfig
  }
  
  private static getBaseConfig(network: string): NetworkConfig {
    switch (network) {
      case 'mainnet':
        return { ...MAINNET_CONFIG }
      case 'testnet':
        return { ...TESTNET_CONFIG }
      case 'regtest':
        return { ...REGTEST_CONFIG }
      default:
        return { ...MAINNET_CONFIG }
    }
  }
  
  static validate(config: NetworkConfig): void {
    if (!config.name) {
      throw new Error('Network name is required')
    }
    
    if (!config.network) {
      throw new Error('Network type is required')
    }
    
    if (!config.chainTracker.url && config.chainTracker.provider === 'custom') {
      throw new Error('Chain tracker URL is required for custom provider')
    }
    
    if (config.chainTracker.timeout < 1000) {
      throw new Error('Chain tracker timeout must be at least 1000ms')
    }
    
    if (config.limits.maxTransactionSize > config.limits.maxBlockSize) {
      throw new Error('Max transaction size cannot exceed max block size')
    }
    
    if (config.security.ssl.enabled && !config.chainTracker.url?.startsWith('https')) {
      console.warn('SSL is enabled but chain tracker URL is not HTTPS')
    }
  }
}
```

## Network Configuration Best Practices

### Production Configuration

```typescript
const productionConfig = new NetworkConfigBuilder()
  .name('Production Mainnet')
  .network('mainnet')
  .chainTracker({
    provider: 'whatsonchain',
    timeout: 30000,
    retries: 3,
    retryDelay: 2000,
    fallbacks: [
      {
        provider: 'blockchair',
        url: 'https://api.blockchair.com/bitcoin-sv',
        priority: 1,
        healthCheck: true
      }
    ],
    caching: {
      enabled: true,
      ttl: 300000, // 5 minutes
      maxSize: 10000,
      storage: 'memory'
    }
  })
  .security({
    ssl: {
      enabled: true,
      validateCertificates: true,
      allowSelfSigned: false
    },
    rateLimit: {
      enabled: true,
      windowMs: 60000,
      maxRequests: 100
    }
  })
  .monitoring({
    enabled: true,
    healthCheck: {
      interval: 30000,
      timeout: 10000,
      endpoints: ['chainInfo', 'fees']
    },
    metrics: {
      enabled: true,
      collectInterval: 60000,
      retentionPeriod: 86400000
    }
  })
  .build()
```

### Development Configuration

```typescript
const developmentConfig = new NetworkConfigBuilder()
  .name('Development Testnet')
  .network('testnet')
  .chainTracker({
    provider: 'whatsonchain',
    timeout: 15000,
    retries: 5,
    caching: {
      enabled: false // Disable caching for development
    }
  })
  .security({
    ssl: { enabled: true },
    rateLimit: { enabled: false }, // Disable rate limiting for development
    cors: {
      enabled: true,
      origins: ['*'],
      methods: ['*'],
      headers: ['*']
    }
  })
  .monitoring({
    enabled: true,
    healthCheck: { interval: 10000, timeout: 5000 }
  })
  .build()
```

This comprehensive network configuration reference provides developers with all the tools needed to effectively configure and manage network connectivity in their BSV TypeScript SDK applications.
