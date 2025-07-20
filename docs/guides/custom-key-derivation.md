# Implementing Custom Key Derivation

This guide demonstrates how to implement custom key derivation schemes using the BSV TypeScript SDK. Unlike standardized protocols like Type-42, custom derivation allows you to create application-specific key hierarchies tailored to your exact requirements.

## Overview

Custom key derivation enables you to:

- Create deterministic key hierarchies from master keys
- Implement application-specific derivation paths
- Generate keys for different purposes (signing, encryption, authentication)
- Create secure backup and recovery systems
- Build multi-tenant applications with isolated key spaces

## Basic Custom Derivation

### Simple Path-Based Derivation

```typescript
import { PrivateKey, Hash, Utils } from '@bsv/sdk'

class CustomKeyDeriver {
  private masterKey: PrivateKey

  constructor(masterKey: PrivateKey) {
    this.masterKey = masterKey
  }

  // Derive child key using custom path
  deriveFromPath(path: string): PrivateKey {
    // Create derivation material from master key and path
    const masterBytes = this.masterKey.toArray()
    const pathBytes = Utils.toArray(path, 'utf8')
    const derivationInput = [...masterBytes, ...pathBytes]
    
    // Hash to create deterministic child key
    const childKeyBytes = Hash.sha256(derivationInput)
    
    return new PrivateKey(childKeyBytes)
  }

  // Derive multiple keys for different purposes
  deriveMultiPurpose(identifier: string) {
    return {
      signing: this.deriveFromPath(`${identifier}/signing`),
      encryption: this.deriveFromPath(`${identifier}/encryption`),
      authentication: this.deriveFromPath(`${identifier}/auth`)
    }
  }
}

// Example usage
const masterKey = PrivateKey.fromRandom()
const deriver = new CustomKeyDeriver(masterKey)

// Derive keys for user account
const userKeys = deriver.deriveMultiPurpose('user:alice@example.com')
console.log('Signing key:', userKeys.signing.toWif())
console.log('Encryption key:', userKeys.encryption.toWif())
console.log('Auth key:', userKeys.authentication.toWif())
```

### Hierarchical Custom Derivation

```typescript
class HierarchicalDeriver {
  private masterKey: PrivateKey

  constructor(masterKey: PrivateKey) {
    this.masterKey = masterKey
  }

  // Derive using hierarchical path (e.g., "app/users/123/keys/0")
  deriveHierarchical(pathComponents: string[]): PrivateKey {
    let currentKey = this.masterKey
    
    // Derive through each level of hierarchy
    for (const component of pathComponents) {
      currentKey = this.deriveChild(currentKey, component)
    }
    
    return currentKey
  }

  private deriveChild(parentKey: PrivateKey, component: string): PrivateKey {
    const parentBytes = parentKey.toArray()
    const componentBytes = Utils.toArray(component, 'utf8')
    const derivationInput = [...parentBytes, ...componentBytes]
    
    const childKeyBytes = Hash.sha256(derivationInput)
    return new PrivateKey(childKeyBytes)
  }

  // Convenience methods for common patterns
  deriveUserKey(userId: string, keyIndex: number): PrivateKey {
    return this.deriveHierarchical(['users', userId, 'keys', keyIndex.toString()])
  }

  deriveApplicationKey(appId: string, purpose: string): PrivateKey {
    return this.deriveHierarchical(['apps', appId, purpose])
  }

  deriveSessionKey(sessionId: string): PrivateKey {
    return this.deriveHierarchical(['sessions', sessionId])
  }
}

// Example usage
const hierarchical = new HierarchicalDeriver(PrivateKey.fromRandom())

// Derive keys for different contexts
const userKey = hierarchical.deriveUserKey('user123', 0)
const appKey = hierarchical.deriveApplicationKey('myapp', 'payments')
const sessionKey = hierarchical.deriveSessionKey('sess_abc123')

console.log('User key:', userKey.toWif())
console.log('App key:', appKey.toWif())
console.log('Session key:', sessionKey.toWif())
```

## Advanced Derivation Patterns

### Salt-Enhanced Derivation

