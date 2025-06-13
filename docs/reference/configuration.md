# Configuration Reference

Complete reference for SDK configuration options, interfaces, and setup patterns in the BSV TypeScript SDK.

## Core Configuration Interface

```typescript
interface SDKConfig {
  network: NetworkType
  arc: ARCConfig
  fees: FeeConfig
  security: SecurityConfig
  wallet: WalletConfig
  chainTracker: ChainTrackerConfig
  logging: LoggingConfig
}
```

## Network Configuration

### NetworkType

```typescript
type NetworkType = 'mainnet' | 'testnet' | 'regtest'
```

### Network Parameters

```typescript
interface NetworkConfig {
  name: NetworkType
  chainParams: {
    genesisHash: string
    port: number
    dnsSeeds: string[]
    addressPrefix: {
      pubKeyHash: number
      scriptHash: number
      privateKey: number
    }
  }
  defaultEndpoints: {
    arc: string[]
    chainTracker: string[]
    broadcast: string[]
  }
}
```

### Predefined Networks

#### Mainnet Configuration
```typescript
const MAINNET_CONFIG: NetworkConfig = {
  name: 'mainnet',
  chainParams: {
    genesisHash: '000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f',
    port: 8333,
    dnsSeeds: [
      'seed.bitcoinsv.io',
      'seed.cascharia.com',
      'seed.satoshisvision.network'
    ],
    addressPrefix: {
      pubKeyHash: 0x00,
      scriptHash: 0x05,
      privateKey: 0x80
    }
  },
  defaultEndpoints: {
    arc: ['https://arc.taal.com'],
    chainTracker: ['https://api.whatsonchain.com/v1/bsv/main'],
    broadcast: ['https://api.whatsonchain.com/v1/bsv/main/tx/raw']
  }
}
```

#### Testnet Configuration
```typescript
const TESTNET_CONFIG: NetworkConfig = {
  name: 'testnet',
  chainParams: {
    genesisHash: '000000000933ea01ad0ee984209779baaec3ced90fa3f408719526f8d77f4943',
    port: 18333,
    dnsSeeds: [
      'testnet-seed.bitcoinsv.io',
      'testnet-seed.cascharia.com'
    ],
    addressPrefix: {
      pubKeyHash: 0x6f,
      scriptHash: 0xc4,
      privateKey: 0xef
    }
  },
  defaultEndpoints: {
    arc: ['https://arc-testnet.taal.com'],
    chainTracker: ['https://api.whatsonchain.com/v1/bsv/test'],
    broadcast: ['https://api.whatsonchain.com/v1/bsv/test/tx/raw']
  }
}
```

#### Regtest Configuration
```typescript
const REGTEST_CONFIG: NetworkConfig = {
  name: 'regtest',
  chainParams: {
    genesisHash: '0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206',
    port: 18444,
    dnsSeeds: [],
    addressPrefix: {
      pubKeyHash: 0x6f,
      scriptHash: 0xc4,
      privateKey: 0xef
    }
  },
  defaultEndpoints: {
    arc: ['http://localhost:9090'],
    chainTracker: ['http://localhost:3001/v1/bsv/regtest'],
    broadcast: ['http://localhost:3001/v1/bsv/regtest/tx/raw']
  }
}
```

## ARC Configuration

### ARCConfig Interface

```typescript
interface ARCConfig {
  apiUrl: string
  apiKey?: string
  timeout: number
  retryAttempts: number
  retryDelay: number
  rateLimiting: {
    requestsPerSecond: number
    burstLimit: number
  }
  endpoints: {
    submit: string
    status: string
    policy: string
  }
}
```

### Default ARC Configuration

```typescript
const DEFAULT_ARC_CONFIG: ARCConfig = {
  apiUrl: 'https://arc.taal.com',
  timeout: 30000, // 30 seconds
  retryAttempts: 3,
  retryDelay: 1000, // 1 second
  rateLimiting: {
    requestsPerSecond: 10,
    burstLimit: 50
  },
  endpoints: {
    submit: '/v1/tx',
    status: '/v1/tx/{txid}',
    policy: '/v1/policy'
  }
}
```

