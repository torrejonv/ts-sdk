# Error Handling and Edge Cases

This tutorial covers robust error handling patterns and edge case management when working with the BSV TypeScript SDK. You'll learn to build resilient applications that gracefully handle network failures, wallet issues, validation errors, and other common scenarios.

## Prerequisites

- Basic familiarity with the BSV TypeScript SDK
- Understanding of WalletClient usage
- Knowledge of async/await and Promise handling
- Basic TypeScript/JavaScript error handling concepts

## Learning Goals

By the end of this tutorial, you will:

- Understand common error types in the BSV ecosystem
- Implement robust retry mechanisms and recovery strategies
- Handle wallet-specific errors and edge cases
- Build production-ready error handling patterns
- Debug and troubleshoot common SDK issues
- Create user-friendly error reporting systems

## Error Types and Categories

### WalletErrorObject Interface

The SDK uses a standardized error interface for wallet operations:

```typescript
import { WalletClient, WalletErrorObject } from '@bsv/sdk'

function isWalletError(error: any): error is WalletErrorObject {
  return error && typeof error === 'object' && error.isError === true
}

async function handleWalletOperation() {
  try {
    const wallet = new WalletClient('auto', 'localhost')
    const result = await wallet.createAction({
      description: 'Test transaction',
      outputs: [{
        satoshis: 100,
        lockingScript: '006a0474657374', // "test"
        outputDescription: 'Test output'
      }]
    })
    
    console.log('Transaction successful:', result.txid)
    
  } catch (error) {
    if (isWalletError(error)) {
      console.error('Wallet error occurred:', error.message)
      // Handle wallet-specific error
    } else {
      console.error('General error:', error)
      // Handle other types of errors
    }
  }
}
```

### Common Error Categories

```typescript
enum ErrorCategory {
  NETWORK = 'network',
  WALLET = 'wallet', 
  VALIDATION = 'validation',
  AUTHENTICATION = 'authentication',
  INSUFFICIENT_FUNDS = 'insufficient_funds',
  SCRIPT = 'script',
  CRYPTOGRAPHIC = 'cryptographic'
}

interface CategorizedError {
  category: ErrorCategory
  message: string
  originalError: Error
  retryable: boolean
  suggestedAction?: string
}

function categorizeError(error: Error): CategorizedError {
  const message = error.message.toLowerCase()
  
  if (message.includes('insufficient funds')) {
    return {
      category: ErrorCategory.INSUFFICIENT_FUNDS,
      message: error.message,
      originalError: error,
      retryable: true,
      suggestedAction: 'Check wallet balance and available UTXOs'
    }
  }
  
  if (message.includes('rpc error') || message.includes('no header should have returned false')) {
    return {
      category: ErrorCategory.NETWORK,
      message: error.message,
      originalError: error,
      retryable: true,
      suggestedAction: 'Wait for wallet synchronization or restart wallet'
    }
  }
  
  if (message.includes('not authenticated') || message.includes('authentication')) {
    return {
      category: ErrorCategory.AUTHENTICATION,
      message: error.message,
      originalError: error,
      retryable: true,
      suggestedAction: 'Re-authenticate with wallet'
    }
  }
  
  if (message.includes('script') || message.includes('validation')) {
    return {
      category: ErrorCategory.VALIDATION,
      message: error.message,
      originalError: error,
      retryable: false,
      suggestedAction: 'Check transaction inputs and script validity'
    }
  }
  
  return {
    category: ErrorCategory.NETWORK,
    message: error.message,
    originalError: error,
    retryable: false
  }
}
```

## Retry Mechanisms and Recovery Strategies

### Exponential Backoff Implementation

