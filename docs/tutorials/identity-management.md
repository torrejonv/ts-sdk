# Identity Management and Certificates

**Duration**: 90 minutes  
**Prerequisites**: Node.js, basic TypeScript knowledge, understanding of digital certificates  
**Learning Goals**:

- Understand decentralized identity concepts
- Use IdentityClient for certificate management
- Implement identity verification systems
- Build identity-based applications

## Introduction

The BSV ecosystem uses a decentralized identity system based on cryptographic certificates. The `IdentityClient` provides tools for managing identity certificates, verifying identities, and building identity-aware applications.

## Key Concepts

- **Identity Certificates**: Cryptographically signed identity claims
- **Certificate Revelation**: Publicly revealing identity attributes
- **Identity Resolution**: Finding identities by key or attributes
- **Certificate Verification**: Validating identity claims

## What You'll Build

- Identity certificate management system
- Identity verification service
- Public identity revelation
- Identity-based authentication

## Setting Up Identity Management with `WalletClient`

### Basic Identity Operations

```typescript
import { IdentityClient, WalletClient } from '@bsv/sdk'

async function basicIdentityOperations() {
  const wallet = new WalletClient()
  const identityClient = new IdentityClient(wallet)
  
  console.log('Identity client initialized')
  
  // Get our identity key
  const { publicKey: identityKey } = await wallet.getPublicKey({
    identityKey: true
  })
  
  console.log('Our identity key:', identityKey.substring(0, 20) + '...')
  
  return { identityClient, identityKey }
}

basicIdentityOperations().catch(console.error)
```

### Resolving Identities by Key

```typescript
import { IdentityClient, WalletClient } from '@bsv/sdk'

async function resolveIdentityByKey(identityKey: string) {
  const identityClient = new IdentityClient()
  
  try {
    console.log('Resolving identity for key:', identityKey.substring(0, 20) + '...')
    
    const identities = await identityClient.resolveByIdentityKey({
      identityKey
    })
    
    console.log('Found identities:', identities.length)
    
    identities.forEach((identity, index) => {
      console.log(`Identity ${index + 1}:`)
      console.log('  Name:', identity.name)
      console.log('  Badge:', identity.badgeLabel)
      console.log('  Avatar:', identity.avatarURL)
      console.log('  Key:', identity.abbreviatedKey)
    })
    
    return identities
  } catch (error) {
    console.error('Failed to resolve identity:', error)
    return []
  }
}

// Example usage (replace with actual identity key)
// resolveIdentityByKey('actual-identity-key-here').catch(console.error)
```

### Resolving Identities by Attributes

```typescript
import { IdentityClient } from '@bsv/sdk'

async function resolveIdentityByAttributes() {
  const identityClient = new IdentityClient()
  
  try {
    console.log('Searching for identities by attributes...')
    
    const identities = await identityClient.resolveByAttributes({
      attributes: {
        email: 'user@example.com'
      }
    })
    
    console.log('Found identities with matching attributes:', identities.length)
    
    identities.forEach((identity, index) => {
      console.log(`Match ${index + 1}:`)
      console.log('  Name:', identity.name)
      console.log('  Badge:', identity.badgeLabel)
      console.log('  Identity Key:', identity.identityKey.substring(0, 20) + '...')
    })
    
    return identities
  } catch (error) {
    console.error('Failed to resolve by attributes:', error)
    return []
  }
}

resolveIdentityByAttributes().catch(console.error)
```

## Certificate Management System

### Identity Manager Class