### ARC Authentication

```typescript
interface ARCAuth {
  type: 'bearer' | 'api-key' | 'none'
  credentials: {
    token?: string
    apiKey?: string
    secret?: string
  }
}
```

## Fee Configuration

### FeeConfig Interface

```typescript
interface FeeConfig {
  strategy: FeeStrategy
  rates: FeeRates
  limits: FeeLimits
  estimation: FeeEstimationConfig
}
```

### Fee Strategy Types

```typescript
type FeeStrategy = 'fixed' | 'dynamic' | 'priority' | 'custom'

interface FeeRates {
  // Satoshis per byte
  standard: number
  priority: number
  economy: number
  custom?: number
}

interface FeeLimits {
  minFeeRate: number // Minimum satoshis per byte
  maxFeeRate: number // Maximum satoshis per byte
  maxTotalFee: number // Maximum total fee in satoshis
}
```

### Default Fee Configuration

```typescript
const DEFAULT_FEE_CONFIG: FeeConfig = {
  strategy: 'standard',
  rates: {
    standard: 0.5,  // 0.5 sat/byte
    priority: 1.0,  // 1.0 sat/byte
    economy: 0.25   // 0.25 sat/byte
  },
  limits: {
    minFeeRate: 0.25,
    maxFeeRate: 10.0,
    maxTotalFee: 100000 // 1000 sat maximum
  },
  estimation: {
    enabled: true,
    source: 'arc',
    fallbackRate: 0.5,
    updateInterval: 300000 // 5 minutes
  }
}
```

### Fee Estimation Configuration

```typescript
interface FeeEstimationConfig {
  enabled: boolean
  source: 'arc' | 'chainTracker' | 'static'
  fallbackRate: number
  updateInterval: number
  cacheTimeout: number
}
```

## Security Configuration

### SecurityConfig Interface

```typescript
interface SecurityConfig {
  keyGeneration: KeyGenerationConfig
  encryption: EncryptionConfig
  validation: ValidationConfig
  randomness: RandomnessConfig
}
```

### Key Generation Configuration

```typescript
interface KeyGenerationConfig {
  source: 'secure-random' | 'deterministic'
  entropy: {
    minBits: number
    sources: EntropySource[]
  }
  derivation: {
    hardened: boolean
    maxDepth: number
  }
}

type EntropySource = 'crypto' | 'mouse' | 'keyboard' | 'timing'
```

### Encryption Configuration

```typescript
interface EncryptionConfig {
  algorithm: 'AES-GCM' | 'AES-CBC'
  keySize: 128 | 256
  ivSize: number
  tagSize: number
  keyDerivation: {
    algorithm: 'PBKDF2' | 'scrypt'
    iterations: number
    saltSize: number
  }
}
```

### Default Security Configuration

```typescript
const DEFAULT_SECURITY_CONFIG: SecurityConfig = {
  keyGeneration: {
    source: 'secure-random',
    entropy: {
      minBits: 256,
      sources: ['crypto']
    },
    derivation: {
      hardened: true,
      maxDepth: 5
    }
  },
  encryption: {
    algorithm: 'AES-GCM',
    keySize: 256,
    ivSize: 12,
    tagSize: 16,
    keyDerivation: {
      algorithm: 'PBKDF2',
      iterations: 100000,
      saltSize: 32
    }
  },
  validation: {
    strictMode: true,
    checkSignatures: true,
    validateScripts: true,
    enforceMinimumFees: true
  },
  randomness: {
    source: 'crypto.getRandomValues',
    testRandomness: false,
    fallbackToMath: false
  }
}
```

## Wallet Configuration

### WalletConfig Interface