```typescript
interface RetryOptions {
  maxAttempts: number
  baseDelay: number
  maxDelay: number
  backoffMultiplier: number
  retryableErrors: string[]
}

class RetryManager {
  private defaultOptions: RetryOptions = {
    maxAttempts: 3,
    baseDelay: 1000,
    maxDelay: 30000,
    backoffMultiplier: 2,
    retryableErrors: [
      'insufficient funds',
      'rpc error',
      'no header should have returned false',
      'network error',
      'timeout'
    ]
  }
  
  async executeWithRetry<T>(
    operation: () => Promise<T>,
    options: Partial<RetryOptions> = {}
  ): Promise<T> {
    const config = { ...this.defaultOptions, ...options }
    let lastError: Error
    
    for (let attempt = 1; attempt <= config.maxAttempts; attempt++) {
      try {
        console.log(`Attempt ${attempt}/${config.maxAttempts}`)
        return await operation()
        
      } catch (error: any) {
        lastError = error
        const categorized = categorizeError(error)
        
        console.log(`Attempt ${attempt} failed:`, categorized.message)
        
        if (!categorized.retryable || attempt === config.maxAttempts) {
          break
        }
        
        const isRetryableError = config.retryableErrors.some(
          retryableError => error.message.toLowerCase().includes(retryableError)
        )
        
        if (!isRetryableError) {
          console.log('Error is not retryable, stopping attempts')
          break
        }
        
        // Calculate delay with exponential backoff
        const delay = Math.min(
          config.baseDelay * Math.pow(config.backoffMultiplier, attempt - 1),
          config.maxDelay
        )
        
        console.log(`Waiting ${delay}ms before retry...`)
        await new Promise(resolve => setTimeout(resolve, delay))
      }
    }
    
    throw new Error(`Operation failed after ${config.maxAttempts} attempts: ${lastError.message}`)
  }
}

// Example usage
const retryManager = new RetryManager()

async function robustTransactionCreation() {
  const wallet = new WalletClient('auto', 'localhost')
  
  const result = await retryManager.executeWithRetry(async () => {
    return await wallet.createAction({
      description: 'Robust transaction with retry logic',
      outputs: [{
        satoshis: 100,
        lockingScript: '006a0e526f6275737420746573742074786e', // "Robust test txn"
        outputDescription: 'Retry demo output'
      }]
    })
  }, {
    maxAttempts: 5,
    baseDelay: 2000
  })
  
  console.log('Transaction successful:', result.txid)
}

// Example usage
const walletManager = new WalletManager()

async function demonstrateWalletErrorHandling() {
  try {
    const result = await walletManager.safeCreateAction({
      description: 'Wallet error handling demo',
      outputs: [{
        satoshis: 100,
        lockingScript: '006a0f57616c6c657420657272206465616c', // "Wallet err deal"
        outputDescription: 'Error handling demo'
      }]
    })
    
    console.log('Transaction created successfully:', result.txid)
    
  } catch (error) {
    console.error('Final error after all recovery attempts:', error.message)
  }
}

## Network and Chain Tracker Error Handling

### HTTP Client Error Management

```typescript
import { ChainTracker } from '@bsv/sdk'

class RobustChainTracker {
  private tracker: ChainTracker
  private retryManager: RetryManager
  
  constructor(baseURL: string = 'https://api.whatsonchain.com/v1/bsv/main') {
    this.tracker = new ChainTracker(baseURL)
    this.retryManager = new RetryManager()
  }
  
  async safeIsValidRootForHeight(root: string, height: number): Promise<boolean> {
    return this.retryManager.executeWithRetry(async () => {
      try {
        return await this.tracker.isValidRootForHeight(root, height)
        
      } catch (error: any) {
        if (error.message.includes('404')) {
          throw new Error(`Block height ${height} not found on chain`)
        }
        
        if (error.message.includes('429')) {
          console.log('Rate limited, waiting longer...')
          await new Promise(resolve => setTimeout(resolve, 5000))
          throw error // Will be retried
        }
        
        if (error.message.includes('timeout') || error.message.includes('ECONNRESET')) {
          console.log('Network timeout, retrying...')
          throw error // Will be retried
        }
        
        // For other HTTP errors, don't retry
        throw new Error(`Chain tracker error: ${error.message}`)
      }
    }, {
      maxAttempts: 5,
      baseDelay: 2000,
      retryableErrors: ['timeout', 'econnreset', '429', 'network error']
    })
  }
}

// Example with fallback chain trackers
class FallbackChainTracker {
  private trackers: RobustChainTracker[]
  private currentTrackerIndex: number = 0
  