```typescript
import { IdentityClient, WalletClient } from '@bsv/sdk'

interface IdentityRecord {
  identityKey: string
  name: string
  badgeLabel: string
  avatarURL: string
  verificationStatus: 'verified' | 'unverified' | 'pending'
  certificates: any[]
  lastUpdated: Date
}

class IdentityManager {
  private identityClient: IdentityClient
  private wallet: WalletClient
  private identityCache: Map<string, IdentityRecord> = new Map()
  
  constructor(wallet?: WalletClient) {
    this.wallet = wallet || new WalletClient()
    this.identityClient = new IdentityClient(this.wallet)
  }
  
  async getOurIdentity(): Promise<string> {
    const { publicKey } = await this.wallet.getPublicKey({
      identityKey: true
    })
    return publicKey
  }
  
  async verifyIdentity(identityKey: string): Promise<IdentityRecord | null> {
    try {
      console.log('Verifying identity:', identityKey.substring(0, 20) + '...')
      
      // Check cache first
      const cached = this.identityCache.get(identityKey)
      if (cached && this.isCacheValid(cached)) {
        console.log('Using cached identity data')
        return cached
      }
      
      // Resolve identity from network
      const identities = await this.identityClient.resolveByIdentityKey({
        identityKey
      })
      
      if (identities.length === 0) {
        console.log('No identity found for key')
        return null
      }
      
      // Use the first identity found
      const identity = identities[0]
      
      const record: IdentityRecord = {
        identityKey,
        name: identity.name,
        badgeLabel: identity.badgeLabel,
        avatarURL: identity.avatarURL,
        verificationStatus: 'verified',
        certificates: [], // Would contain actual certificates
        lastUpdated: new Date()
      }
      
      // Cache the result
      this.identityCache.set(identityKey, record)
      
      console.log('Identity verified:', record.name)
      return record
    } catch (error) {
      console.error('Identity verification failed:', error)
      return null
    }
  }
  
  async searchIdentities(query: {
    name?: string
    email?: string
    attributes?: Record<string, string>
  }): Promise<IdentityRecord[]> {
    try {
      console.log('Searching identities with query:', query)
      
      const searchAttributes = {
        ...query.attributes,
        ...(query.name && { name: query.name }),
        ...(query.email && { email: query.email })
      }
      
      const identities = await this.identityClient.resolveByAttributes({
        attributes: searchAttributes
      })
      
      const records: IdentityRecord[] = identities.map(identity => ({
        identityKey: identity.identityKey,
        name: identity.name,
        badgeLabel: identity.badgeLabel,
        avatarURL: identity.avatarURL,
        verificationStatus: 'verified' as const,
        certificates: [],
        lastUpdated: new Date()
      }))
      
      console.log('Found', records.length, 'matching identities')
      return records
    } catch (error) {
      console.error('Identity search failed:', error)
      return []
    }
  }
  
  async revealPublicAttributes(
    certificate: any,
    fieldsToReveal: string[]
  ): Promise<boolean> {
    try {
      console.log('Revealing public attributes:', fieldsToReveal)
      
      const result = await this.identityClient.publiclyRevealAttributes(
        certificate,
        fieldsToReveal
      )
      
      console.log('Attributes revealed successfully')
      return true
    } catch (error) {
      console.error('Failed to reveal attributes:', error)
      return false
    }
  }
  
  private isCacheValid(record: IdentityRecord): boolean {
    const cacheAge = Date.now() - record.lastUpdated.getTime()
    const maxAge = 60 * 60 * 1000 // 1 hour
    return cacheAge < maxAge
  }
  
  getCachedIdentities(): IdentityRecord[] {
    return Array.from(this.identityCache.values())
  }
  
  clearCache(): void {
    this.identityCache.clear()
    console.log('Identity cache cleared')
  }
}

async function demonstrateIdentityManager() {
  const identityManager = new IdentityManager()
  
  console.log('=== Identity Manager Demo ===')
  
  // Get our own identity
  const ourIdentity = await identityManager.getOurIdentity()
  console.log('Our identity key:', ourIdentity.substring(0, 20) + '...')
  
  // Search for identities
  const searchResults = await identityManager.searchIdentities({
    name: 'John'
  })
  
  console.log('Search results:', searchResults.length)
  
  // Verify specific identities
  for (const result of searchResults.slice(0, 2)) { // Limit to first 2
    const verified = await identityManager.verifyIdentity(result.identityKey)
    if (verified) {
      console.log('Verified identity:', verified.name)
    }
  }
  
  // Show cached identities
  const cached = identityManager.getCachedIdentities()
  console.log('Cached identities:', cached.length)
  
  return { ourIdentity, searchResults, cached }
}

demonstrateIdentityManager().catch(console.error)
```

## Identity-Based Authentication

### Authentication Service