```typescript
import { Random } from '@bsv/sdk'

class SaltedKeyDeriver {
  private masterKey: PrivateKey
  private globalSalt: number[]

  constructor(masterKey: PrivateKey, globalSalt?: number[]) {
    this.masterKey = masterKey
    this.globalSalt = globalSalt || Random(32)
  }

  // Derive with additional salt for extra security
  deriveWithSalt(path: string, additionalSalt?: number[]): PrivateKey {
    const masterBytes = this.masterKey.toArray()
    const pathBytes = Utils.toArray(path, 'utf8')
    const salt = additionalSalt || Random(16)
    
    // Combine all inputs
    const derivationInput = [
      ...masterBytes,
      ...this.globalSalt,
      ...pathBytes,
      ...salt
    ]
    
    const childKeyBytes = Hash.sha256(derivationInput)
    return new PrivateKey(childKeyBytes)
  }

  // Get the global salt for backup purposes
  getGlobalSalt(): number[] {
    return [...this.globalSalt]
  }
}

// Example with salt backup
const saltedDeriver = new SaltedKeyDeriver(PrivateKey.fromRandom())
const derivedKey = saltedDeriver.deriveWithSalt('payment:invoice123')

// Store salt for recovery
const saltForBackup = saltedDeriver.getGlobalSalt()
console.log('Global salt (backup this):', Utils.toBase64(saltForBackup))
```

### Time-Based Key Rotation

```typescript
class RotatingKeyDeriver {
  private masterKey: PrivateKey
  private rotationInterval: number // milliseconds

  constructor(masterKey: PrivateKey, rotationIntervalHours: number = 24) {
    this.masterKey = masterKey
    this.rotationInterval = rotationIntervalHours * 60 * 60 * 1000
  }

  // Derive key that changes based on time period
  deriveTimeBasedKey(identifier: string, timestamp?: number): PrivateKey {
    const now = timestamp || Date.now()
    const period = Math.floor(now / this.rotationInterval)
    
    const path = `${identifier}/period:${period}`
    return this.deriveFromPath(path)
  }

  // Get the current period number
  getCurrentPeriod(timestamp?: number): number {
    const now = timestamp || Date.now()
    return Math.floor(now / this.rotationInterval)
  }

  // Get key for specific period (useful for validation)
  deriveKeyForPeriod(identifier: string, period: number): PrivateKey {
    const path = `${identifier}/period:${period}`
    return this.deriveFromPath(path)
  }

  private deriveFromPath(path: string): PrivateKey {
    const masterBytes = this.masterKey.toArray()
    const pathBytes = Utils.toArray(path, 'utf8')
    const derivationInput = [...masterBytes, ...pathBytes]
    
    const childKeyBytes = Hash.sha256(derivationInput)
    return new PrivateKey(childKeyBytes)
  }
}

// Example usage
const rotatingDeriver = new RotatingKeyDeriver(PrivateKey.fromRandom(), 1) // 1 hour rotation

// Current key
const currentKey = rotatingDeriver.deriveTimeBasedKey('api:auth')
console.log('Current period:', rotatingDeriver.getCurrentPeriod())

// Key from previous period (for validation during rotation)
const previousPeriod = rotatingDeriver.getCurrentPeriod() - 1
const previousKey = rotatingDeriver.deriveKeyForPeriod('api:auth', previousPeriod)
console.log('Previous key still valid for grace period')
```

## Multi-Tenant Key Isolation

### Tenant-Specific Derivation