  constructor() {
    this.trackers = [
      new RobustChainTracker('https://api.whatsonchain.com/v1/bsv/main'),
      new RobustChainTracker('https://api.taal.com/api/v1/bsv/main'),
      // Add more fallback endpoints as needed
    ]
  }
  
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    for (let i = 0; i < this.trackers.length; i++) {
      const trackerIndex = (this.currentTrackerIndex + i) % this.trackers.length
      const tracker = this.trackers[trackerIndex]
      
      try {
        console.log(`Trying chain tracker ${trackerIndex + 1}/${this.trackers.length}`)
        const result = await tracker.safeIsValidRootForHeight(root, height)
        
        // Success - update current tracker for next time
        this.currentTrackerIndex = trackerIndex
        return result
        
      } catch (error: any) {
        console.log(`Chain tracker ${trackerIndex + 1} failed:`, error.message)
        
        if (i === this.trackers.length - 1) {
          throw new Error(`All chain trackers failed. Last error: ${error.message}`)
        }
      }
    }
    
    throw new Error('No chain trackers available')
  }
}
```

### Practical Retry Implementation

Here's a practical example that handles common wallet errors with specific recovery strategies:

```typescript
import { WalletClient } from '@bsv/sdk'

async function robustTransactionCreation() {
  try {
    const wallet = new WalletClient('auto', 'localhost')
    
    const { authenticated } = await wallet.isAuthenticated()
    if (!authenticated) {
      await wallet.waitForAuthentication()
    }

    // Implement retry logic for common wallet issues
    async function createActionWithRetry(args: any, maxRetries = 3) {
      for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
          console.log(`Transaction attempt ${attempt}/${maxRetries}`)
          
          const result = await wallet.createAction(args)
          console.log('Transaction successful!')
          return result
          
        } catch (error: any) {
          console.log(`Attempt ${attempt} failed:`, error.message)
          
          if (error.message?.includes('Insufficient funds')) {
            console.log('Checking available UTXOs...')
            
            const { outputs } = await wallet.listOutputs({
              basket: 'tutorial',
              limit: 20
            })
            
            const spendableValue = outputs
              .filter(o => o.spendable)
              .reduce((sum, o) => sum + o.satoshis, 0)
            
            console.log(`Total spendable: ${spendableValue} satoshis`)
            
            if (spendableValue < 500) {
              throw new Error('Insufficient confirmed funds available')
            }
            
            // Try with a smaller amount
            if (args.outputs && args.outputs[0]) {
              args.outputs[0].satoshis = Math.min(100, spendableValue - 50)
              console.log(`Retrying with reduced amount: ${args.outputs[0].satoshis} satoshis`)
            }
            
          } else if (error.message?.includes('no header should have returned false')) {
            console.log('Wallet synchronization issue detected')
            console.log('Waiting for wallet to sync...')
            await new Promise(resolve => setTimeout(resolve, 5000))
            
          } else if (error.message?.includes('RPC Error')) {
            console.log('RPC communication error, retrying...')
            await new Promise(resolve => setTimeout(resolve, 2000))
            
          } else {
            // For other errors, don't retry
            throw error
          }
          
          if (attempt === maxRetries) {
            throw new Error(`Transaction failed after ${maxRetries} attempts: ${error.message}`)
          }
        }
      }
    }

    // Use the robust transaction creation
    const result = await createActionWithRetry({
      description: 'Robust transaction with error handling',
      outputs: [
        {
          satoshis: 100,
          lockingScript: '006a0e526f6275737420746573742074786e', // "Robust test txn"
          outputDescription: 'Error handling demo'
        }
      ]
    })

    if (result && result.txid) {
      console.log(`Robust transaction created: ${result.txid}`)
    } else {
      console.log('Transaction completed but no TXID returned')
    }

  } catch (error: unknown) {
    console.error('Final error:', error)
  }
}

robustTransactionCreation().catch(console.error)
```

This example demonstrates:
- **Specific error pattern matching** for different error types
- **Dynamic amount adjustment** for insufficient funds
- **Wallet synchronization handling** with appropriate delays
- **RPC error recovery** with retry logic
- **Graceful degradation** when all retries are exhausted

## SPV Verification Error Handling

```typescript
import { Transaction, MerklePath } from '@bsv/sdk'

class SPVVerificationManager {
  private chainTracker: FallbackChainTracker
  
  constructor() {
    this.chainTracker = new FallbackChainTracker()
  }
  
