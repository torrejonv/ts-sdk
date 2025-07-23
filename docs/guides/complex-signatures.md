# Verifying Complex Signatures

This guide covers advanced signature verification patterns and complex validation scenarios using the BSV TypeScript SDK. While basic signature verification is covered in other tutorials, this guide focuses on sophisticated verification workflows, batch processing, threshold schemes, and performance optimization.

## Prerequisites

- Understanding of [Elliptic Curve Fundamentals](../tutorials/elliptic-curve-fundamentals.md)
- Familiarity with [Transaction Signing Methods](./transaction-signing-methods.md)
- Knowledge of [Multi-Signature Transactions](./multisig-transactions.md)
- Basic understanding of [Error Handling](../tutorials/error-handling.md)

## Overview

Complex signature verification involves scenarios beyond simple single-signature validation:

- **Batch Verification**: Efficiently verifying multiple signatures
- **Threshold Signatures**: Validating threshold signature schemes
- **Multi-Context Validation**: Handling different signature types together
- **Performance Optimization**: Caching and parallel processing
- **Complex Scenarios**: Time-locked, conditional, and multi-party signatures
- **Robust Error Handling**: Detailed validation with recovery strategies

## 1. Batch Signature Verification

When dealing with multiple signatures, batch verification provides significant performance improvements over individual verification.

### Basic Batch Verifier

```typescript
import { PublicKey, Signature, Hash } from '@bsv/sdk'

interface SignatureVerificationItem {
  message: string | number[]
  signature: Signature
  publicKey: PublicKey
  id?: string // Optional identifier for tracking
}

interface BatchVerificationResult {
  allValid: boolean
  results: Array<{
    id?: string
    valid: boolean
    error?: string
  }>
  totalProcessed: number
  processingTimeMs: number
}

class BatchSignatureVerifier {
  private cache = new Map<string, boolean>()
  private maxCacheSize = 1000

  /**
   * Verify multiple signatures efficiently with caching
   */
  async verifyBatch(items: SignatureVerificationItem[]): Promise<BatchVerificationResult> {
    const startTime = Date.now()
    const results: BatchVerificationResult['results'] = []
    let allValid = true

    try {
      // Process signatures in parallel for better performance
      const verificationPromises = items.map(async (item, index) => {
        try {
          const cacheKey = this.getCacheKey(item)
          
          // Check cache first
          if (this.cache.has(cacheKey)) {
            return {
              id: item.id || `item-${index}`,
              valid: this.cache.get(cacheKey)!,
              cached: true
            }
          }

          // Verify signature
          const isValid = item.signature.verify(item.message, item.publicKey)
          
          // Cache result
          this.cacheResult(cacheKey, isValid)
          
          return {
            id: item.id || `item-${index}`,
            valid: isValid
          }
        } catch (error) {
          return {
            id: item.id || `item-${index}`,
            valid: false,
            error: error instanceof Error ? error.message : 'Unknown verification error'
          }
        }
      })

      const verificationResults = await Promise.all(verificationPromises)
      
      for (const result of verificationResults) {
        results.push(result)
        if (!result.valid) {
          allValid = false
        }
      }

      return {
        allValid,
        results,
        totalProcessed: items.length,
        processingTimeMs: Date.now() - startTime
      }

    } catch (error) {
      throw new Error(`Batch verification failed: ${error instanceof Error ? error.message : 'Unknown error'}`)
    }
  }

  /**
   * Generate cache key for signature verification
   */
  private getCacheKey(item: SignatureVerificationItem): string {
    const messageHash = typeof item.message === 'string' 
      ? Hash.sha256(item.message, 'utf8') 
      : Hash.sha256(item.message)
    
    return `${Buffer.from(messageHash).toString('hex')}-${item.signature.toDER('hex')}-${item.publicKey.toString()}`
  }

  /**
   * Cache verification result with LRU eviction
   */
  private cacheResult(key: string, result: boolean): void {
    if (this.cache.size >= this.maxCacheSize) {
      // Remove oldest entry (LRU)
      const firstKey = this.cache.keys().next().value
      if (firstKey !== undefined) {
        this.cache.delete(firstKey)
      }
    }
    this.cache.set(key, result)
  }

  /**
   * Clear verification cache
   */
  clearCache(): void {
    this.cache.clear()
  }

  /**
   * Get cache statistics
   */
  getCacheStats(): { size: number; maxSize: number; hitRate?: number } {
    return {
      size: this.cache.size,
      maxSize: this.maxCacheSize
    }
  }
}
```

### Advanced Batch Processing with Error Recovery