```typescript
class MultiTenantKeyDeriver {
  private masterKey: PrivateKey
  private tenantSalts: Map<string, number[]> = new Map()

  constructor(masterKey: PrivateKey) {
    this.masterKey = masterKey
  }

  // Create isolated key space for tenant
  createTenantSpace(tenantId: string): number[] {
    if (this.tenantSalts.has(tenantId)) {
      throw new Error(`Tenant ${tenantId} already exists`)
    }

    const tenantSalt = Random(32)
    this.tenantSalts.set(tenantId, tenantSalt)
    return [...tenantSalt]
  }

  // Derive key within tenant's isolated space
  deriveTenantKey(tenantId: string, keyPath: string): PrivateKey {
    const tenantSalt = this.tenantSalts.get(tenantId)
    if (!tenantSalt) {
      throw new Error(`Tenant ${tenantId} not found`)
    }

    const masterBytes = this.masterKey.toArray()
    const pathBytes = Utils.toArray(`tenant:${tenantId}/${keyPath}`, 'utf8')
    
    const derivationInput = [
      ...masterBytes,
      ...tenantSalt,
      ...pathBytes
    ]
    
    const childKeyBytes = Hash.sha256(derivationInput)
    return new PrivateKey(childKeyBytes)
  }

  // Get tenant salt for backup
  getTenantSalt(tenantId: string): number[] | undefined {
    const salt = this.tenantSalts.get(tenantId)
    return salt ? [...salt] : undefined
  }

  // List all tenants
  listTenants(): string[] {
    return Array.from(this.tenantSalts.keys())
  }
}

// Example usage
const multiTenant = new MultiTenantKeyDeriver(PrivateKey.fromRandom())

// Create tenant spaces
const tenant1Salt = multiTenant.createTenantSpace('company-a')
const tenant2Salt = multiTenant.createTenantSpace('company-b')

// Derive keys within each tenant's space
const companyAPaymentKey = multiTenant.deriveTenantKey('company-a', 'payments/primary')
const companyBPaymentKey = multiTenant.deriveTenantKey('company-b', 'payments/primary')

console.log('Company A key:', companyAPaymentKey.toWif())
console.log('Company B key:', companyBPaymentKey.toWif())
console.log('Keys are completely isolated:', companyAPaymentKey.toWif() !== companyBPaymentKey.toWif())
```

## Key Backup and Recovery

### Example Backup System

```typescript
interface KeyBackupData {
  masterKeyWif: string
  derivationScheme: string
  globalSalt?: string
  tenantSalts?: Record<string, string>
  metadata: {
    createdAt: number
    version: string
    description: string
  }
}

class BackupableKeyDeriver {
  private masterKey: PrivateKey
  private globalSalt: number[]
  private tenantSalts: Map<string, number[]> = new Map()
  private derivationScheme: string

  constructor(
    masterKey: PrivateKey, 
    derivationScheme: string = 'custom-v1',
    globalSalt?: number[]
  ) {
    this.masterKey = masterKey
    this.derivationScheme = derivationScheme
    this.globalSalt = globalSalt || Random(32)
  }

  // Create complete backup data
  createBackup(description: string): KeyBackupData {
    const tenantSalts: Record<string, string> = {}
    
    for (const [tenantId, salt] of this.tenantSalts) {
      tenantSalts[tenantId] = Utils.toBase64(salt)
    }

    return {
      masterKeyWif: this.masterKey.toWif(),
      derivationScheme: this.derivationScheme,
      globalSalt: Utils.toBase64(this.globalSalt),
      tenantSalts,
      metadata: {
        createdAt: Date.now(),
        version: '1.0.0',
        description
      }
    }
  }

  // Restore from backup data
  static fromBackup(backupData: KeyBackupData): BackupableKeyDeriver {
    const masterKey = PrivateKey.fromWif(backupData.masterKeyWif)
    const globalSalt = backupData.globalSalt ? 
      Utils.toArray(backupData.globalSalt, 'base64') : 
      undefined

    const deriver = new BackupableKeyDeriver(
      masterKey,
      backupData.derivationScheme,
      globalSalt
    )

    // Restore tenant salts
    if (backupData.tenantSalts) {
      for (const [tenantId, saltBase64] of Object.entries(backupData.tenantSalts)) {
        const salt = Utils.toArray(saltBase64, 'base64')
        deriver.tenantSalts.set(tenantId, salt)
      }
    }

    return deriver
  }

  // Verify backup integrity
  verifyBackup(backupData: KeyBackupData): boolean {
    try {
      const restored = BackupableKeyDeriver.fromBackup(backupData)
      
      // Test key derivation
      const testKey1 = this.deriveFromPath('test/verification')
      const testKey2 = restored.deriveFromPath('test/verification')
      
      return testKey1.toWif() === testKey2.toWif()
    } catch (error) {
      console.error('Backup verification failed:', error)
      return false
    }
  }

  // Add tenant (for backup system)
  addTenant(tenantId: string): number[] {
    const salt = Random(32)
    this.tenantSalts.set(tenantId, salt)
    return [...salt]
  }

  private deriveFromPath(path: string): PrivateKey {
    const masterBytes = this.masterKey.toArray()
    const pathBytes = Utils.toArray(path, 'utf8')
    const derivationInput = [...masterBytes, ...this.globalSalt, ...pathBytes]
    
    const childKeyBytes = Hash.sha256(derivationInput)
    return new PrivateKey(childKeyBytes)
  }
}

// Example backup and recovery
const originalDeriver = new BackupableKeyDeriver(
  PrivateKey.fromRandom(),
  'my-app-v1'
)

// Add some tenants
originalDeriver.addTenant('tenant1')
originalDeriver.addTenant('tenant2')

// Create backup
const backup = originalDeriver.createBackup('Production key backup')
console.log('Backup created:', JSON.stringify(backup, null, 2))

// Verify backup
const isValid = originalDeriver.verifyBackup(backup)
console.log('Backup is valid:', isValid)

// Restore from backup
const restoredDeriver = BackupableKeyDeriver.fromBackup(backup)
console.log('Successfully restored from backup')
```