  async verifyTransactionSPV(
    transaction: Transaction,
    merklePath?: MerklePath,
    chainTracker?: ChainTracker
  ): Promise<{ verified: boolean; error?: string }> {
    try {
      // Use provided chain tracker or fallback
      const tracker = chainTracker || this.chainTracker
      
      const isValid = await transaction.verify(tracker, merklePath)
      
      return { verified: isValid }
      
    } catch (error: any) {
      const errorMessage = error.message.toLowerCase()
      
      if (errorMessage.includes('missing source transaction')) {
        return {
          verified: false,
          error: 'BEEF structure incomplete - missing input transactions'
        }
      }
      
      if (errorMessage.includes('merkle root')) {
        return {
          verified: false,
          error: 'Merkle proof verification failed - invalid proof or root mismatch'
        }
      }
      
      if (errorMessage.includes('script')) {
        return {
          verified: false,
          error: 'Script validation failed - invalid unlocking script'
        }
      }
      
      if (errorMessage.includes('chain tracker')) {
        return {
          verified: false,
          error: 'Chain tracker unavailable - cannot verify block headers'
        }
      }
      
      return {
        verified: false,
        error: `SPV verification failed: ${error.message}`
      }
    }
  }
  
  async batchVerifyTransactions(
    transactions: Transaction[],
    merkleProofs?: MerklePath[]
  ): Promise<Array<{ txid: string; verified: boolean; error?: string }>> {
    const results = []
    
    for (let i = 0; i < transactions.length; i++) {
      const tx = transactions[i]
      const proof = merkleProofs?.[i]
      const txid = Buffer.from(tx.id()).toString('hex')
      
      try {
        console.log(`Verifying transaction ${i + 1}/${transactions.length}: ${txid}`)
        
        const result = await this.verifyTransactionSPV(tx, proof)
        results.push({ txid, ...result })
        
        // Add small delay to avoid overwhelming the chain tracker
        if (i < transactions.length - 1) {
          await new Promise(resolve => setTimeout(resolve, 100))
        }
        
      } catch (error: any) {
        results.push({
          txid,
          verified: false,
          error: `Batch verification error: ${error.message}`
        })
      }
    }
    
    return results
  }
}
```

## Cryptographic Operation Error Handling

### Key Validation and Recovery

```typescript
import { PrivateKey, PublicKey, SymmetricKey } from '@bsv/sdk'

class CryptographicErrorHandler {
  
  static validatePrivateKey(key: PrivateKey): { valid: boolean; error?: string } {
    try {
      // Test key operations
      const publicKey = key.toPublicKey()
      const wif = key.toWif()
      
      // Verify the key can sign and verify
      const testMessage = 'validation test'
      const signature = key.sign(Buffer.from(testMessage, 'utf8'))
      const isValid = publicKey.verify(Buffer.from(testMessage, 'utf8'), signature)
      
      if (!isValid) {
        return { valid: false, error: 'Key failed signature verification test' }
      }
      
      return { valid: true }
      
    } catch (error: any) {
      return { valid: false, error: `Key validation failed: ${error.message}` }
    }
  }
  
  static validatePublicKey(key: PublicKey): { valid: boolean; error?: string } {
    try {
      // Test key operations
      const point = key.point
      const der = key.toDER()
      
      // Verify point is on curve
      if (!point.isOnCurve()) {
        return { valid: false, error: 'Public key point is not on the secp256k1 curve' }
      }
      
      return { valid: true }
      
    } catch (error: any) {
      return { valid: false, error: `Public key validation failed: ${error.message}` }
    }
  }
  
  static safeECDHKeyExchange(
    privateKey: PrivateKey,
    publicKey: PublicKey
  ): { success: boolean; sharedSecret?: number[]; error?: string } {
    try {
      // Validate inputs first
      const privateValidation = this.validatePrivateKey(privateKey)
      if (!privateValidation.valid) {
        return { success: false, error: `Invalid private key: ${privateValidation.error}` }
      }
      
      const publicValidation = this.validatePublicKey(publicKey)
      if (!publicValidation.valid) {
        return { success: false, error: `Invalid public key: ${publicValidation.error}` }
      }
      
      // Perform ECDH
      const sharedSecret = privateKey.deriveSharedSecret(publicKey)
      
      // Validate the result
      if (!sharedSecret || sharedSecret.length !== 32) {
        return { success: false, error: 'ECDH produced invalid shared secret' }
      }
      
      return { success: true, sharedSecret }
      
    } catch (error: any) {
      return { success: false, error: `ECDH key exchange failed: ${error.message}` }
    }
  }
  