```typescript
interface BatchVerificationOptions {
  maxConcurrency?: number
  timeoutMs?: number
  retryFailedItems?: boolean
  maxRetries?: number
  continueOnError?: boolean
}

class AdvancedBatchVerifier extends BatchSignatureVerifier {
  /**
   * Advanced batch verification with configurable options
   */
  async verifyBatchAdvanced(
    items: SignatureVerificationItem[], 
    options: BatchVerificationOptions = {}
  ): Promise<BatchVerificationResult> {
    const {
      maxConcurrency = 10,
      timeoutMs = 30000,
      retryFailedItems = true,
      maxRetries = 2,
      continueOnError = true
    } = options

    const startTime = Date.now()
    let allValid = true
    const results: BatchVerificationResult['results'] = []

    try {
      // Process in chunks to control concurrency
      const chunks = this.chunkArray(items, maxConcurrency)
      
      for (const chunk of chunks) {
        const chunkResults = await Promise.allSettled(
          chunk.map(async (item, index) => {
            const timeoutPromise = new Promise((_, reject) => {
              setTimeout(() => reject(new Error('Verification timeout')), timeoutMs)
            })

            const verificationPromise = this.verifyWithRetry(item, maxRetries)
            
            return Promise.race([verificationPromise, timeoutPromise])
          })
        )

        // Process chunk results
        for (let i = 0; i < chunkResults.length; i++) {
          const result = chunkResults[i]
          const item = chunk[i]
          
          if (result.status === 'fulfilled') {
            results.push(result.value)
            if (!result.value.valid) {
              allValid = false
            }
          } else {
            const errorResult = {
              id: item.id || `item-${results.length}`,
              valid: false,
              error: result.reason?.message || 'Verification failed'
            }
            results.push(errorResult)
            allValid = false
            
            if (!continueOnError) {
              throw new Error(`Batch verification stopped due to error: ${errorResult.error}`)
            }
          }
        }
      }

      return {
        allValid,
        results,
        totalProcessed: items.length,
        processingTimeMs: Date.now() - startTime
      }

    } catch (error) {
      throw new Error(`Advanced batch verification failed: ${error instanceof Error ? error.message : 'Unknown error'}`)
    }
  }

  /**
   * Verify single item with retry logic
   */
  private async verifyWithRetry(
    item: SignatureVerificationItem, 
    maxRetries: number
  ): Promise<{ id?: string; valid: boolean; error?: string; attempts: number }> {
    let lastError: Error | undefined
    
    for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
      try {
        const cacheKey = this.getCacheKey(item)
        
        // Check cache
        if (this.cache.has(cacheKey)) {
          return {
            id: item.id,
            valid: this.cache.get(cacheKey)!,
            attempts: attempt
          }
        }

        // Verify signature
        const isValid = item.signature.verify(item.message, item.publicKey)
        
        // Cache result
        this.cacheResult(cacheKey, isValid)
        
        return {
          id: item.id,
          valid: isValid,
          attempts: attempt
        }
      } catch (error) {
        lastError = error instanceof Error ? error : new Error('Unknown error')
        
        if (attempt <= maxRetries) {
          // Wait before retry with exponential backoff
          await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempt - 1) * 100))
        }
      }
    }

    return {
      id: item.id,
      valid: false,
      error: lastError?.message || 'Max retries exceeded',
      attempts: maxRetries + 1
    }
  }

  /**
   * Split array into chunks for controlled concurrency
   */
  private chunkArray<T>(array: T[], chunkSize: number): T[][] {
    const chunks: T[][] = []
    for (let i = 0; i < array.length; i += chunkSize) {
      chunks.push(array.slice(i, i + chunkSize))
    }
    return chunks
  }
}
```

## 2. Threshold Signature Verification

The SDK supports threshold signature schemes using polynomial interpolation. Here's how to verify threshold signatures:

### Basic Threshold Verification

```typescript
import { PrivateKey, PublicKey, Signature } from '@bsv/sdk'

interface ThresholdSignatureData {
  message: string | number[]
  signatures: Signature[]
  publicKeys: PublicKey[]
  threshold: number
  totalShares: number
}

interface ThresholdVerificationResult {
  valid: boolean
  validSignatures: number
  requiredSignatures: number
  details: Array<{
    index: number
    valid: boolean
    publicKey: string
    error?: string
  }>
}

class ThresholdSignatureVerifier {
  /**
   * Verify threshold signature scheme
   */
  async verifyThresholdSignature(data: ThresholdSignatureData): Promise<ThresholdVerificationResult> {
    const { message, signatures, publicKeys, threshold, totalShares } = data

    // Validate input parameters
    if (signatures.length !== publicKeys.length) {
      throw new Error('Number of signatures must match number of public keys')
    }

    if (threshold > totalShares) {
      throw new Error('Threshold cannot exceed total shares')
    }

    if (signatures.length > totalShares) {
      throw new Error('Cannot have more signatures than total shares')
    }

    const details: ThresholdVerificationResult['details'] = []
    let validSignatures = 0

    // Verify each signature
    for (let i = 0; i < signatures.length; i++) {
      try {
        const isValid = signatures[i].verify(message, publicKeys[i])
        
        details.push({
          index: i,
          valid: isValid,
          publicKey: publicKeys[i].toString()
        })

        if (isValid) {
          validSignatures++
        }
      } catch (error) {
        details.push({
          index: i,
          valid: false,
          publicKey: publicKeys[i].toString(),
          error: error instanceof Error ? error.message : 'Verification failed'
        })
      }
    }

    return {
      valid: validSignatures >= threshold,
      validSignatures,
      requiredSignatures: threshold,
      details
    }
  }

  /**
   * Verify threshold signature with key share reconstruction
   */
  async verifyWithKeyShares(
    message: string | number[],
    signatures: Signature[],
    keyShares: Array<{ x: number; y: number[] }>,
    threshold: number,
    expectedPublicKey?: PublicKey
  ): Promise<{ valid: boolean; reconstructedKey?: PublicKey; error?: string }> {
    try {
      // Validate we have enough shares
      if (keyShares.length < threshold) {
        return {
          valid: false,
          error: `Insufficient key shares: need ${threshold}, got ${keyShares.length}`
        }
      }

      // Convert key shares to the format expected by PrivateKey.fromKeyShares
      const points = keyShares.slice(0, threshold).map(share => ({
        x: new (await import('@bsv/sdk')).BigNumber(share.x),
        y: new (await import('@bsv/sdk')).BigNumber(share.y)
      }))

      // For demonstration, we'll verify signatures against individual shares
      // In a real threshold scheme, you'd reconstruct the master key
      let validSignatures = 0
      
      for (let i = 0; i < Math.min(signatures.length, threshold); i++) {
        try {
          // Create public key from key share (simplified approach)
          const sharePrivateKey = new PrivateKey(keyShares[i].y)
          const sharePublicKey = sharePrivateKey.toPublicKey()
          
          const isValid = signatures[i].verify(message, sharePublicKey)
          if (isValid) {
            validSignatures++
          }
        } catch (error) {
          console.warn(`Failed to verify signature ${i}:`, error)
        }
      }

      return {
        valid: validSignatures >= threshold,
        reconstructedKey: expectedPublicKey // Would be reconstructed in real implementation
      }
    } catch (error) {
      return {
        valid: false,
        error: error instanceof Error ? error.message : 'Threshold verification failed'
      }
    }
  }
}
```