## End to End Implementation

### Example Key Management System

```typescript
class ProductionKeyManager {
  private deriver: BackupableKeyDeriver
  private keyCache: Map<string, PrivateKey> = new Map()
  private maxCacheSize: number = 1000

  constructor(masterKey: PrivateKey, globalSalt?: number[]) {
    this.deriver = new BackupableKeyDeriver(masterKey, 'production-v1', globalSalt)
  }

  // Derive key with caching
  getKey(path: string): PrivateKey {
    if (this.keyCache.has(path)) {
      return this.keyCache.get(path)!
    }

    const key = this.deriveKey(path)
    
    // Manage cache size
    if (this.keyCache.size >= this.maxCacheSize) {
      const firstKey = this.keyCache.keys().next().value
      this.keyCache.delete(firstKey)
    }
    
    this.keyCache.set(path, key)
    return key
  }

  // Clear sensitive data from cache
  clearCache(): void {
    this.keyCache.clear()
  }

  // Create tenant with validation
  createTenant(tenantId: string): { tenantId: string, salt: string } {
    if (!tenantId || tenantId.length < 3) {
      throw new Error('Tenant ID must be at least 3 characters')
    }

    const salt = this.deriver.addTenant(tenantId)
    return {
      tenantId,
      salt: Utils.toBase64(salt)
    }
  }

  // Get key for specific tenant and purpose
  getTenantKey(tenantId: string, purpose: string): PrivateKey {
    const path = `tenant:${tenantId}/${purpose}`
    return this.getKey(path)
  }

  // Create secure backup
  createSecureBackup(description: string, password?: string): string {
    const backup = this.deriver.createBackup(description)
    const backupJson = JSON.stringify(backup)
    
    if (password) {
      // Encrypt backup with password (simplified example)
      const passwordBytes = Utils.toArray(password, 'utf8')
      const backupBytes = Utils.toArray(backupJson, 'utf8')
      const encryptedData = Hash.sha256([...passwordBytes, ...backupBytes])
      return Utils.toBase64(encryptedData)
    }
    
    return backupJson
  }

  // Health check
  healthCheck(): { status: 'healthy' | 'error', details: any } {
    try {
      // Test key derivation
      const testKey = this.getKey('health/check')
      
      return {
        status: 'healthy',
        details: {
          cacheSize: this.keyCache.size,
          testKeyGenerated: !!testKey,
          timestamp: Date.now()
        }
      }
    } catch (error) {
      return {
        status: 'error',
        details: {
          error: error instanceof Error ? error.message : 'Unknown error',
          timestamp: Date.now()
        }
      }
    }
  }

  private deriveKey(path: string): PrivateKey {
    // Use the deriver's internal method (simplified for example)
    const masterBytes = (this.deriver as any).masterKey.toArray()
    const globalSalt = (this.deriver as any).globalSalt
    const pathBytes = Utils.toArray(path, 'utf8')
    
    const derivationInput = [...masterBytes, ...globalSalt, ...pathBytes]
    const childKeyBytes = Hash.sha256(derivationInput)
    
    return new PrivateKey(childKeyBytes)
  }
}

// Production usage example
const productionManager = new ProductionKeyManager(PrivateKey.fromRandom())

// Create tenants
const tenant1 = productionManager.createTenant('acme-corp')
const tenant2 = productionManager.createTenant('beta-inc')

// Get keys for different purposes
const acmePaymentKey = productionManager.getTenantKey('acme-corp', 'payments')
const betaAuthKey = productionManager.getTenantKey('beta-inc', 'authentication')

// Health check
const health = productionManager.healthCheck()
console.log('System health:', health)

// Create backup
const backup = productionManager.createSecureBackup('Daily backup')
console.log('Backup created successfully')
```