  static safeSymmetricEncryption(
    data: string,
    key?: SymmetricKey
  ): { success: boolean; encrypted?: number[]; key?: SymmetricKey; error?: string } {
    try {
      const encryptionKey = key || SymmetricKey.fromRandom()
      
      if (!data || data.length === 0) {
        return { success: false, error: 'Data cannot be empty' }
      }
      
      const encrypted = encryptionKey.encrypt(data) as number[]
      
      if (!encrypted || encrypted.length === 0) {
        return { success: false, error: 'Encryption produced empty result' }
      }
      
      // Verify we can decrypt
      const decrypted = encryptionKey.decrypt(encrypted, 'utf8') as string
      if (decrypted !== data) {
        return { success: false, error: 'Encryption verification failed - decrypt mismatch' }
      }
      
      return { success: true, encrypted, key: encryptionKey }
      
    } catch (error: any) {
      return { success: false, error: `Symmetric encryption failed: ${error.message}` }
    }
  }
}

// Example usage with comprehensive error handling
async function demonstrateCryptographicErrorHandling() {
  console.log('=== Cryptographic Error Handling Demo ===')
  
  try {
    // Test private key validation
    const privateKey = PrivateKey.fromRandom()
    const validation = CryptographicErrorHandler.validatePrivateKey(privateKey)
    
    if (!validation.valid) {
      console.error('Private key validation failed:', validation.error)
      return
    }
    
    console.log('Private key validation: PASSED')
    
    // Test ECDH with error handling
    const otherPrivateKey = PrivateKey.fromRandom()
    const otherPublicKey = otherPrivateKey.toPublicKey()
    
    const ecdhResult = CryptographicErrorHandler.safeECDHKeyExchange(privateKey, otherPublicKey)
    
    if (!ecdhResult.success) {
      console.error('ECDH failed:', ecdhResult.error)
      return
    }
    
    console.log('ECDH key exchange: PASSED')
    
    // Test symmetric encryption with error handling
    const testData = 'Sensitive data for encryption testing'
    const encryptionResult = CryptographicErrorHandler.safeSymmetricEncryption(testData)
    
    if (!encryptionResult.success) {
      console.error('Symmetric encryption failed:', encryptionResult.error)
      return
    }
    
    console.log('Symmetric encryption: PASSED')
    console.log('All cryptographic operations completed successfully')
    
  } catch (error) {
    console.error('Unexpected error in cryptographic demo:', error)
  }
}
```

## Production Error Monitoring

### Comprehensive Error Logging

```typescript
interface ErrorLogEntry {
  timestamp: Date
  category: ErrorCategory
  operation: string
  message: string
  stack?: string
  context?: any
  retryCount?: number
  resolved?: boolean
}

class ErrorLogger {
  private logs: ErrorLogEntry[] = []
  private maxLogs: number = 1000
  
  log(
    category: ErrorCategory,
    operation: string,
    error: Error,
    context?: any,
    retryCount?: number
  ): void {
    const entry: ErrorLogEntry = {
      timestamp: new Date(),
      category,
      operation,
      message: error.message,
      stack: error.stack,
      context,
      retryCount,
      resolved: false
    }
    
    this.logs.push(entry)
    
    // Keep only recent logs
    if (this.logs.length > this.maxLogs) {
      this.logs = this.logs.slice(-this.maxLogs)
    }
    
    // Console output for development
    console.error(`[${category.toUpperCase()}] ${operation}: ${error.message}`)
    
    if (context) {
      console.error('Context:', context)
    }
  }
  
  markResolved(operation: string, timestamp: Date): void {
    const entry = this.logs.find(
      log => log.operation === operation && 
             Math.abs(log.timestamp.getTime() - timestamp.getTime()) < 1000
    )
    
    if (entry) {
      entry.resolved = true
    }
  }
  
  getErrorSummary(): { [key in ErrorCategory]: number } {
    const summary = {} as { [key in ErrorCategory]: number }
    
    for (const category of Object.values(ErrorCategory)) {
      summary[category] = this.logs.filter(
        log => log.category === category && !log.resolved
      ).length
    }
    
    return summary
  }
  