## 3. Multi-Context Signature Validation

Handle different types of signatures in a single validation workflow:

### Multi-Context Verifier

```typescript
import { PublicKey, Signature, TransactionSignature, Transaction } from '@bsv/sdk'

interface MessageSignatureContext {
  type: 'message'
  message: string | number[]
  signature: Signature
  publicKey: PublicKey
}

interface TransactionSignatureContext {
  type: 'transaction'
  transaction: Transaction
  inputIndex: number
  signature: TransactionSignature
  publicKey: PublicKey
}

interface WalletSignatureContext {
  type: 'wallet'
  data: number[]
  signature: number[]
  protocolID: [number, string]
  keyID: string
  counterparty?: string
}

type SignatureContext = MessageSignatureContext | TransactionSignatureContext | WalletSignatureContext

interface MultiContextResult {
  allValid: boolean
  results: Array<{
    type: string
    valid: boolean
    context?: any
    error?: string
  }>
  summary: {
    total: number
    valid: number
    invalid: number
    byType: Record<string, { valid: number; invalid: number }>
  }
}

class MultiContextSignatureVerifier {
  private walletClient?: any // `WalletClient` instance

  constructor(walletClient?: any) {
    this.walletClient = walletClient
  }

  /**
   * Verify signatures across different contexts
   */
  async verifyMultiContext(contexts: SignatureContext[]): Promise<MultiContextResult> {
    const results: MultiContextResult['results'] = []
    const summary = {
      total: contexts.length,
      valid: 0,
      invalid: 0,
      byType: {} as Record<string, { valid: number; invalid: number }>
    }

    let allValid = true

    for (const context of contexts) {
      try {
        let isValid = false
        let contextInfo: any = {}

        switch (context.type) {
          case 'message':
            isValid = await this.verifyMessageSignature(context)
            contextInfo = { messageLength: typeof context.message === 'string' ? context.message.length : context.message.length }
            break

          case 'transaction':
            isValid = await this.verifyTransactionSignature(context)
            contextInfo = { inputIndex: context.inputIndex, txid: Buffer.from(context.transaction.id()).toString('hex') }
            break

          case 'wallet':
            isValid = await this.verifyWalletSignature(context)
            contextInfo = { protocolID: context.protocolID, keyID: context.keyID }
            break

          default:
            throw new Error(`Unknown signature context type: ${(context as any).type}`)
        }

        results.push({
          type: context.type,
          valid: isValid,
          context: contextInfo
        })

        // Update summary
        if (!summary.byType[context.type]) {
          summary.byType[context.type] = { valid: 0, invalid: 0 }
        }

        if (isValid) {
          summary.valid++
          summary.byType[context.type].valid++
        } else {
          summary.invalid++
          summary.byType[context.type].invalid++
          allValid = false
        }

      } catch (error) {
        results.push({
          type: context.type,
          valid: false,
          error: error instanceof Error ? error.message : 'Verification failed'
        })

        summary.invalid++
        if (!summary.byType[context.type]) {
          summary.byType[context.type] = { valid: 0, invalid: 0 }
        }
        summary.byType[context.type].invalid++
        allValid = false
      }
    }

    return {
      allValid,
      results,
      summary
    }
  }

  /**
   * Verify message signature
   */
  private async verifyMessageSignature(context: MessageSignatureContext): Promise<boolean> {
    return context.signature.verify(context.message, context.publicKey)
  }

  /**
   * Verify transaction signature
   */
  private async verifyTransactionSignature(context: TransactionSignatureContext): Promise<boolean> {
    try {
      // Use the transaction's built-in verification
      const isValid = context.transaction.verify('scripts only')
      
      if (!isValid) {
        return false
      }

      // Additional verification for specific input if needed
      const input = context.transaction.inputs[context.inputIndex]
      if (!input || !input.unlockingScript) {
        return false
      }

      return true
    } catch (error) {
      console.warn('Transaction signature verification failed:', error)
      return false
    }
  }

  /**
   * Verify wallet signature using `WalletClient`
   */
  private async verifyWalletSignature(context: WalletSignatureContext): Promise<boolean> {
    if (!this.walletClient) {
      throw new Error('`WalletClient` required for wallet signature verification')
    }

    try {
      const result = await this.walletClient.verifySignature({
        data: context.data,
        signature: context.signature,
        protocolID: context.protocolID,
        keyID: context.keyID,
        counterparty: context.counterparty || 'self'
      })

      return result.valid === true
    } catch (error) {
      console.warn('Wallet signature verification failed:', error)
      return false
    }
  }
}
```