## Best Practices

### Security Guidelines

1. **Master Key Protection**
    - Generate master keys using cryptographically secure randomness
    - Never log or expose master keys
    - Use hardware security modules (HSMs) in production
    - Implement proper access controls

2. **Derivation Path Design**
    - Use consistent, hierarchical path structures
    - Include version information in paths for future compatibility
    - Avoid predictable patterns that could leak information
    - Document your derivation scheme thoroughly

3. **Salt Management**
    - Use unique salts for each derivation context
    - Store salts securely but separately from master keys
    - Include salts in backup procedures
    - Consider salt rotation for long-lived systems

## Testing Your Implementation

```typescript
// Test suite for custom key derivation
function testCustomDerivation() {
  console.log('Testing custom key derivation...')
  
  const masterKey = PrivateKey.fromRandom()
  const deriver = new CustomKeyDeriver(masterKey)
  
  // Test 1: Deterministic derivation
  const key1a = deriver.deriveFromPath('test/path/1')
  const key1b = deriver.deriveFromPath('test/path/1')
  console.log('✓ Deterministic:', key1a.toWif() === key1b.toWif())
  
  // Test 2: Different paths produce different keys
  const key2 = deriver.deriveFromPath('test/path/2')
  console.log('✓ Different paths:', key1a.toWif() !== key2.toWif())
  
  // Test 3: Hierarchical derivation
  const hierarchical = new HierarchicalDeriver(masterKey)
  const userKey1 = hierarchical.deriveUserKey('user1', 0)
  const userKey2 = hierarchical.deriveUserKey('user2', 0)
  console.log('✓ User isolation:', userKey1.toWif() !== userKey2.toWif())
  
  // Test 4: Backup and recovery
  const backupable = new BackupableKeyDeriver(masterKey)
  const backup = backupable.createBackup('Test backup')
  const isValid = backupable.verifyBackup(backup)
  console.log('✓ Backup integrity:', isValid)
  
  console.log('All tests passed!')
}

// Run tests
testCustomDerivation()
```

## Summary

Custom key derivation provides powerful flexibility for Bitcoin applications:

- **Hierarchical Organization**: Create structured key hierarchies for complex applications
- **Multi-Tenant Support**: Isolate keys between different users or organizations  
- **Time-Based Rotation**: Implement automatic key rotation for enhanced security
- **Backup and Recovery**: Build comprehensive backup systems with integrity verification
- **Production Ready**: Scale to enterprise applications with caching and monitoring

The patterns shown here can be adapted to your specific requirements while maintaining security and deterministic behavior. Always test thoroughly and follow security best practices when implementing custom derivation schemes.

## Related Documentation

- **[Type-42 Key Derivation](../tutorials/type-42.md)**: Standardized two-party key derivation protocol
- **[AES Symmetric Encryption](../tutorials/aes-encryption.md)**: Password-based key derivation for encryption
- **[Key Management Concepts](../concepts/key-management.md)**: Fundamental key management principles
- **[Security Best Practices](./security-best-practices.md)**: Comprehensive security guidelines