  getRecentErrors(minutes: number = 60): ErrorLogEntry[] {
    const cutoff = new Date(Date.now() - minutes * 60 * 1000)
    return this.logs.filter(log => log.timestamp > cutoff)
  }
  
  exportLogs(): string {
    return JSON.stringify(this.logs, null, 2)
  }
}

// Global error logger instance
const errorLogger = new ErrorLogger()
```

### User-Friendly Error Messages

```typescript
class UserErrorHandler {
  private errorLogger: ErrorLogger
  
  constructor() {
    this.errorLogger = errorLogger
  }
  
  getUserFriendlyMessage(error: Error, operation: string): string {
    const categorized = categorizeError(error)
    this.errorLogger.log(categorized.category, operation, error)
    
    switch (categorized.category) {
      case ErrorCategory.INSUFFICIENT_FUNDS:
        return 'Insufficient funds available. Please check your wallet balance and try again with a smaller amount.'
        
      case ErrorCategory.AUTHENTICATION:
        return 'Wallet authentication required. Please unlock your wallet and try again.'
        
      case ErrorCategory.NETWORK:
        if (error.message.includes('no header should have returned false')) {
          return 'Wallet is synchronizing with the network. Please wait a moment and try again.'
        }
        return 'Network connection issue. Please check your internet connection and try again.'
        
      case ErrorCategory.VALIDATION:
        return 'Transaction validation failed. Please check your transaction details and try again.'
        
      case ErrorCategory.SCRIPT:
        return 'Transaction script error. Please verify your transaction parameters.'
        
      case ErrorCategory.CRYPTOGRAPHIC:
        return 'Cryptographic operation failed. Please try again or contact support if the issue persists.'
        
      default:
        return 'An unexpected error occurred. Please try again or contact support if the issue persists.'
    }
  }
  
  async handleOperationWithUserFeedback<T>(
    operation: () => Promise<T>,
    operationName: string,
    onProgress?: (message: string) => void,
    onError?: (userMessage: string, technicalError: Error) => void
  ): Promise<T> {
    const retryManager = new RetryManager()
    
    try {
      return await retryManager.executeWithRetry(async () => {
        if (onProgress) {
          onProgress(`Executing ${operationName}...`)
        }
        
        return await operation()
        
      }, {
        maxAttempts: 3,
        baseDelay: 2000
      })
      
    } catch (error: any) {
      const userMessage = this.getUserFriendlyMessage(error, operationName)
      
      if (onError) {
        onError(userMessage, error)
      }
      
      throw new Error(userMessage)
    }
  }
}

// Example usage in a user interface
async function demonstrateUserErrorHandling() {
  const userErrorHandler = new UserErrorHandler()
  const wallet = new WalletClient('auto', 'localhost')
  
  try {
    await userErrorHandler.handleOperationWithUserFeedback(
      async () => {
        return await wallet.createAction({
          description: 'User-friendly error handling demo',
          outputs: [{
            satoshis: 100,
            lockingScript: '006a0f5573657220667269656e646c79', // "User friendly"
            outputDescription: 'User error demo'
          }]
        })
      },
      'Create Transaction',
      (message) => console.log('Progress:', message),
      (userMessage, technicalError) => {
        console.log('User Message:', userMessage)
        console.log('Technical Details:', technicalError.message)
      }
    )
    
    console.log('Transaction completed successfully!')
    
  } catch (error) {
    console.log('Operation failed with user-friendly error:', error.message)
  }
}
```

## Testing Error Scenarios

### Error Simulation for Testing

```typescript
class ErrorSimulator {
  private shouldSimulateError: boolean = false
  private errorType: ErrorCategory = ErrorCategory.NETWORK
  private errorMessage: string = 'Simulated error'
  
  enableErrorSimulation(type: ErrorCategory, message: string): void {
    this.shouldSimulateError = true
    this.errorType = type
    this.errorMessage = message
  }
  
  disableErrorSimulation(): void {
    this.shouldSimulateError = false
  }
  
  checkAndThrowSimulatedError(operation: string): void {
    if (this.shouldSimulateError) {
      console.log(`Simulating ${this.errorType} error for operation: ${operation}`)
      throw new Error(this.errorMessage)
    }
  }
  