## 4. Performance-Optimized Verification

For high-throughput applications, signature verification performance is critical. Here are optimization strategies:

### Signature Cache with Performance Monitoring

```typescript
interface PerformanceMetrics {
  totalVerifications: number
  cacheHits: number
  cacheMisses: number
  averageVerificationTime: number
  peakVerificationTime: number
}

class PerformanceOptimizedVerifier {
  private cache = new Map<string, { result: boolean; timestamp: number }>()
  private metrics: PerformanceMetrics = {
    totalVerifications: 0,
    cacheHits: 0,
    cacheMisses: 0,
    averageVerificationTime: 0,
    peakVerificationTime: 0
  }
  private verificationTimes: number[] = []
  private maxCacheAge = 300000 // 5 minutes
  private maxCacheSize = 5000

  /**
   * High-performance signature verification with caching and metrics
   */
  async verifyOptimized(
    message: string | number[],
    signature: Signature,
    publicKey: PublicKey
  ): Promise<{ valid: boolean; fromCache: boolean; verificationTime: number }> {
    const startTime = performance.now()
    const cacheKey = this.generateCacheKey(message, signature, publicKey)

    // Check cache first
    const cached = this.cache.get(cacheKey)
    if (cached && (Date.now() - cached.timestamp) < this.maxCacheAge) {
      this.metrics.cacheHits++
      this.metrics.totalVerifications++
      
      return {
        valid: cached.result,
        fromCache: true,
        verificationTime: performance.now() - startTime
      }
    }

    // Perform verification
    try {
      const isValid = signature.verify(message, publicKey)
      const verificationTime = performance.now() - startTime
      
      // Update cache
      this.updateCache(cacheKey, isValid)
      
      // Update metrics
      this.updateMetrics(verificationTime, false)
      
      return {
        valid: isValid,
        fromCache: false,
        verificationTime
      }
    } catch (error) {
      const verificationTime = performance.now() - startTime
      this.updateMetrics(verificationTime, false)
      
      throw new Error(`Signature verification failed: ${error instanceof Error ? error.message : 'Unknown error'}`)
    }
  }

  /**
   * Parallel verification with worker threads simulation
   */
  async verifyParallel(
    items: Array<{
      message: string | number[]
      signature: Signature
      publicKey: PublicKey
      id?: string
    }>,
    maxConcurrency = 4
  ): Promise<Array<{ id?: string; valid: boolean; verificationTime: number }>> {
    const chunks = this.chunkArray(items, maxConcurrency)
    const results: Array<{ id?: string; valid: boolean; verificationTime: number }> = []

    for (const chunk of chunks) {
      const chunkPromises = chunk.map(async (item, index) => {
        const result = await this.verifyOptimized(item.message, item.signature, item.publicKey)
        return {
          id: item.id || `item-${results.length + index}`,
          valid: result.valid,
          verificationTime: result.verificationTime
        }
      })

      const chunkResults = await Promise.all(chunkPromises)
      results.push(...chunkResults)
    }

    return results
  }

  /**
   * Generate cache key for signature verification
   */
  private generateCacheKey(
    message: string | number[],
    signature: Signature,
    publicKey: PublicKey
  ): string {
    const messageStr = typeof message === 'string' ? message : Buffer.from(message).toString('hex')
    return `${messageStr}-${signature.toDER('hex')}-${publicKey.toString()}`
  }

  /**
   * Update cache with LRU eviction
   */
  private updateCache(key: string, result: boolean): void {
    // Remove expired entries
    const now = Date.now()
    for (const [cacheKey, value] of this.cache.entries()) {
      if (now - value.timestamp > this.maxCacheAge) {
        this.cache.delete(cacheKey)
      }
    }

    // LRU eviction if needed
    if (this.cache.size >= this.maxCacheSize) {
      const firstKey = this.cache.keys().next().value
      if (firstKey !== undefined) {
        this.cache.delete(firstKey)
      }
    }

    this.cache.set(key, { result, timestamp: now })
  }

  /**
   * Update performance metrics
   */
  private updateMetrics(verificationTime: number, fromCache: boolean): void {
    this.metrics.totalVerifications++
    
    if (fromCache) {
      this.metrics.cacheHits++
    } else {
      this.metrics.cacheMisses++
    }

    this.verificationTimes.push(verificationTime)
    
    // Keep only last 1000 measurements for average calculation
    if (this.verificationTimes.length > 1000) {
      this.verificationTimes = this.verificationTimes.slice(-1000)
    }

    this.metrics.averageVerificationTime = 
      this.verificationTimes.reduce((sum, time) => sum + time, 0) / this.verificationTimes.length
    
    this.metrics.peakVerificationTime = Math.max(this.metrics.peakVerificationTime, verificationTime)
  }

  /**
   * Get performance metrics
   */
  getMetrics(): PerformanceMetrics & { cacheHitRate: number } {
    const cacheHitRate = this.metrics.totalVerifications > 0 
      ? (this.metrics.cacheHits / this.metrics.totalVerifications) * 100 
      : 0

    return {
      ...this.metrics,
      cacheHitRate
    }
  }

  /**
   * Clear cache and reset metrics
   */
  reset(): void {
    this.cache.clear()
    this.metrics = {
      totalVerifications: 0,
      cacheHits: 0,
      cacheMisses: 0,
      averageVerificationTime: 0,
      peakVerificationTime: 0
    }
    this.verificationTimes = []
  }

  /**
   * Split array into chunks
   */
  private chunkArray<T>(array: T[], chunkSize: number): T[][] {
    const chunks: T[][] = []
    for (let i = 0; i < array.length; i += chunkSize) {
      chunks.push(array.slice(i, i + chunkSize))
    }
    return chunks
  }
}
```