```typescript
import { IdentityClient, WalletClient } from '@bsv/sdk'

interface AuthenticationResult {
  success: boolean
  identityKey?: string
  identity?: any
  error?: string
}

class IdentityAuthService {
  private identityClient: IdentityClient
  private authenticatedUsers: Map<string, {
    identity: any
    sessionStart: Date
    lastActivity: Date
  }> = new Map()
  
  constructor(wallet?: WalletClient) {
    this.identityClient = new IdentityClient(wallet)
  }
  
  async authenticateUser(identityKey: string): Promise<AuthenticationResult> {
    try {
      console.log('Authenticating user:', identityKey.substring(0, 20) + '...')
      
      // Resolve identity
      const identities = await this.identityClient.resolveByIdentityKey({
        identityKey
      })
      
      if (identities.length === 0) {
        return {
          success: false,
          error: 'Identity not found'
        }
      }
      
      const identity = identities[0]
      
      // Create session
      this.authenticatedUsers.set(identityKey, {
        identity,
        sessionStart: new Date(),
        lastActivity: new Date()
      })
      
      console.log('User authenticated:', identity.name)
      
      return {
        success: true,
        identityKey,
        identity
      }
    } catch (error) {
      console.error('Authentication failed:', error)
      return {
        success: false,
        error: error.message
      }
    }
  }
  
  async verifySession(identityKey: string): Promise<boolean> {
    const session = this.authenticatedUsers.get(identityKey)
    
    if (!session) {
      return false
    }
    
    // Check session timeout (24 hours)
    const sessionAge = Date.now() - session.sessionStart.getTime()
    const maxAge = 24 * 60 * 60 * 1000
    
    if (sessionAge > maxAge) {
      this.authenticatedUsers.delete(identityKey)
      return false
    }
    
    // Update last activity
    session.lastActivity = new Date()
    return true
  }
  
  async requireAuthentication(identityKey: string): Promise<{
    authorized: boolean
    identity?: any
    error?: string
  }> {
    const isValid = await this.verifySession(identityKey)
    
    if (!isValid) {
      return {
        authorized: false,
        error: 'Authentication required'
      }
    }
    
    const session = this.authenticatedUsers.get(identityKey)
    return {
      authorized: true,
      identity: session?.identity
    }
  }
  
  logout(identityKey: string): void {
    this.authenticatedUsers.delete(identityKey)
    console.log('User logged out:', identityKey.substring(0, 20) + '...')
  }
  
  getActiveSessions(): Array<{
    identityKey: string
    name: string
    sessionStart: Date
    lastActivity: Date
  }> {
    return Array.from(this.authenticatedUsers.entries()).map(([key, session]) => ({
      identityKey: key,
      name: session.identity.name,
      sessionStart: session.sessionStart,
      lastActivity: session.lastActivity
    }))
  }
}

async function demonstrateAuthentication() {
  const authService = new IdentityAuthService()
  
  console.log('=== Identity Authentication Demo ===')
  
  // Simulate user authentication
  const wallet = new WalletClient()
  const { publicKey: userIdentity } = await wallet.getPublicKey({
    identityKey: true
  })
  
  // Authenticate user
  const authResult = await authService.authenticateUser(userIdentity)
  
  if (authResult.success) {
    console.log('✅ Authentication successful')
    console.log('User:', authResult.identity?.name || 'Unknown')
    
    // Verify session
    const sessionValid = await authService.verifySession(userIdentity)
    console.log('Session valid:', sessionValid)
    
    // Test authorization
    const authCheck = await authService.requireAuthentication(userIdentity)
    console.log('Authorization check:', authCheck.authorized ? 'PASSED' : 'FAILED')
    
    // Show active sessions
    const sessions = authService.getActiveSessions()
    console.log('Active sessions:', sessions.length)
    
    // Logout
    authService.logout(userIdentity)
    console.log('User logged out')
    
  } else {
    console.log('❌ Authentication failed:', authResult.error)
  }
  
  return authResult
}

demonstrateAuthentication().catch(console.error)
```

## Identity Verification Service

### Complete Verification System