  async simulateNetworkDelay(minMs: number = 100, maxMs: number = 1000): Promise<void> {
    const delay = Math.random() * (maxMs - minMs) + minMs
    await new Promise(resolve => setTimeout(resolve, delay))
  }
}

// Test suite for error handling
class ErrorHandlingTestSuite {
  private simulator: ErrorSimulator
  private userErrorHandler: UserErrorHandler
  
  constructor() {
    this.simulator = new ErrorSimulator()
    this.userErrorHandler = new UserErrorHandler()
  }
  
  async testInsufficientFundsHandling(): Promise<boolean> {
    console.log('Testing insufficient funds error handling...')
    
    this.simulator.enableErrorSimulation(
      ErrorCategory.INSUFFICIENT_FUNDS,
      'Insufficient funds: 101 more satoshis are needed'
    )
    
    try {
      await this.userErrorHandler.handleOperationWithUserFeedback(
        async () => {
          this.simulator.checkAndThrowSimulatedError('createAction')
          return { txid: 'simulated-success' }
        },
        'Test Transaction'
      )
      
      return false // Should have thrown
      
    } catch (error: any) {
      const isCorrectMessage = error.message.includes('Insufficient funds available')
      console.log('Insufficient funds test:', isCorrectMessage ? 'PASSED' : 'FAILED')
      return isCorrectMessage
      
    } finally {
      this.simulator.disableErrorSimulation()
    }
  }
  
  async testNetworkErrorHandling(): Promise<boolean> {
    console.log('Testing network error handling...')
    
    this.simulator.enableErrorSimulation(
      ErrorCategory.NETWORK,
      'RPC Error: no header should have returned false'
    )
    
    try {
      await this.userErrorHandler.handleOperationWithUserFeedback(
        async () => {
          this.simulator.checkAndThrowSimulatedError('createAction')
          return { txid: 'simulated-success' }
        },
        'Test Transaction'
      )
      
      return false // Should have thrown
      
    } catch (error: any) {
      const isCorrectMessage = error.message.includes('synchronizing with the network')
      console.log('Network error test:', isCorrectMessage ? 'PASSED' : 'FAILED')
      return isCorrectMessage
      
    } finally {
      this.simulator.disableErrorSimulation()
    }
  }
  
  async runAllTests(): Promise<void> {
    console.log('=== Error Handling Test Suite ===')
    
    const tests = [
      this.testInsufficientFundsHandling(),
      this.testNetworkErrorHandling()
    ]
    
    const results = await Promise.all(tests)
    const passedTests = results.filter(result => result).length
    
    console.log(`\nTest Results: ${passedTests}/${results.length} tests passed`)
    
    if (passedTests === results.length) {
      console.log('All error handling tests PASSED! ✅')
    } else {
      console.log('Some error handling tests FAILED! ❌')
    }
  }
}
```

## Best Practices Summary

### Production-Ready Error Handling Checklist

```typescript
// Production error handling implementation example
class ProductionWalletService {
  private wallet: WalletClient
  private retryManager: RetryManager
  private errorLogger: ErrorLogger
  private userErrorHandler: UserErrorHandler
  
  constructor() {
    this.wallet = new WalletClient('auto', 'localhost')
    this.retryManager = new RetryManager()
    this.errorLogger = new ErrorLogger()
    this.userErrorHandler = new UserErrorHandler()
  }
  
  async createTransaction(
    description: string,
    outputs: any[],
    options?: any
  ): Promise<{ success: boolean; txid?: string; userMessage?: string }> {
    const operation = 'createTransaction'
    
    try {
      const result = await this.userErrorHandler.handleOperationWithUserFeedback(
        async () => {
          return await this.retryManager.executeWithRetry(async () => {
            // Ensure authentication
            const { authenticated } = await this.wallet.isAuthenticated()
            if (!authenticated) {
              await this.wallet.waitForAuthentication()
            }
            
            // Create transaction
            return await this.wallet.createAction({
              description,
              outputs,
              ...options
            })
          })
        },
        operation
      )
      
      return {
        success: true,
        txid: result.txid
      }
      
    } catch (error: any) {
      const userMessage = this.userErrorHandler.getUserFriendlyMessage(error, operation)
      
      return {
        success: false,
        userMessage
      }
    }
  }
  