## 5. Complex Validation Scenarios

Real-world applications often require validating signatures in complex scenarios involving time locks, conditional logic, and multi-party coordination.

### Time-Locked Signature Validation

```typescript
interface TimeLockSignatureData {
  message: string | number[]
  signature: Signature
  publicKey: PublicKey
  lockTime: number // Unix timestamp
  gracePeriod?: number // Optional grace period in seconds
}

class TimeLockSignatureVerifier {
  /**
   * Verify signature with time lock validation
   */
  async verifyTimeLocked(data: TimeLockSignatureData): Promise<{
    valid: boolean
    timeLockValid: boolean
    signatureValid: boolean
    currentTime: number
    lockTime: number
    error?: string
  }> {
    const currentTime = Math.floor(Date.now() / 1000)
    const { message, signature, publicKey, lockTime, gracePeriod = 0 } = data

    try {
      // Check if time lock has expired
      const timeLockValid = currentTime >= (lockTime - gracePeriod)
      
      // Verify signature regardless of time lock for complete validation
      const signatureValid = signature.verify(message, publicKey)
      
      return {
        valid: timeLockValid && signatureValid,
        timeLockValid,
        signatureValid,
        currentTime,
        lockTime
      }
    } catch (error) {
      return {
        valid: false,
        timeLockValid: false,
        signatureValid: false,
        currentTime,
        lockTime,
        error: error instanceof Error ? error.message : 'Verification failed'
      }
    }
  }
}
```

## 6. Error Handling and Recovery

Robust error handling is crucial for complex signature verification systems. Here's a comprehensive approach:

### Comprehensive Error Handler