```typescript
import { IdentityClient, WalletClient } from '@bsv/sdk'

interface VerificationRequest {
  identityKey: string
  requiredCertificates: string[]
  purpose: string
  timestamp: Date
}

interface VerificationResult {
  verified: boolean
  identityKey: string
  certificates: any[]
  trustScore: number
  issues: string[]
}

class IdentityVerificationService {
  private identityClient: IdentityClient
  private verificationHistory: Map<string, VerificationResult[]> = new Map()
  
  constructor(wallet?: WalletClient) {
    this.identityClient = new IdentityClient(wallet)
  }
  
  async verifyIdentity(request: VerificationRequest): Promise<VerificationResult> {
    console.log('Starting identity verification for:', request.identityKey.substring(0, 20) + '...')
    console.log('Purpose:', request.purpose)
    
    const result: VerificationResult = {
      verified: false,
      identityKey: request.identityKey,
      certificates: [],
      trustScore: 0,
      issues: []
    }
    
    try {
      // Resolve identity
      const identities = await this.identityClient.resolveByIdentityKey({
        identityKey: request.identityKey
      })
      
      if (identities.length === 0) {
        result.issues.push('Identity not found')
        return result
      }
      
      const identity = identities[0]
      
      // Calculate trust score based on available information
      result.trustScore = this.calculateTrustScore(identity)
      
      // Check if verification meets requirements
      if (result.trustScore >= 70) { // Minimum trust score
        result.verified = true
        console.log('✅ Identity verified:', identity.name)
      } else {
        result.issues.push('Insufficient trust score')
        console.log('❌ Identity verification failed: low trust score')
      }
      
      // Store verification history
      const history = this.verificationHistory.get(request.identityKey) || []
      history.push(result)
      this.verificationHistory.set(request.identityKey, history)
      
    } catch (error) {
      result.issues.push(`Verification error: ${error.message}`)
      console.error('Verification error:', error)
    }
    
    return result
  }
  
  private calculateTrustScore(identity: any): number {
    let score = 0
    
    // Base score for having an identity
    score += 30
    
    // Score for having a name
    if (identity.name && identity.name !== 'Unknown') {
      score += 20
    }
    
    // Score for having an avatar
    if (identity.avatarURL) {
      score += 10
    }
    
    // Score for having a badge
    if (identity.badgeLabel) {
      score += 20
    }
    
    // Additional score for verified badge
    if (identity.badgeLabel?.includes('Verified')) {
      score += 20
    }
    
    return Math.min(score, 100) // Cap at 100
  }
  
  async batchVerify(identityKeys: string[], purpose: string): Promise<{
    verified: VerificationResult[]
    failed: VerificationResult[]
  }> {
    console.log(`Starting batch verification of ${identityKeys.length} identities`)
    
    const requests = identityKeys.map(key => ({
      identityKey: key,
      requiredCertificates: [],
      purpose,
      timestamp: new Date()
    }))
    
    const results = await Promise.all(
      requests.map(request => this.verifyIdentity(request))
    )
    
    const verified = results.filter(r => r.verified)
    const failed = results.filter(r => !r.verified)
    
    console.log(`Batch verification complete: ${verified.length} verified, ${failed.length} failed`)
    
    return { verified, failed }
  }
  
  getVerificationHistory(identityKey: string): VerificationResult[] {
    return this.verificationHistory.get(identityKey) || []
  }
  
  getTrustStatistics(): {
    totalVerifications: number
    successRate: number
    averageTrustScore: number
  } {
    let totalVerifications = 0
    let successfulVerifications = 0
    let totalTrustScore = 0
    
    for (const history of this.verificationHistory.values()) {
      for (const result of history) {
        totalVerifications++
        totalTrustScore += result.trustScore
        
        if (result.verified) {
          successfulVerifications++
        }
      }
    }
    
    return {
      totalVerifications,
      successRate: totalVerifications > 0 ? successfulVerifications / totalVerifications : 0,
      averageTrustScore: totalVerifications > 0 ? totalTrustScore / totalVerifications : 0
    }
  }
}

async function demonstrateVerificationService() {
  const verificationService = new IdentityVerificationService()
  
  console.log('=== Identity Verification Service Demo ===')
  
  // Create test identities
  const wallet1 = new WalletClient()
  const wallet2 = new WalletClient()
  
  const { publicKey: identity1 } = await wallet1.getPublicKey({ identityKey: true })
  const { publicKey: identity2 } = await wallet2.getPublicKey({ identityKey: true })
  
  const testIdentities = [identity1, identity2]
  
  // Batch verification
  const batchResult = await verificationService.batchVerify(
    testIdentities,
    'User registration verification'
  )
  
  console.log('Batch verification results:')
  console.log('  Verified:', batchResult.verified.length)
  console.log('  Failed:', batchResult.failed.length)
  
  // Individual verification with details
  for (const identity of testIdentities.slice(0, 1)) { // Just first one
    const request: VerificationRequest = {
      identityKey: identity,
      requiredCertificates: ['identity'],
      purpose: 'Account access verification',
      timestamp: new Date()
    }
    
    const result = await verificationService.verifyIdentity(request)
    
    console.log(`\nVerification for ${identity.substring(0, 20)}...`)
    console.log('  Verified:', result.verified)
    console.log('  Trust Score:', result.trustScore)
    console.log('  Issues:', result.issues)
  }
  
  // Get statistics
  const stats = verificationService.getTrustStatistics()
  console.log('\nVerification Statistics:')
  console.log('  Total Verifications:', stats.totalVerifications)
  console.log('  Success Rate:', (stats.successRate * 100).toFixed(1) + '%')
  console.log('  Average Trust Score:', stats.averageTrustScore.toFixed(1))
  
  return { batchResult, stats }
}

demonstrateVerificationService().catch(console.error)
```

## Conclusion

Congratulations! You've successfully built a comprehensive identity management system using the BSV TypeScript SDK. In this tutorial, you've learned how to create, manage, and verify digital identities on the BSV blockchain.

### Core Concepts Mastered

1. **Identity Creation**: Generated unique BSV identities using cryptographic key pairs
2. **Certificate Management**: Created and managed identity certificates for enhanced trust
3. **Identity Resolution**: Implemented identity lookup and verification services
4. **Trust Scoring**: Built systems to calculate and evaluate identity trustworthiness
5. **Batch Processing**: Optimized identity operations for handling multiple identities efficiently

## Next Steps

- Learn about [AuthFetch Integration](./authfetch-tutorial.md) to use identities for API authentication
- Explore [Identity Verification Systems](../guides/identity-verification-systems.md) for advanced verification patterns

## Additional Resources

- [Identity Client API Reference](../reference/identity.md)
- [Certificate Management Guide](../concepts/identity-certificates.md)
- [Decentralized Identity Concepts](../concepts/decentralized-identity.md)
- [BSV Identity Standards](https://projectbabbage.com/docs/guides/identity)