  getHealthStatus(): {
    status: 'healthy' | 'degraded' | 'unhealthy'
    errors: { [key in ErrorCategory]: number }
    uptime: number
  } {
    const errors = this.errorLogger.getErrorSummary()
    const totalErrors = Object.values(errors).reduce((sum, count) => sum + count, 0)
    
    let status: 'healthy' | 'degraded' | 'unhealthy'
    
    if (totalErrors === 0) {
      status = 'healthy'
    } else if (totalErrors < 5) {
      status = 'degraded'
    } else {
      status = 'unhealthy'
    }
    
    return {
      status,
      errors,
      uptime: process.uptime()
    }
  }
}
```

## Troubleshooting Common Issues

### Quick Diagnostic Guide

```typescript
async function diagnoseCommonIssues(): Promise<void> {
  console.log('=== BSV SDK Diagnostic Tool ===')
  
  const wallet = new WalletClient('auto', 'localhost')
  
  // Test 1: Wallet Connection
  try {
    console.log('1. Testing wallet connection...')
    const { authenticated } = await wallet.isAuthenticated()
    console.log(`   Wallet authenticated: ${authenticated}`)
    
    if (!authenticated) {
      console.log('   ⚠️  Wallet not authenticated - this may cause transaction failures')
    } else {
      console.log('   ✅ Wallet connection OK')
    }
  } catch (error) {
    console.log(`   ❌ Wallet connection failed: ${error.message}`)
  }
  
  // Test 2: UTXO Availability
  try {
    console.log('2. Checking UTXO availability...')
    const { outputs, totalValue } = await wallet.listOutputs({ limit: 10 })
    const spendable = outputs.filter(o => o.spendable)
    
    console.log(`   Total outputs: ${outputs.length}`)
    console.log(`   Spendable outputs: ${spendable.length}`)
    console.log(`   Total value: ${totalValue} satoshis`)
    
    if (spendable.length === 0) {
      console.log('   ⚠️  No spendable UTXOs available')
    } else if (totalValue < 1000) {
      console.log('   ⚠️  Low balance - may cause insufficient funds errors')
    } else {
      console.log('   ✅ UTXO availability OK')
    }
  } catch (error) {
    console.log(`   ❌ UTXO check failed: ${error.message}`)
  }
  
  // Test 3: Simple Transaction
  try {
    console.log('3. Testing simple transaction creation...')
    const result = await wallet.createAction({
      description: 'Diagnostic test transaction',
      outputs: [{
        satoshis: 100,
        lockingScript: '006a0a4469616720746573742074786e', // "Diag test txn"
        outputDescription: 'Diagnostic test'
      }]
    })
    
    console.log(`   ✅ Transaction created successfully: ${result.txid}`)
    
  } catch (error: any) {
    console.log(`   ❌ Transaction creation failed: ${error.message}`)
    
    if (error.message.includes('Insufficient funds')) {
      console.log('   💡 Try reducing the transaction amount or funding your wallet')
    } else if (error.message.includes('no header should have returned false')) {
      console.log('   💡 Try restarting your wallet or waiting for synchronization')
    } else if (error.message.includes('not authenticated')) {
      console.log('   💡 Ensure your wallet is unlocked and authenticated')
    }
  }
  
  console.log('\n=== Diagnostic Complete ===')
}
```

## Conclusion

You've now mastered comprehensive error handling for the BSV TypeScript SDK. You can:

- Categorize and handle different types of errors appropriately
- Implement robust retry mechanisms with exponential backoff
- Handle wallet-specific errors and authentication issues
- Manage network failures and chain tracker problems
- Validate cryptographic operations and handle edge cases
- Build production-ready error monitoring and logging systems
- Create user-friendly error messages and recovery strategies
- Test error scenarios systematically

These patterns enable you to build resilient Bitcoin applications that gracefully handle failures and provide excellent user experiences even when things go wrong.

## Next Steps

- Review the [Large Transactions Guide](../guides/large-transactions.md) for efficiency patterns
- Check out [Security Best Practices](../guides/security-best-practices.md) for comprehensive security
- Explore specific error handling in other tutorials for your use case

## Additional Resources

- [WalletClient API Reference](../reference/wallet.md)
- [BSV Blockchain Documentation](https://docs.bsvblockchain.org/)
- [Error Handling Best Practices](https://docs.bsvblockchain.org/error-handling)