```typescript
enum SignatureErrorType {
  INVALID_SIGNATURE = 'INVALID_SIGNATURE',
  INVALID_PUBLIC_KEY = 'INVALID_PUBLIC_KEY',
  INVALID_MESSAGE = 'INVALID_MESSAGE',
  VERIFICATION_TIMEOUT = 'VERIFICATION_TIMEOUT',
  CACHE_ERROR = 'CACHE_ERROR',
  BATCH_PROCESSING_ERROR = 'BATCH_PROCESSING_ERROR',
  THRESHOLD_NOT_MET = 'THRESHOLD_NOT_MET',
  DEPENDENCY_ERROR = 'DEPENDENCY_ERROR',
  TIME_LOCK_ERROR = 'TIME_LOCK_ERROR',
  NETWORK_ERROR = 'NETWORK_ERROR',
  UNKNOWN_ERROR = 'UNKNOWN_ERROR'
}

interface SignatureError {
  type: SignatureErrorType
  message: string
  code: string
  details?: any
  recoverable: boolean
  timestamp: number
  context?: Record<string, any>
}

class SignatureErrorHandler {
  private errorLog: SignatureError[] = []
  private maxLogSize = 1000
  private retryAttempts = new Map<string, number>()
  private maxRetries = 3

  /**
   * Handle and categorize signature verification errors
   */
  handleError(error: any, context?: Record<string, any>): SignatureError {
    const signatureError: SignatureError = {
      type: this.categorizeError(error),
      message: error instanceof Error ? error.message : String(error),
      code: this.generateErrorCode(error),
      recoverable: this.isRecoverable(error),
      timestamp: Date.now(),
      context
    }

    // Add additional details based on error type
    signatureError.details = this.extractErrorDetails(error, signatureError.type)

    // Log error
    this.logError(signatureError)

    return signatureError
  }

  /**
   * Attempt recovery from signature verification error
   */
  async attemptRecovery(
    error: SignatureError,
    originalOperation: () => Promise<any>,
    recoveryKey?: string
  ): Promise<{ success: boolean; result?: any; error?: SignatureError }> {
    if (!error.recoverable) {
      return { success: false, error }
    }

    const key = recoveryKey || `${error.type}-${error.code}`
    const attempts = this.retryAttempts.get(key) || 0

    if (attempts >= this.maxRetries) {
      return {
        success: false,
        error: {
          ...error,
          type: SignatureErrorType.UNKNOWN_ERROR,
          message: `Max retry attempts exceeded (${this.maxRetries})`,
          recoverable: false
        }
      }
    }

    try {
      // Increment retry counter
      this.retryAttempts.set(key, attempts + 1)

      // Apply recovery strategy based on error type
      await this.applyRecoveryStrategy(error)

      // Retry original operation
      const result = await originalOperation()

      // Reset retry counter on success
      this.retryAttempts.delete(key)

      return { success: true, result }
    } catch (retryError) {
      const newError = this.handleError(retryError, {
        ...error.context,
        retryAttempt: attempts + 1,
        originalError: error
      })

      return { success: false, error: newError }
    }
  }

  /**
   * Categorize error type for appropriate handling
   */
  private categorizeError(error: any): SignatureErrorType {
    const message = error instanceof Error ? error.message.toLowerCase() : String(error).toLowerCase()

    if (message.includes('invalid signature') || message.includes('signature verification failed')) {
      return SignatureErrorType.INVALID_SIGNATURE
    }
    if (message.includes('invalid public key') || message.includes('public key')) {
      return SignatureErrorType.INVALID_PUBLIC_KEY
    }
    if (message.includes('invalid message') || message.includes('message')) {
      return SignatureErrorType.INVALID_MESSAGE
    }
    if (message.includes('timeout') || message.includes('timed out')) {
      return SignatureErrorType.VERIFICATION_TIMEOUT
    }
    if (message.includes('cache') || message.includes('memory')) {
      return SignatureErrorType.CACHE_ERROR
    }
    if (message.includes('batch') || message.includes('parallel')) {
      return SignatureErrorType.BATCH_PROCESSING_ERROR
    }
    if (message.includes('threshold') || message.includes('minimum')) {
      return SignatureErrorType.THRESHOLD_NOT_MET
    }
    if (message.includes('dependency') || message.includes('depends')) {
      return SignatureErrorType.DEPENDENCY_ERROR
    }
    if (message.includes('time lock') || message.includes('lock time')) {
      return SignatureErrorType.TIME_LOCK_ERROR
    }
    if (message.includes('network') || message.includes('connection')) {
      return SignatureErrorType.NETWORK_ERROR
    }

    return SignatureErrorType.UNKNOWN_ERROR
  }

  /**
   * Determine if error is recoverable
   */
  private isRecoverable(error: any): boolean {
    const message = error instanceof Error ? error.message.toLowerCase() : String(error).toLowerCase()

    // Non-recoverable errors
    if (message.includes('invalid signature') || 
        message.includes('invalid public key') ||
        message.includes('invalid message')) {
      return false
    }

    // Recoverable errors
    if (message.includes('timeout') ||
        message.includes('network') ||
        message.includes('cache') ||
        message.includes('memory')) {
      return true
    }

    return false
  }

  /**
   * Apply recovery strategy based on error type
   */
  private async applyRecoveryStrategy(error: SignatureError): Promise<void> {
    switch (error.type) {
      case SignatureErrorType.VERIFICATION_TIMEOUT:
        // Wait before retry with exponential backoff
        const attempts = this.retryAttempts.get(`${error.type}-${error.code}`) || 0
        await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempts) * 1000))
        break

      case SignatureErrorType.CACHE_ERROR:
        // Clear cache and force garbage collection
        if (global.gc) {
          global.gc()
        }
        break

      case SignatureErrorType.NETWORK_ERROR:
        // Wait for network recovery
        await new Promise(resolve => setTimeout(resolve, 2000))
        break

      case SignatureErrorType.BATCH_PROCESSING_ERROR:
        // Reduce batch size or concurrency
        break

      default:
        // Generic recovery delay
        await new Promise(resolve => setTimeout(resolve, 500))
    }
  }

  /**
   * Extract additional error details
   */
  private extractErrorDetails(error: any, type: SignatureErrorType): any {
    const details: any = {}

    if (error instanceof Error) {
      details.stack = error.stack
      details.name = error.name
    }

    // Add type-specific details
    switch (type) {
      case SignatureErrorType.VERIFICATION_TIMEOUT:
        details.timeoutDuration = error.timeout || 'unknown'
        break
      case SignatureErrorType.BATCH_PROCESSING_ERROR:
        details.batchSize = error.batchSize || 'unknown'
        details.failedIndex = error.failedIndex || 'unknown'
        break
    }

    return details
  }

  /**
   * Generate unique error code
   */
  private generateErrorCode(error: any): string {
    const timestamp = Date.now().toString(36)
    const hash = this.simpleHash(error instanceof Error ? error.message : String(error))
    return `${timestamp}-${hash}`
  }

  /**
   * Simple hash function for error codes
   */
  private simpleHash(str: string): string {
    let hash = 0
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i)
      hash = ((hash << 5) - hash) + char
      hash = hash & hash // Convert to 32-bit integer
    }
    return Math.abs(hash).toString(36)
  }

  /**
   * Log error with size management
   */
  private logError(error: SignatureError): void {
    this.errorLog.push(error)

    // Maintain log size
    if (this.errorLog.length > this.maxLogSize) {
      this.errorLog = this.errorLog.slice(-this.maxLogSize)
    }
  }

  /**
   * Get error statistics
   */
  getErrorStats(): {
    totalErrors: number
    errorsByType: Record<SignatureErrorType, number>
    recoverableErrors: number
    recentErrors: SignatureError[]
  } {
    const errorsByType = {} as Record<SignatureErrorType, number>
    let recoverableErrors = 0

    for (const error of this.errorLog) {
      errorsByType[error.type] = (errorsByType[error.type] || 0) + 1
      if (error.recoverable) {
        recoverableErrors++
      }
    }

    return {
      totalErrors: this.errorLog.length,
      errorsByType,
      recoverableErrors,
      recentErrors: this.errorLog.slice(-10)
    }
  }

  /**
   * Clear error log and retry counters
   */
  reset(): void {
    this.errorLog = []
    this.retryAttempts.clear()
  }
}
```