```typescript
interface WalletConfig {
  substrate: WalletSubstrate
  connection: ConnectionConfig
  authentication: AuthConfig
  permissions: PermissionConfig
}
```

### Wallet Substrate Types

```typescript
type WalletSubstrate = 
  | 'auto'
  | 'Cicada'
  | 'XDM'
  | 'window.CWI'
  | 'json-api'
  | 'react-native'
  | WalletInterface
```

### Connection Configuration

```typescript
interface ConnectionConfig {
  timeout: number
  retryAttempts: number
  retryDelay: number
  keepAlive: boolean
  autoReconnect: boolean
}
```

### Authentication Configuration

```typescript
interface AuthConfig {
  required: boolean
  timeout: number
  cacheCredentials: boolean
  refreshInterval: number
  originator: string
}
```

### Default Wallet Configuration

```typescript
const DEFAULT_WALLET_CONFIG: WalletConfig = {
  substrate: 'auto',
  connection: {
    timeout: 10000,
    retryAttempts: 3,
    retryDelay: 1000,
    keepAlive: true,
    autoReconnect: true
  },
  authentication: {
    required: true,
    timeout: 30000,
    cacheCredentials: false,
    refreshInterval: 3600000, // 1 hour
    originator: 'localhost'
  },
  permissions: {
    createActions: true,
    signTransactions: true,
    accessKeys: false,
    manageOutputs: true
  }
}
```

## Chain Tracker Configuration

### ChainTrackerConfig Interface

```typescript
interface ChainTrackerConfig {
  primary: ChainTrackerEndpoint
  fallbacks: ChainTrackerEndpoint[]
  failover: FailoverConfig
  caching: CacheConfig
}
```

### Chain Tracker Endpoint

```typescript
interface ChainTrackerEndpoint {
  url: string
  apiKey?: string
  timeout: number
  rateLimiting: RateLimitConfig
  capabilities: TrackerCapability[]
}

type TrackerCapability = 
  | 'getHeight'
  | 'getHeader'
  | 'getTransaction'
  | 'getMerkleProof'
  | 'broadcast'
```

### Failover Configuration

```typescript
interface FailoverConfig {
  enabled: boolean
  maxFailures: number
  failureWindow: number
  recoveryTime: number
  healthCheck: {
    enabled: boolean
    interval: number
    timeout: number
  }
}
```

### Default Chain Tracker Configuration

```typescript
const DEFAULT_CHAINTRACKER_CONFIG: ChainTrackerConfig = {
  primary: {
    url: 'https://api.whatsonchain.com/v1/bsv/main',
    timeout: 10000,
    rateLimiting: {
      requestsPerSecond: 5,
      burstLimit: 20
    },
    capabilities: ['getHeight', 'getHeader', 'getTransaction', 'getMerkleProof']
  },
  fallbacks: [
    {
      url: 'https://api.bitindex.network',
      timeout: 15000,
      rateLimiting: {
        requestsPerSecond: 3,
        burstLimit: 10
      },
      capabilities: ['getHeight', 'getTransaction']
    }
  ],
  failover: {
    enabled: true,
    maxFailures: 3,
    failureWindow: 300000, // 5 minutes
    recoveryTime: 600000,  // 10 minutes
    healthCheck: {
      enabled: true,
      interval: 60000,     // 1 minute
      timeout: 5000
    }
  },
  caching: {
    enabled: true,
    ttl: 30000,           // 30 seconds
    maxSize: 1000,
    strategy: 'lru'
  }
}
```

## Logging Configuration

### LoggingConfig Interface

```typescript
interface LoggingConfig {
  level: LogLevel
  outputs: LogOutput[]
  format: LogFormat
  filters: LogFilter[]
  performance: PerformanceLoggingConfig
}
```

### Log Levels and Outputs

```typescript
type LogLevel = 'error' | 'warn' | 'info' | 'debug' | 'trace'

interface LogOutput {
  type: 'console' | 'file' | 'remote'
  config: {
    file?: string
    url?: string
    maxSize?: number
    rotation?: boolean
  }
}

interface LogFormat {
  timestamp: boolean
  level: boolean
  component: boolean
  structured: boolean
  colors: boolean
}
```