## 7. Security Considerations

Complex signature verification introduces additional security considerations beyond basic signature validation:

### Security Best Practices

```typescript
class SecureSignatureVerifier {
  private readonly TIMING_ATTACK_DELAY = 10 // milliseconds
  private readonly MAX_VERIFICATION_TIME = 30000 // 30 seconds
  private readonly MAX_BATCH_SIZE = 100
  
  /**
   * Timing-attack resistant signature verification
   */
  async verifySecure(
    message: string | number[],
    signature: Signature,
    publicKey: PublicKey
  ): Promise<boolean> {
    const startTime = Date.now()
    
    try {
      // Input validation
      this.validateInputs(message, signature, publicKey)
      
      // Perform verification
      const isValid = signature.verify(message, publicKey)
      
      // Constant-time delay to prevent timing attacks
      await this.constantTimeDelay(startTime)
      
      return isValid
    } catch (error) {
      // Always delay on error to prevent timing attacks
      await this.constantTimeDelay(startTime)
      throw error
    }
  }

  /**
   * Secure batch verification with rate limiting
   */
  async verifyBatchSecure(
    items: Array<{
      message: string | number[]
      signature: Signature
      publicKey: PublicKey
    }>,
    options: {
      maxBatchSize?: number
      timeoutMs?: number
      rateLimitMs?: number
    } = {}
  ): Promise<boolean[]> {
    const { 
      maxBatchSize = this.MAX_BATCH_SIZE,
      timeoutMs = this.MAX_VERIFICATION_TIME,
      rateLimitMs = 100
    } = options

    // Validate batch size
    if (items.length > maxBatchSize) {
      throw new Error(`Batch size ${items.length} exceeds maximum ${maxBatchSize}`)
    }

    // Rate limiting
    if (rateLimitMs > 0) {
      await new Promise(resolve => setTimeout(resolve, rateLimitMs))
    }

    const startTime = Date.now()
    const results: boolean[] = []

    try {
      // Process with timeout
      const verificationPromise = Promise.all(
        items.map(async (item) => {
          this.validateInputs(item.message, item.signature, item.publicKey)
          return item.signature.verify(item.message, item.publicKey)
        })
      )

      const timeoutPromise = new Promise<never>((_, reject) => {
        setTimeout(() => reject(new Error('Batch verification timeout')), timeoutMs)
      })

      const batchResults = await Promise.race([verificationPromise, timeoutPromise])
      results.push(...batchResults)

      return results
    } catch (error) {
      // Ensure minimum processing time for security
      await this.constantTimeDelay(startTime)
      throw error
    }
  }

  /**
   * Validate inputs to prevent injection attacks
   */
  private validateInputs(
    message: string | number[],
    signature: Signature,
    publicKey: PublicKey
  ): void {
    // Validate message
    if (typeof message === 'string') {
      if (message.length > 1000000) { // 1MB limit
        throw new Error('Message too large')
      }
    } else if (Array.isArray(message)) {
      if (message.length > 1000000) {
        throw new Error('Message array too large')
      }
      // Validate array contains only numbers
      if (!message.every(item => typeof item === 'number' && Number.isInteger(item) && item >= 0 && item <= 255)) {
        throw new Error('Invalid message array format')
      }
    } else {
      throw new Error('Invalid message type')
    }

    // Validate signature format
    try {
      const derBytes = signature.toDER()
      if (derBytes.length < 6 || derBytes.length > 73) {
        throw new Error('Invalid signature length')
      }
    } catch (error) {
      throw new Error('Invalid signature format')
    }

    // Validate public key
    try {
      const pubKeyHex = publicKey.toString()
      if (pubKeyHex.length !== 66 && pubKeyHex.length !== 130) {
        throw new Error('Invalid public key length')
      }
    } catch (error) {
      throw new Error('Invalid public key format')
    }
  }

  /**
   * Constant-time delay to prevent timing attacks
   */
  private async constantTimeDelay(startTime: number): Promise<void> {
    const elapsed = Date.now() - startTime
    const minDelay = this.TIMING_ATTACK_DELAY
    
    if (elapsed < minDelay) {
      await new Promise(resolve => setTimeout(resolve, minDelay - elapsed))
    }
  }
}
```

## 8. Best Practices

### Production-Ready Implementation