### Default Logging Configuration

```typescript
const DEFAULT_LOGGING_CONFIG: LoggingConfig = {
  level: 'info',
  outputs: [
    {
      type: 'console',
      config: {}
    }
  ],
  format: {
    timestamp: true,
    level: true,
    component: true,
    structured: false,
    colors: true
  },
  filters: [],
  performance: {
    enabled: false,
    threshold: 1000,
    includeStackTrace: false
  }
}
```

## Configuration Loading and Management

### Configuration Builder

```typescript
class SDKConfigBuilder {
  private config: Partial<SDKConfig> = {}
  
  network(type: NetworkType): this {
    this.config.network = type
    return this
  }
  
  arc(config: Partial<ARCConfig>): this {
    this.config.arc = { ...DEFAULT_ARC_CONFIG, ...config }
    return this
  }
  
  fees(config: Partial<FeeConfig>): this {
    this.config.fees = { ...DEFAULT_FEE_CONFIG, ...config }
    return this
  }
  
  security(config: Partial<SecurityConfig>): this {
    this.config.security = { ...DEFAULT_SECURITY_CONFIG, ...config }
    return this
  }
  
  wallet(config: Partial<WalletConfig>): this {
    this.config.wallet = { ...DEFAULT_WALLET_CONFIG, ...config }
    return this
  }
  
  chainTracker(config: Partial<ChainTrackerConfig>): this {
    this.config.chainTracker = { ...DEFAULT_CHAINTRACKER_CONFIG, ...config }
    return this
  }
  
  logging(config: Partial<LoggingConfig>): this {
    this.config.logging = { ...DEFAULT_LOGGING_CONFIG, ...config }
    return this
  }
  
  build(): SDKConfig {
    return {
      network: this.config.network || 'testnet',
      arc: this.config.arc || DEFAULT_ARC_CONFIG,
      fees: this.config.fees || DEFAULT_FEE_CONFIG,
      security: this.config.security || DEFAULT_SECURITY_CONFIG,
      wallet: this.config.wallet || DEFAULT_WALLET_CONFIG,
      chainTracker: this.config.chainTracker || DEFAULT_CHAINTRACKER_CONFIG,
      logging: this.config.logging || DEFAULT_LOGGING_CONFIG
    }
  }
}
```

### Configuration Usage Examples

#### Basic Configuration

```typescript
import { SDKConfigBuilder } from '@bsv/sdk'

const config = new SDKConfigBuilder()
  .network('testnet')
  .fees({ strategy: 'priority' })
  .build()
```

#### Production Configuration

```typescript
const productionConfig = new SDKConfigBuilder()
  .network('mainnet')
  .arc({
    apiUrl: 'https://arc.taal.com',
    apiKey: process.env.TAAL_API_KEY,
    timeout: 30000
  })
  .fees({
    strategy: 'dynamic',
    rates: {
      standard: 0.5,
      priority: 1.0,
      economy: 0.25
    }
  })
  .security({
    validation: {
      strictMode: true,
      checkSignatures: true,
      validateScripts: true
    }
  })
  .chainTracker({
    primary: {
      url: 'https://api.whatsonchain.com/v1/bsv/main',
      timeout: 10000
    },
    failover: {
      enabled: true,
      maxFailures: 3
    }
  })
  .logging({
    level: 'warn',
    outputs: [
      { type: 'console', config: {} },
      { type: 'file', config: { file: '/var/log/bsv-sdk.log' } }
    ]
  })
  .build()
```

#### Development Configuration