```typescript
/**
 * Production-ready complex signature verifier combining all patterns
 */
class ProductionSignatureVerifier {
  private batchVerifier: AdvancedBatchVerifier
  private performanceVerifier: PerformanceOptimizedVerifier
  private errorHandler: SignatureErrorHandler
  private secureVerifier: SecureSignatureVerifier
  private multiContextVerifier: MultiContextSignatureVerifier

  constructor(walletClient?: any) {
    this.batchVerifier = new AdvancedBatchVerifier()
    this.performanceVerifier = new PerformanceOptimizedVerifier()
    this.errorHandler = new SignatureErrorHandler()
    this.secureVerifier = new SecureSignatureVerifier()
    this.multiContextVerifier = new MultiContextSignatureVerifier(walletClient)
  }

  /**
   * Comprehensive signature verification with all safety measures
   */
  async verifyComprehensive(
    message: string | number[],
    signature: Signature,
    publicKey: PublicKey,
    options: {
      useCache?: boolean
      secureMode?: boolean
      timeout?: number
    } = {}
  ): Promise<{
    valid: boolean
    verificationTime: number
    fromCache?: boolean
    securityChecks: boolean
    error?: SignatureError
  }> {
    const { useCache = true, secureMode = true, timeout = 10000 } = options
    const startTime = Date.now()

    try {
      let result: { valid: boolean; fromCache?: boolean; verificationTime: number }

      if (secureMode) {
        // Use secure verification
        const isValid = await this.secureVerifier.verifySecure(message, signature, publicKey)
        result = {
          valid: isValid,
          verificationTime: Date.now() - startTime
        }
      } else if (useCache) {
        // Use performance-optimized verification with caching
        result = await this.performanceVerifier.verifyOptimized(message, signature, publicKey)
      } else {
        // Basic verification
        const isValid = signature.verify(message, publicKey)
        result = {
          valid: isValid,
          verificationTime: Date.now() - startTime
        }
      }

      return {
        valid: result.valid,
        verificationTime: result.verificationTime,
        fromCache: result.fromCache,
        securityChecks: secureMode
      }

    } catch (error) {
      const signatureError = this.errorHandler.handleError(error, {
        message: typeof message === 'string' ? message.substring(0, 100) : 'binary',
        publicKey: publicKey.toString().substring(0, 20) + '...',
        secureMode,
        useCache
      })

      // Attempt recovery for recoverable errors
      if (signatureError.recoverable) {
        const recovery = await this.errorHandler.attemptRecovery(
          signatureError,
          () => this.verifyComprehensive(message, signature, publicKey, { ...options, useCache: false })
        )

        if (recovery.success) {
          return recovery.result
        }
      }

      return {
        valid: false,
        verificationTime: Date.now() - startTime,
        securityChecks: secureMode,
        error: signatureError
      }
    }
  }

  /**
   * Get comprehensive system status
   */
  getSystemStatus(): {
    performance: ReturnType<PerformanceOptimizedVerifier['getMetrics']>
    errors: ReturnType<SignatureErrorHandler['getErrorStats']>
    uptime: number
    memoryUsage?: NodeJS.MemoryUsage
  } {
    return {
      performance: this.performanceVerifier.getMetrics(),
      errors: this.errorHandler.getErrorStats(),
      uptime: process.uptime(),
      memoryUsage: process.memoryUsage?.()
    }
  }

  /**
   * Reset all internal state
   */
  reset(): void {
    this.batchVerifier.clearCache()
    this.performanceVerifier.reset()
    this.errorHandler.reset()
  }
}
```

### Usage Examples

```typescript
// Example 1: Basic batch verification
const batchVerifier = new BatchSignatureVerifier()
const items = [
  { message: 'Hello World', signature: sig1, publicKey: pub1, id: 'msg1' },
  { message: 'Test Message', signature: sig2, publicKey: pub2, id: 'msg2' }
]

const batchResult = await batchVerifier.verifyBatch(items)
console.log(`Batch verification: ${batchResult.allValid}`)
console.log(`Processing time: ${batchResult.processingTimeMs}ms`)

// Example 2: Performance-optimized verification
const perfVerifier = new PerformanceOptimizedVerifier()
const result = await perfVerifier.verifyOptimized('Hello', signature, publicKey)
console.log(`Valid: ${result.valid}, From cache: ${result.fromCache}`)

// Example 3: Production-ready verification
const prodVerifier = new ProductionSignatureVerifier()
const compResult = await prodVerifier.verifyComprehensive(
  'Important message',
  signature,
  publicKey,
  { secureMode: true, useCache: true }
)

if (compResult.valid) {
  console.log('Signature verified successfully')
} else if (compResult.error) {
  console.error('Verification failed:', compResult.error.message)
}

// Example 4: System monitoring
const status = prodVerifier.getSystemStatus()
console.log('Performance metrics:', status.performance)
console.log('Error statistics:', status.errors)
```

## Related Resources

- [Signature Concepts](../concepts/signatures.md) - Basic signature concepts
- [Advanced Transaction Signing](./advanced-transaction-signing.md) - SIGHASH types and manual signing
- [Multi-Signature Transactions](./multisig-transactions.md) - Multi-signature patterns
- [Error Handling Tutorial](../tutorials/error-handling.md) - Comprehensive error handling
- [Security Best Practices](./security-best-practices.md) - Security guidelines
- [Performance Optimization](../performance.md) - Performance tuning techniques

## Summary

This guide covered advanced signature verification patterns including:

- **Batch Processing**: Efficient verification of multiple signatures with caching and parallel processing
- **Threshold Signatures**: Validation of threshold signature schemes using polynomial interpolation
- **Multi-Context Validation**: Handling different signature types in unified workflows
- **Performance Optimization**: Caching, metrics, and parallel processing for high-throughput applications
- **Complex Scenarios**: Time-locked, conditional, and multi-party signature validation
- **Error Handling**: Comprehensive error categorization, recovery strategies, and logging
- **Security Considerations**: Timing attack prevention, input validation, and secure verification patterns
- **Production Patterns**: Complete implementation combining all techniques with monitoring and diagnostics

These patterns enable building robust, scalable signature verification systems that can handle complex real-world requirements while maintaining security and performance.