```typescript
const devConfig = new SDKConfigBuilder()
  .network('regtest')
  .arc({
    apiUrl: 'http://localhost:9090',
    timeout: 5000
  })
  .fees({
    strategy: 'fixed',
    rates: { standard: 1.0 }
  })
  .chainTracker({
    primary: {
      url: 'http://localhost:3001/v1/bsv/regtest',
      timeout: 5000
    },
    failover: { enabled: false }
  })
  .logging({
    level: 'debug',
    format: {
      timestamp: true,
      colors: true,
      structured: true
    }
  })
  .build()
```

## Environment-Based Configuration

### Configuration from Environment Variables

```typescript
function loadConfigFromEnv(): SDKConfig {
  return new SDKConfigBuilder()
    .network((process.env.BSV_NETWORK as NetworkType) || 'testnet')
    .arc({
      apiUrl: process.env.ARC_API_URL || 'https://arc-testnet.taal.com',
      apiKey: process.env.ARC_API_KEY,
      timeout: parseInt(process.env.ARC_TIMEOUT || '30000')
    })
    .fees({
      strategy: (process.env.FEE_STRATEGY as FeeStrategy) || 'standard',
      rates: {
        standard: parseFloat(process.env.FEE_RATE_STANDARD || '0.5'),
        priority: parseFloat(process.env.FEE_RATE_PRIORITY || '1.0'),
        economy: parseFloat(process.env.FEE_RATE_ECONOMY || '0.25')
      }
    })
    .wallet({
      substrate: (process.env.WALLET_SUBSTRATE as WalletSubstrate) || 'auto',
      authentication: {
        originator: process.env.WALLET_ORIGINATOR || 'localhost'
      }
    })
    .logging({
      level: (process.env.LOG_LEVEL as LogLevel) || 'info'
    })
    .build()
}
```

### Configuration Validation

```typescript
function validateConfig(config: SDKConfig): string[] {
  const errors: string[] = []
  
  // Validate network
  if (!['mainnet', 'testnet', 'regtest'].includes(config.network)) {
    errors.push('Invalid network type')
  }
  
  // Validate ARC configuration
  if (!config.arc.apiUrl) {
    errors.push('ARC API URL is required')
  }
  
  if (config.arc.timeout < 1000) {
    errors.push('ARC timeout must be at least 1000ms')
  }
  
  // Validate fee configuration
  if (config.fees.rates.standard <= 0) {
    errors.push('Standard fee rate must be positive')
  }
  
  if (config.fees.limits.minFeeRate > config.fees.limits.maxFeeRate) {
    errors.push('Minimum fee rate cannot exceed maximum fee rate')
  }
  
  // Validate security configuration
  if (config.security.encryption.keySize !== 128 && config.security.encryption.keySize !== 256) {
    errors.push('Encryption key size must be 128 or 256 bits')
  }
  
  return errors
}
```

## Configuration Best Practices

### Security Considerations

1. **Never hardcode API keys** - Use environment variables or secure configuration management
2. **Use HTTPS endpoints** - Ensure all external API calls use secure connections
3. **Validate configuration** - Always validate configuration before using
4. **Rotate credentials** - Regularly rotate API keys and authentication tokens
5. **Limit permissions** - Use principle of least privilege for wallet permissions

### Performance Optimization

1. **Configure appropriate timeouts** - Balance responsiveness with reliability
2. **Use connection pooling** - Reuse connections where possible
3. **Enable caching** - Cache frequently accessed data with appropriate TTLs
4. **Configure failover** - Use multiple endpoints for high availability
5. **Monitor performance** - Enable performance logging in production

### Environment-Specific Settings

#### Development
- Use testnet or regtest networks
- Enable debug logging
- Shorter timeouts for faster feedback
- Disable strict validation for testing

#### Staging
- Mirror production configuration
- Enable comprehensive logging
- Use production-like endpoints
- Enable all validation checks

#### Production
- Use mainnet network
- Minimal logging (warn/error only)
- Longer timeouts for reliability
- Enable all security features
- Use redundant endpoints

This comprehensive configuration reference provides developers with all the tools needed to properly configure the BSV TypeScript SDK for any environment or use case.
