# Implementing Transaction Monitoring

This guide provides comprehensive patterns for monitoring Bitcoin transactions using the BSV TypeScript SDK, from basic status checking to advanced real-time monitoring systems.

## Table of Contents

1. [Transaction Status Fundamentals](#transaction-status-fundamentals)
2. [Real-Time Transaction Tracking](#real-time-transaction-tracking)
3. [Confirmation Monitoring](#confirmation-monitoring)
4. [Double-Spend Detection](#double-spend-detection)

## Transaction Status Fundamentals

Understanding transaction lifecycle states and monitoring requirements.

### Transaction Lifecycle States

Bitcoin transactions progress through several states from creation to final confirmation:

```typescript
// Transaction states in the Bitcoin network
type TransactionState = 
  | 'created'      // Transaction built but not broadcast
  | 'broadcast'    // Sent to network but not in mempool
  | 'mempool'      // Accepted into mempool, awaiting mining
  | 'confirmed'    // Included in a block
  | 'mature'       // Has sufficient confirmations
  | 'orphaned'     // In an orphaned block chain
  | 'double-spent' // Conflicting transaction confirmed

interface TransactionStatus {
  txid: string
  state: TransactionState
  blockHeight?: number
  blockHash?: string
  confirmations: number
  timestamp?: number
  fee?: number
  size: number
}
```

### Monitoring Requirements

Different applications have varying monitoring needs:

```typescript
interface MonitoringRequirements {
  // Confirmation thresholds
  minConfirmations: number
  maxConfirmations?: number
  
  // Timing requirements
  timeoutMs: number
  pollingIntervalMs: number
  
  // Detection capabilities
  detectDoubleSpends: boolean
  trackMempoolStatus: boolean
  monitorFeeRates: boolean
  
  // Notification preferences
  realTimeUpdates: boolean
  batchNotifications: boolean
  errorAlerts: boolean
}

// Example configurations for different use cases
const paymentMonitoring: MonitoringRequirements = {
  minConfirmations: 1,
  maxConfirmations: 6,
  timeoutMs: 600000, // 10 minutes
  pollingIntervalMs: 30000, // 30 seconds
  detectDoubleSpends: true,
  trackMempoolStatus: true,
  monitorFeeRates: false,
  realTimeUpdates: true,
  batchNotifications: false,
  errorAlerts: true
}

const dataStorageMonitoring: MonitoringRequirements = {
  minConfirmations: 1,
  timeoutMs: 300000, // 5 minutes
  pollingIntervalMs: 60000, // 1 minute
  detectDoubleSpends: false,
  trackMempoolStatus: false,
  monitorFeeRates: false,
  realTimeUpdates: false,
  batchNotifications: true,
  errorAlerts: true
}

const highValueMonitoring: MonitoringRequirements = {
  minConfirmations: 6,
  maxConfirmations: 12,
  timeoutMs: 7200000, // 2 hours
  pollingIntervalMs: 15000, // 15 seconds
  detectDoubleSpends: true,
  trackMempoolStatus: true,
  monitorFeeRates: true,
  realTimeUpdates: true,
  batchNotifications: false,
  errorAlerts: true
}
```

### Chain Tracker Integration

The SDK provides chain tracking capabilities for transaction monitoring:

```typescript
import { WhatsOnChain, defaultChainTracker } from '@bsv/sdk'

// Using default chain tracker
const chainTracker = defaultChainTracker()

// Using WhatsOnChain with custom configuration
const customTracker = new WhatsOnChain('main', {
  apiKey: process.env.WHATSONCHAIN_API_KEY,
  httpClient: customHttpClient
})

// Basic chain tracker operations
async function demonstrateChainTracker() {
  try {
    // Get current blockchain height
    const currentHeight = await chainTracker.currentHeight()
    console.log('Current block height:', currentHeight)
    
    // Validate block header for specific height
    const blockHash = 'your-block-hash-here'
    const height = 700000
    const isValid = await chainTracker.isValidRootForHeight(blockHash, height)
    console.log(`Block ${blockHash} valid for height ${height}:`, isValid)
    
  } catch (error) {
    console.error('Chain tracker error:', error)
  }
}
```

## Real-Time Transaction Tracking

Implementing continuous monitoring with polling and event-driven patterns.

### Event-Driven Transaction Monitor

Advanced monitoring system with event emissions and real-time updates:

```typescript
import { EventEmitter } from 'events'

interface TransactionEvent {
  txid: string
  event: 'found' | 'confirmed' | 'mature' | 'timeout' | 'error'
  data?: any
  timestamp: number
}

class RealTimeTransactionMonitor extends EventEmitter {
  private readonly network: 'main' | 'test'
  private readonly baseUrl: string
  private readonly monitoredTransactions = new Map<string, {
    requirements: MonitoringRequirements
    lastStatus?: BasicTransactionInfo
    startTime: number
    intervalId?: NodeJS.Timeout
  }>()

  constructor(network: 'main' | 'test' = 'main') {
    super()
    this.network = network
    this.baseUrl = network === 'test' 
      ? 'https://api.whatsonchain.com/v1/bsv/test'
      : 'https://api.whatsonchain.com/v1/bsv/main'
  }

  startMonitoring(txid: string, requirements: MonitoringRequirements): void {
    if (this.monitoredTransactions.has(txid)) {
      throw new Error(`Transaction ${txid} is already being monitored`)
    }

    const monitoringData = {
      requirements,
      startTime: Date.now(),
      intervalId: undefined
    }

    this.monitoredTransactions.set(txid, monitoringData)

    // Start polling
    const intervalId = setInterval(async () => {
      await this.checkTransaction(txid)
    }, requirements.pollingIntervalMs)

    monitoringData.intervalId = intervalId

    // Initial check
    this.checkTransaction(txid).catch(error => {
      this.emit('error', { txid, error, timestamp: Date.now() })
    })

    console.log(`Started monitoring transaction ${txid}`)
  }

  stopMonitoring(txid: string): void {
    const monitoringData = this.monitoredTransactions.get(txid)
    if (!monitoringData) {
      return
    }

    if (monitoringData.intervalId) {
      clearInterval(monitoringData.intervalId)
    }

    this.monitoredTransactions.delete(txid)
    console.log(`Stopped monitoring transaction ${txid}`)
  }

  private async checkTransaction(txid: string): Promise<void> {
    const monitoringData = this.monitoredTransactions.get(txid)
    if (!monitoringData) {
      return
    }

    const { requirements, lastStatus, startTime } = monitoringData

    try {
      // Check for timeout
      if (Date.now() - startTime > requirements.timeoutMs) {
        this.emit('timeout', { txid, timestamp: Date.now() })
        this.stopMonitoring(txid)
        return
      }

      const currentStatus = await this.getTransactionStatus(txid)

      // Transaction not found yet
      if (!currentStatus) {
        return
      }

      // First time finding the transaction
      if (!lastStatus) {
        this.emit('found', { 
          txid, 
          data: currentStatus, 
          timestamp: Date.now() 
        })
        monitoringData.lastStatus = currentStatus
      }

      // Check for confirmation changes
      if (lastStatus && currentStatus.confirmations !== lastStatus.confirmations) {
        if (currentStatus.confirmations >= requirements.minConfirmations) {
          this.emit('confirmed', { 
            txid, 
            data: currentStatus, 
            timestamp: Date.now() 
          })
        }

        if (requirements.maxConfirmations && 
            currentStatus.confirmations >= requirements.maxConfirmations) {
          this.emit('mature', { 
            txid, 
            data: currentStatus, 
            timestamp: Date.now() 
          })
          
          // Stop monitoring if we've reached max confirmations
          this.stopMonitoring(txid)
        }

        monitoringData.lastStatus = currentStatus
      }

    } catch (error) {
      this.emit('error', { 
        txid, 
        error, 
        timestamp: Date.now() 
      })
    }
  }

  private async getTransactionStatus(txid: string): Promise<BasicTransactionInfo | null> {
    const maxRetries = 3
    const baseDelay = 1000 // 1 second
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const controller = new AbortController()
        const timeoutId = setTimeout(() => controller.abort(), 10000) // 10 second timeout
        
        const response = await fetch(`${this.baseUrl}/tx/${txid}`, {
          signal: controller.signal
        })
        
        clearTimeout(timeoutId)
        
        if (response.status === 404) {
          return null
        }
        
        // Handle specific API errors with retry logic
        if (response.status === 502 || response.status === 503 || response.status === 504) {
          if (attempt < maxRetries) {
            const delay = baseDelay * Math.pow(2, attempt - 1) // Exponential backoff
            console.warn(`API error ${response.status} for ${txid}, retrying in ${delay}ms (attempt ${attempt}/${maxRetries})`)
            await new Promise(resolve => setTimeout(resolve, delay))
            continue
          }
          throw new Error(`API error after ${maxRetries} attempts: ${response.status} ${response.statusText}`)
        }
        
        if (!response.ok) {
          throw new Error(`API error: ${response.status} ${response.statusText}`)
        }
        
        const data = await response.json()
        
        return {
          txid: data.txid,
          confirmed: data.confirmations > 0,
          blockHeight: data.blockheight,
          confirmations: data.confirmations || 0,
          timestamp: data.time
        }
        
      } catch (error: any) {
        // Handle network timeouts and connection errors
        if (error.name === 'AbortError' || error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT') {
          if (attempt < maxRetries) {
            const delay = baseDelay * Math.pow(2, attempt - 1)
            console.warn(`Network error for ${txid}, retrying in ${delay}ms (attempt ${attempt}/${maxRetries}): ${error.message}`)
            await new Promise(resolve => setTimeout(resolve, delay))
            continue
          }
          throw new Error(`Network error after ${maxRetries} attempts: ${error.message}`)
        }
        
        // Re-throw other errors immediately
        throw error
      }
    }
    
    throw new Error(`Failed to get transaction status after ${maxRetries} attempts`)
  }

  getMonitoredTransactions(): string[] {
    return Array.from(this.monitoredTransactions.keys())
  }

  stopAllMonitoring(): void {
    for (const txid of this.monitoredTransactions.keys()) {
      this.stopMonitoring(txid)
    }
  }
}
```

### Usage Examples

```typescript
async function realTimeMonitoringExample() {
  const monitor = new RealTimeTransactionMonitor('test')

  // Set up event listeners
  monitor.on('found', (event: TransactionEvent) => {
    console.log(`✅ Transaction ${event.txid} found on network`)
    console.log('Status:', event.data)
  })

  monitor.on('confirmed', (event: TransactionEvent) => {
    console.log(`🎉 Transaction ${event.txid} confirmed!`)
    console.log(`Confirmations: ${event.data.confirmations}`)
  })

  monitor.on('mature', (event: TransactionEvent) => {
    console.log(`🔒 Transaction ${event.txid} is mature`)
    console.log(`Final confirmations: ${event.data.confirmations}`)
  })

  monitor.on('timeout', (event: TransactionEvent) => {
    console.log(`⏰ Transaction ${event.txid} monitoring timed out`)
  })

  monitor.on('error', (event: TransactionEvent) => {
    console.error(`❌ Error monitoring ${event.txid}:`, event.error)
  })

  // Start monitoring transactions
  const txid1 = 'your-transaction-id-1'
  const txid2 = 'your-transaction-id-2'

  monitor.startMonitoring(txid1, paymentMonitoring)
  monitor.startMonitoring(txid2, highValueMonitoring)

  // Monitor for 10 minutes, then cleanup
  setTimeout(() => {
    console.log('Stopping all monitoring...')
    monitor.stopAllMonitoring()
  }, 600000)
}
```

### Multi-Transaction Coordinator

Coordinating monitoring of related transactions:

```typescript
class TransactionCoordinator {
  private readonly monitor: RealTimeTransactionMonitor
  private readonly transactionGroups = new Map<string, {
    txids: string[]
    requirements: MonitoringRequirements
    completedCount: number
    callback?: (results: Map<string, BasicTransactionInfo>) => void
  }>()

  constructor(network: 'main' | 'test' = 'main') {
    this.monitor = new RealTimeTransactionMonitor(network)
    this.setupEventHandlers()
  }

  private setupEventHandlers(): void {
    this.monitor.on('confirmed', (event: TransactionEvent) => {
      this.handleTransactionConfirmed(event.txid, event.data)
    })

    this.monitor.on('mature', (event: TransactionEvent) => {
      this.handleTransactionMature(event.txid, event.data)
    })

    this.monitor.on('error', (event: any) => {
      console.error(`Transaction ${event.txid} error:`, event.error)
    })

    this.monitor.on('timeout', (event: TransactionEvent) => {
      console.warn(`Transaction ${event.txid} timed out`)
    })
  }

  monitorTransactionGroup(
    groupId: string,
    txids: string[],
    requirements: MonitoringRequirements,
    callback?: (results: Map<string, BasicTransactionInfo>) => void
  ): void {
    this.transactionGroups.set(groupId, {
      txids,
      requirements,
      completedCount: 0,
      callback
    })

    // Start monitoring each transaction
    for (const txid of txids) {
      this.monitor.startMonitoring(txid, requirements)
    }

    console.log(`Started monitoring group ${groupId} with ${txids.length} transactions`)
  }

  private handleTransactionConfirmed(txid: string, data: BasicTransactionInfo): void {
    // Find which group this transaction belongs to
    for (const [groupId, group] of this.transactionGroups.entries()) {
      if (group.txids.includes(txid)) {
        group.completedCount++
        
        console.log(`Group ${groupId}: ${group.completedCount}/${group.txids.length} transactions confirmed`)
        
        // Check if all transactions in group are complete
        if (group.completedCount === group.txids.length) {
          this.handleGroupComplete(groupId)
        }
        break
      }
    }
  }

  private handleTransactionMature(txid: string, data: BasicTransactionInfo): void {
    // Similar logic for mature transactions
    this.handleTransactionConfirmed(txid, data)
  }

  private async handleGroupComplete(groupId: string): Promise<void> {
    const group = this.transactionGroups.get(groupId)
    if (!group || !group.callback) {
      return
    }

    // Collect final status for all transactions
    const results = new Map<string, BasicTransactionInfo>()
    
    for (const txid of group.txids) {
      try {
        const status = await this.getTransactionStatus(txid)
        if (status) {
          results.set(txid, status)
        }
      } catch (error) {
        console.error(`Error getting final status for ${txid}:`, error)
      }
    }

    // Call the completion callback
    group.callback(results)
    
    // Cleanup
    this.transactionGroups.delete(groupId)
    console.log(`Group ${groupId} completed and cleaned up`)
  }

  private async getTransactionStatus(txid: string): Promise<BasicTransactionInfo | null> {
    // Implementation similar to BasicTransactionMonitor with robust error handling
    const baseUrl = this.monitor['baseUrl']
    const maxRetries = 3
    const baseDelay = 1000
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const controller = new AbortController()
        const timeoutId = setTimeout(() => controller.abort(), 10000)
        
        const response = await fetch(`${baseUrl}/tx/${txid}`, {
          signal: controller.signal
        })
        
        clearTimeout(timeoutId)
        
        if (response.status === 404) {
          return null
        }
        
        // Handle gateway errors with retry
        if (response.status === 502 || response.status === 503 || response.status === 504) {
          if (attempt < maxRetries) {
            const delay = baseDelay * Math.pow(2, attempt - 1)
            console.warn(`API error ${response.status} for ${txid}, retrying in ${delay}ms (attempt ${attempt}/${maxRetries})`)
            await new Promise(resolve => setTimeout(resolve, delay))
            continue
          }
          throw new Error(`API error after ${maxRetries} attempts: ${response.status}`)
        }
        
        if (!response.ok) {
          throw new Error(`API error: ${response.status}`)
        }
        
        const data = await response.json()
        
        return {
          txid: data.txid,
          confirmed: data.confirmations > 0,
          blockHeight: data.blockheight,
          confirmations: data.confirmations || 0,
          timestamp: data.time
        }
        
      } catch (error: any) {
        if (error.name === 'AbortError' || error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT') {
          if (attempt < maxRetries) {
            const delay = baseDelay * Math.pow(2, attempt - 1)
            console.warn(`Network error for ${txid}, retrying in ${delay}ms (attempt ${attempt}/${maxRetries}): ${error.message}`)
            await new Promise(resolve => setTimeout(resolve, delay))
            continue
          }
          throw new Error(`Network error after ${maxRetries} attempts: ${error.message}`)
        }
        throw error
      }
    }
    
    throw new Error(`Failed to get transaction status after ${maxRetries} attempts`)
  }

  stopAllMonitoring(): void {
    this.monitor.stopAllMonitoring()
    this.transactionGroups.clear()
  }
}
```

### Coordinator Usage Example

```typescript
async function coordinatorExample() {
  const coordinator = new TransactionCoordinator('test')

  // Monitor a batch of related transactions
  const batchTxids = [
    'tx-id-1',
    'tx-id-2', 
    'tx-id-3'
  ]

  coordinator.monitorTransactionGroup(
    'payment-batch-1',
    batchTxids,
    paymentMonitoring,
    (results) => {
      console.log('Batch completed! Results:')
      for (const [txid, status] of results.entries()) {
        console.log(`${txid}: ${status.confirmations} confirmations`)
      }
    }
  )

  // Cleanup after 30 minutes
  setTimeout(() => {
    coordinator.stopAllMonitoring()
  }, 1800000)
}
```

## Confirmation Monitoring

Tracking transaction confirmations and block inclusion.

### Confirmation Tracker

Specialized monitoring for confirmation thresholds and block depth:

```typescript
interface ConfirmationEvent {
  txid: string
  confirmations: number
  blockHeight: number
  blockHash?: string
  timestamp: number
  isReorganization?: boolean
}

class ConfirmationTracker extends EventEmitter {
  private readonly network: 'main' | 'test'
  private readonly baseUrl: string
  private readonly chainTracker: ChainTracker
  private readonly trackedTransactions = new Map<string, {
    targetConfirmations: number
    lastConfirmations: number
    lastBlockHeight?: number
    callback?: (event: ConfirmationEvent) => void
  }>()

  constructor(network: 'main' | 'test' = 'main', chainTracker?: ChainTracker) {
    super()
    this.network = network
    this.baseUrl = network === 'test' 
      ? 'https://api.whatsonchain.com/v1/bsv/test'
      : 'https://api.whatsonchain.com/v1/bsv/main'
    this.chainTracker = chainTracker || defaultChainTracker()
  }

  async trackConfirmations(
    txid: string,
    targetConfirmations: number,
    callback?: (event: ConfirmationEvent) => void
  ): Promise<void> {
    if (this.trackedTransactions.has(txid)) {
      throw new Error(`Transaction ${txid} is already being tracked`)
    }

    this.trackedTransactions.set(txid, {
      targetConfirmations,
      lastConfirmations: 0,
      callback
    })

    // Start monitoring
    await this.checkConfirmations(txid)
    
    console.log(`Started tracking confirmations for ${txid} (target: ${targetConfirmations})`)
  }

  private async checkConfirmations(txid: string): Promise<void> {
    const trackingData = this.trackedTransactions.get(txid)
    if (!trackingData) {
      return
    }

    try {
      const status = await this.getTransactionStatus(txid)
      if (!status) {
        // Transaction not found yet, continue monitoring
        setTimeout(() => this.checkConfirmations(txid), 30000)
        return
      }

      const currentConfirmations = status.confirmations
      const { lastConfirmations, lastBlockHeight, targetConfirmations, callback } = trackingData

      // Check for reorganization
      let isReorganization = false
      if (lastBlockHeight && status.blockHeight && status.blockHeight !== lastBlockHeight) {
        if (currentConfirmations < lastConfirmations) {
          isReorganization = true
          console.warn(`Potential reorganization detected for ${txid}`)
        }
      }

      // Update tracking data
      trackingData.lastConfirmations = currentConfirmations
      trackingData.lastBlockHeight = status.blockHeight

      // Create confirmation event
      const event: ConfirmationEvent = {
        txid,
        confirmations: currentConfirmations,
        blockHeight: status.blockHeight || 0,
        timestamp: Date.now(),
        isReorganization
      }

      // Emit events for confirmation milestones
      if (currentConfirmations !== lastConfirmations) {
        this.emit('confirmation-change', event)
        
        if (callback) {
          callback(event)
        }
      }

      // Check if target reached
      if (currentConfirmations >= targetConfirmations) {
        this.emit('target-reached', event)
        this.trackedTransactions.delete(txid)
        console.log(`Target confirmations reached for ${txid}: ${currentConfirmations}/${targetConfirmations}`)
        return
      }

      // Continue monitoring
      setTimeout(() => this.checkConfirmations(txid), 60000) // Check every minute

    } catch (error) {
      this.emit('error', { txid, error, timestamp: Date.now() })
      // Retry after error
      setTimeout(() => this.checkConfirmations(txid), 120000) // Retry in 2 minutes
    }
  }

  private async getTransactionStatus(txid: string): Promise<BasicTransactionInfo | null> {
    const maxRetries = 3
    const baseDelay = 1000
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const controller = new AbortController()
        const timeoutId = setTimeout(() => controller.abort(), 10000)
        
        const response = await fetch(`${this.baseUrl}/tx/${txid}`, {
          signal: controller.signal
        })
        
        clearTimeout(timeoutId)
        
        if (response.status === 404) {
          return null
        }
        
        // Handle gateway errors with retry
        if (response.status === 502 || response.status === 503 || response.status === 504) {
          if (attempt < maxRetries) {
            const delay = baseDelay * Math.pow(2, attempt - 1)
            console.warn(`API error ${response.status} for ${txid}, retrying in ${delay}ms (attempt ${attempt}/${maxRetries})`)
            await new Promise(resolve => setTimeout(resolve, delay))
            continue
          }
          throw new Error(`API error after ${maxRetries} attempts: ${response.status}`)
        }
        
        if (!response.ok) {
          throw new Error(`API error: ${response.status}`)
        }
        
        const data = await response.json()
        
        return {
          txid: data.txid,
          confirmed: data.confirmations > 0,
          blockHeight: data.blockheight,
          confirmations: data.confirmations || 0,
          timestamp: data.time
        }
        
      } catch (error: any) {
        if (error.name === 'AbortError' || error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT') {
          if (attempt < maxRetries) {
            const delay = baseDelay * Math.pow(2, attempt - 1)
            console.warn(`Network error for ${txid}, retrying in ${delay}ms (attempt ${attempt}/${maxRetries}): ${error.message}`)
            await new Promise(resolve => setTimeout(resolve, delay))
            continue
          }
          throw new Error(`Network error after ${maxRetries} attempts: ${error.message}`)
        }
        throw error
      }
    }
    
    throw new Error(`Failed to get transaction status after ${maxRetries} attempts`)
  }

  stopTracking(txid: string): void {
    this.trackedTransactions.delete(txid)
    console.log(`Stopped tracking confirmations for ${txid}`)
  }

  getTrackedTransactions(): string[] {
    return Array.from(this.trackedTransactions.keys())
  }

  stopAllTracking(): void {
    this.trackedTransactions.clear()
  }
}
```

### Advanced Confirmation Monitoring

Enhanced monitoring with reorganization detection and chain validation:

```typescript
class AdvancedConfirmationMonitor {
  private readonly confirmationTracker: ConfirmationTracker
  private readonly chainTracker: ChainTracker
  private readonly network: 'main' | 'test'

  constructor(network: 'main' | 'test' = 'main') {
    this.network = network
    this.chainTracker = defaultChainTracker()
    this.confirmationTracker = new ConfirmationTracker(network, this.chainTracker)
    this.setupEventHandlers()
  }

  private setupEventHandlers(): void {
    this.confirmationTracker.on('confirmation-change', async (event: ConfirmationEvent) => {
      await this.validateConfirmation(event)
    })

    this.confirmationTracker.on('target-reached', (event: ConfirmationEvent) => {
      console.log(`✅ Transaction ${event.txid} reached target confirmations: ${event.confirmations}`)
    })

    this.confirmationTracker.on('error', (event: any) => {
      console.error(`❌ Confirmation tracking error for ${event.txid}:`, event.error)
    })
  }

  private async validateConfirmation(event: ConfirmationEvent): Promise<void> {
    if (!event.blockHash) {
      return
    }

    try {
      // Validate the block header
      const isValidBlock = await this.chainTracker.isValidRootForHeight(
        event.blockHash, 
        event.blockHeight
      )

      if (!isValidBlock) {
        console.warn(`⚠️ Invalid block detected for transaction ${event.txid}`)
        console.warn(`Block ${event.blockHash} at height ${event.blockHeight} is not valid`)
        
        // Emit reorganization warning
        this.confirmationTracker.emit('reorganization-detected', {
          ...event,
          isReorganization: true
        })
      } else {
        console.log(`✅ Block validation passed for ${event.txid} at height ${event.blockHeight}`)
      }

    } catch (error) {
      console.error(`Error validating block for transaction ${event.txid}:`, error)
    }
  }

  async monitorWithValidation(
    txid: string, 
    targetConfirmations: number,
    options: {
      validateBlocks?: boolean
      reorganizationCallback?: (event: ConfirmationEvent) => void
      progressCallback?: (event: ConfirmationEvent) => void
    } = {}
  ): Promise<ConfirmationEvent> {
    return new Promise((resolve, reject) => {
      // Set up reorganization handler
      if (options.reorganizationCallback) {
        this.confirmationTracker.on('reorganization-detected', options.reorganizationCallback)
      }

      // Set up progress handler
      if (options.progressCallback) {
        this.confirmationTracker.on('confirmation-change', (event: ConfirmationEvent) => {
          if (event.txid === txid) {
            options.progressCallback!(event)
          }
        })
      }

      // Set up completion handler
      const targetHandler = (event: ConfirmationEvent) => {
        if (event.txid === txid) {
          this.confirmationTracker.off('target-reached', targetHandler)
          resolve(event)
        }
      }

      this.confirmationTracker.on('target-reached', targetHandler)

      // Set up error handler
      const errorHandler = (event: any) => {
        if (event.txid === txid) {
          this.confirmationTracker.off('error', errorHandler)
          reject(event.error)
        }
      }

      this.confirmationTracker.on('error', errorHandler)

      // Start tracking
      this.confirmationTracker.trackConfirmations(txid, targetConfirmations)
        .catch(reject)
    })
  }

  stopAllMonitoring(): void {
    this.confirmationTracker.stopAllTracking()
  }
}
```

### Usage Examples

```typescript
async function confirmationMonitoringExamples() {
  const monitor = new AdvancedConfirmationMonitor('test')

  const txid = 'your-transaction-id-here'

  try {
    // Example 1: Basic confirmation monitoring
    console.log('Starting confirmation monitoring...')
    
    const result = await monitor.monitorWithValidation(txid, 6, {
      validateBlocks: true,
      progressCallback: (event) => {
        console.log(`Progress: ${event.confirmations}/6 confirmations`)
        if (event.isReorganization) {
          console.warn('⚠️ Reorganization detected!')
        }
      },
      reorganizationCallback: (event) => {
        console.error('🚨 Block reorganization detected:', event)
        // Handle reorganization - maybe increase required confirmations
      }
    })

    console.log('✅ Transaction fully confirmed:', result)

  } catch (error) {
    console.error('❌ Confirmation monitoring failed:', error)
  } finally {
    monitor.stopAllMonitoring()
  }
}

// Example 2: Multiple transaction confirmation tracking
async function batchConfirmationTracking() {
  const tracker = new ConfirmationTracker('test')
  
  const transactions = [
    { txid: 'tx-1', target: 3 },
    { txid: 'tx-2', target: 6 },
    { txid: 'tx-3', target: 1 }
  ]

  const completedTransactions = new Set<string>()

  tracker.on('target-reached', (event: ConfirmationEvent) => {
    completedTransactions.add(event.txid)
    console.log(`✅ ${event.txid} confirmed with ${event.confirmations} confirmations`)
    
    if (completedTransactions.size === transactions.length) {
      console.log('🎉 All transactions confirmed!')
      tracker.stopAllTracking()
    }
  })

  tracker.on('confirmation-change', (event: ConfirmationEvent) => {
    console.log(`📊 ${event.txid}: ${event.confirmations} confirmations`)
  })

  // Start tracking all transactions
  for (const tx of transactions) {
    await tracker.trackConfirmations(tx.txid, tx.target)
  }
}
```

### Integration with Transaction Creation

Complete workflow from creation to confirmation:

```typescript
async function createAndMonitorTransaction() {
  const wallet = new WalletClient('auto', 'localhost')
  const monitor = new AdvancedConfirmationMonitor('test')

  try {
    // Create transaction
    const result = await wallet.createAction({
      description: 'Payment with confirmation monitoring',
      outputs: [{
        satoshis: 100,
        lockingScript: '76a914' + 'recipient-pubkey-hash' + '88ac',
        outputDescription: 'Payment output'
      }]
    })

    console.log(`Transaction created: ${result.txid}`)

    // Monitor confirmations
    const confirmationResult = await monitor.monitorWithValidation(result.txid, 3, {
      progressCallback: (event) => {
        console.log(`Confirmations: ${event.confirmations}/3`)
      }
    })

    console.log('Payment confirmed:', confirmationResult)
    return confirmationResult

  } catch (error) {
    console.error('Transaction or monitoring failed:', error)
    throw error
  } finally {
    monitor.stopAllMonitoring()
  }
}
```

## Double-Spend Detection

Detecting and handling potential double-spend scenarios.

### Double-Spend Monitor

Advanced monitoring system for detecting conflicting transactions:

```typescript
interface DoubleSpendEvent {
  originalTxid: string
  conflictingTxid: string
  conflictType: 'input-conflict' | 'rbf' | 'chain-reorganization'
  detectedAt: number
  blockHeight?: number
  confidence: 'low' | 'medium' | 'high'
}

interface TransactionInput {
  txid: string
  vout: number
  scriptSig?: string
  sequence?: number
}

class DoubleSpendDetector extends EventEmitter {
  private readonly network: 'main' | 'test'
  private readonly baseUrl: string
  private readonly monitoredInputs = new Map<string, {
    txid: string
    inputs: TransactionInput[]
    confirmations: number
    lastCheck: number
  }>()
  private readonly conflictHistory = new Map<string, DoubleSpendEvent[]>()

  constructor(network: 'main' | 'test' = 'main') {
    super()
    this.network = network
    this.baseUrl = network === 'test' 
      ? 'https://api.whatsonchain.com/v1/bsv/test'
      : 'https://api.whatsonchain.com/v1/bsv/main'
  }

  async monitorTransaction(txid: string): Promise<void> {
    if (this.monitoredInputs.has(txid)) {
      throw new Error(`Transaction ${txid} is already being monitored`)
    }

    try {
      // Get transaction details
      const txData = await this.getTransactionDetails(txid)
      if (!txData) {
        throw new Error(`Transaction ${txid} not found`)
      }

      // Extract inputs
      const inputs: TransactionInput[] = txData.vin.map((input: any) => ({
        txid: input.txid,
        vout: input.vout,
        scriptSig: input.scriptSig?.hex,
        sequence: input.sequence
      }))

      this.monitoredInputs.set(txid, {
        txid,
        inputs,
        confirmations: txData.confirmations || 0,
        lastCheck: Date.now()
      })

      // Start monitoring for conflicts
      this.checkForConflicts(txid)
      
      console.log(`Started double-spend monitoring for ${txid}`)

    } catch (error) {
      throw new Error(`Failed to start monitoring ${txid}: ${error.message}`)
    }
  }

  private async checkForConflicts(txid: string): Promise<void> {
    const monitorData = this.monitoredInputs.get(txid)
    if (!monitorData) {
      return
    }

    try {
      // Check each input for conflicts
      for (const input of monitorData.inputs) {
        await this.checkInputConflicts(txid, input)
      }

      // Update last check time
      monitorData.lastCheck = Date.now()

      // Continue monitoring if transaction is not deeply confirmed
      if (monitorData.confirmations < 6) {
        setTimeout(() => this.checkForConflicts(txid), 30000) // Check every 30 seconds
      } else {
        console.log(`Transaction ${txid} is deeply confirmed, stopping double-spend monitoring`)
        this.stopMonitoring(txid)
      }

    } catch (error) {
      this.emit('error', { txid, error, timestamp: Date.now() })
      // Retry after error
      setTimeout(() => this.checkForConflicts(txid), 60000)
    }
  }

  private async checkInputConflicts(originalTxid: string, input: TransactionInput): Promise<void> {
    try {
      // Get all transactions spending this input
      const spendingTxs = await this.getTransactionsSpendingOutput(input.txid, input.vout)
      
      // Filter out the original transaction
      const conflictingTxs = spendingTxs.filter(tx => tx.txid !== originalTxid)
      
      for (const conflictingTx of conflictingTxs) {
        await this.analyzeConflict(originalTxid, conflictingTx, input)
      }

    } catch (error) {
      console.error(`Error checking input conflicts for ${originalTxid}:`, error)
    }
  }

  private async analyzeConflict(
    originalTxid: string, 
    conflictingTx: any, 
    input: TransactionInput
  ): Promise<void> {
    const conflictingTxid = conflictingTx.txid
    
    // Determine conflict type and confidence
    let conflictType: DoubleSpendEvent['conflictType'] = 'input-conflict'
    let confidence: DoubleSpendEvent['confidence'] = 'medium'

    // Check if it's a Replace-by-Fee (RBF) scenario
    if (this.isRBFConflict(conflictingTx)) {
      conflictType = 'rbf'
      confidence = 'high'
    }

    // Check confirmation status
    const originalData = this.monitoredInputs.get(originalTxid)
    const conflictingConfirmations = conflictingTx.confirmations || 0
    const originalConfirmations = originalData?.confirmations || 0

    // Adjust confidence based on confirmations
    if (conflictingConfirmations > originalConfirmations) {
      confidence = 'high'
    } else if (conflictingConfirmations === 0 && originalConfirmations === 0) {
      confidence = 'low'
    }

    const doubleSpendEvent: DoubleSpendEvent = {
      originalTxid,
      conflictingTxid,
      conflictType,
      detectedAt: Date.now(),
      blockHeight: conflictingTx.blockheight,
      confidence
    }

    // Record the conflict
    this.recordConflict(originalTxid, doubleSpendEvent)

    // Emit the event
    this.emit('double-spend-detected', doubleSpendEvent)

    console.warn(`🚨 Double-spend detected!`)
    console.warn(`Original: ${originalTxid}`)
    console.warn(`Conflicting: ${conflictingTxid}`)
    console.warn(`Type: ${conflictType}, Confidence: ${confidence}`)
  }

  private isRBFConflict(tx: any): boolean {
    // Check if transaction signals RBF (sequence number < 0xfffffffe)
    return tx.vin.some((input: any) => input.sequence < 0xfffffffe)
  }

  private recordConflict(txid: string, event: DoubleSpendEvent): void {
    if (!this.conflictHistory.has(txid)) {
      this.conflictHistory.set(txid, [])
    }
    this.conflictHistory.get(txid)!.push(event)
  }

  private async getTransactionDetails(txid: string): Promise<any> {
    const response = await fetch(`${this.baseUrl}/tx/${txid}`)
    
    if (response.status === 404) {
      return null
    }
    
    if (!response.ok) {
      throw new Error(`API error: ${response.status}`)
    }
    
    return await response.json()
  }

  private async getTransactionsSpendingOutput(txid: string, vout: number): Promise<any[]> {
    try {
      // This is a simplified implementation - in practice, you'd need to:
      // 1. Query mempool for unconfirmed transactions
      // 2. Check blockchain for confirmed transactions
      // 3. Use specialized APIs that track UTXO spending
      
      const response = await fetch(`${this.baseUrl}/tx/${txid}/out/${vout}/spent`)
      
      if (response.status === 404) {
        return [] // Output not spent yet
      }
      
      if (!response.ok) {
        throw new Error(`API error: ${response.status}`)
      }
      
      const spentData = await response.json()
      return [spentData] // WhatsOnChain returns single spending transaction
      
    } catch (error) {
      console.error(`Error getting spending transactions:`, error)
      return []
    }
  }

  stopMonitoring(txid: string): void {
    this.monitoredInputs.delete(txid)
    console.log(`Stopped double-spend monitoring for ${txid}`)
  }

  getConflictHistory(txid: string): DoubleSpendEvent[] {
    return this.conflictHistory.get(txid) || []
  }

  getMonitoredTransactions(): string[] {
    return Array.from(this.monitoredInputs.keys())
  }

  stopAllMonitoring(): void {
    this.monitoredInputs.clear()
  }
}
```

### Enhanced Double-Spend Protection

Advanced protection with mempool monitoring and risk assessment:

```typescript
interface RiskAssessment {
  txid: string
  riskLevel: 'low' | 'medium' | 'high' | 'critical'
  factors: string[]
  recommendation: 'accept' | 'wait' | 'reject'
  requiredConfirmations: number
}

class DoubleSpendProtector {
  private readonly detector: DoubleSpendDetector
  private readonly riskThresholds = {
    lowValue: 1000,      // satoshis
    mediumValue: 100000, // satoshis
    highValue: 1000000   // satoshis
  }

  constructor(network: 'main' | 'test' = 'main') {
    this.detector = new DoubleSpendDetector(network)
    this.setupEventHandlers()
  }

  private setupEventHandlers(): void {
    this.detector.on('double-spend-detected', (event: DoubleSpendEvent) => {
      this.handleDoubleSpendDetection(event)
    })

    this.detector.on('error', (event: any) => {
      console.error(`Double-spend detection error:`, event.error)
    })
  }

  async assessTransactionRisk(txid: string, value: number): Promise<RiskAssessment> {
    const riskFactors: string[] = []
    let riskLevel: RiskAssessment['riskLevel'] = 'low'
    let requiredConfirmations = 1

    try {
      // Get transaction details
      const txData = await this.getTransactionDetails(txid)
      if (!txData) {
        throw new Error('Transaction not found')
      }

      // Factor 1: Transaction value
      if (value > this.riskThresholds.highValue) {
        riskFactors.push('High transaction value')
        riskLevel = 'high'
        requiredConfirmations = Math.max(requiredConfirmations, 6)
      } else if (value > this.riskThresholds.mediumValue) {
        riskFactors.push('Medium transaction value')
        riskLevel = 'medium'
        requiredConfirmations = Math.max(requiredConfirmations, 3)
      }

      // Factor 2: RBF signaling
      const hasRBF = txData.vin.some((input: any) => input.sequence < 0xfffffffe)
      if (hasRBF) {
        riskFactors.push('Transaction signals Replace-by-Fee (RBF)')
        riskLevel = this.escalateRisk(riskLevel)
        requiredConfirmations = Math.max(requiredConfirmations, 3)
      }

      // Factor 3: Low fee rate
      const feeRate = this.calculateFeeRate(txData)
      if (feeRate < 1) { // Less than 1 sat/byte
        riskFactors.push('Low fee rate - may be vulnerable to fee bumping')
        riskLevel = this.escalateRisk(riskLevel)
        requiredConfirmations = Math.max(requiredConfirmations, 2)
      }

      // Factor 4: Unconfirmed inputs
      const hasUnconfirmedInputs = await this.hasUnconfirmedInputs(txData)
      if (hasUnconfirmedInputs) {
        riskFactors.push('Transaction has unconfirmed inputs')
        riskLevel = this.escalateRisk(riskLevel)
        requiredConfirmations = Math.max(requiredConfirmations, 2)
      }

      // Factor 5: Multiple outputs to same address (potential change splitting)
      const outputAddresses = new Set(txData.vout.map((out: any) => out.scriptPubKey.addresses?.[0]).filter(Boolean))
      if (outputAddresses.size < txData.vout.length) {
        riskFactors.push('Multiple outputs to same address detected')
        riskLevel = this.escalateRisk(riskLevel)
      }

      // Determine recommendation
      let recommendation: RiskAssessment['recommendation'] = 'accept'
      if (riskLevel === 'critical') {
        recommendation = 'reject'
      } else if (riskLevel === 'high' || (riskLevel === 'medium' && value > this.riskThresholds.mediumValue)) {
        recommendation = 'wait'
      }

      return {
        txid,
        riskLevel,
        factors: riskFactors,
        recommendation,
        requiredConfirmations
      }

    } catch (error) {
      return {
        txid,
        riskLevel: 'critical',
        factors: [`Error assessing transaction: ${error.message}`],
        recommendation: 'reject',
        requiredConfirmations: 6
      }
    }
  }

  private escalateRisk(currentRisk: RiskAssessment['riskLevel']): RiskAssessment['riskLevel'] {
    const riskLevels = ['low', 'medium', 'high', 'critical']
    const currentIndex = riskLevels.indexOf(currentRisk)
    const nextIndex = Math.min(currentIndex + 1, riskLevels.length - 1)
    return riskLevels[nextIndex] as RiskAssessment['riskLevel']
  }

  private calculateFeeRate(txData: any): number {
    // Simplified fee rate calculation
    const inputValue = txData.vin.reduce((sum: number, input: any) => sum + (input.value || 0), 0)
    const outputValue = txData.vout.reduce((sum: number, output: any) => sum + output.value, 0)
    const fee = inputValue - outputValue
    const size = txData.size || 250 // Estimate if not available
    return fee / size
  }

  private async hasUnconfirmedInputs(txData: any): Promise<boolean> {
    for (const input of txData.vin) {
      try {
        const inputTx = await this.getTransactionDetails(input.txid)
        if (inputTx && inputTx.confirmations === 0) {
          return true
        }
      } catch (error) {
        // If we can't check, assume it's risky
        return true
      }
    }
    return false
  }

  private async getTransactionDetails(txid: string): Promise<any> {
    // Reuse the detector's method
    return await this.detector['getTransactionDetails'](txid)
  }

  private handleDoubleSpendDetection(event: DoubleSpendEvent): void {
    console.error(`🚨 DOUBLE-SPEND ALERT 🚨`)
    console.error(`Original Transaction: ${event.originalTxid}`)
    console.error(`Conflicting Transaction: ${event.conflictingTxid}`)
    console.error(`Conflict Type: ${event.conflictType}`)
    console.error(`Confidence: ${event.confidence}`)
    
    // Implement additional alerting mechanisms here
    // - Send notifications
    // - Log to security systems
    // - Trigger automatic responses
  }

  async monitorWithProtection(txid: string, value: number): Promise<RiskAssessment> {
    // First assess the risk
    const riskAssessment = await this.assessTransactionRisk(txid, value)
    
    // Start monitoring based on risk level
    if (riskAssessment.recommendation !== 'reject') {
      await this.detector.monitorTransaction(txid)
    }

    return riskAssessment
  }

  stopAllMonitoring(): void {
    this.detector.stopAllMonitoring()
  }
}
```

### Usage Examples

```typescript
async function doubleSpendDetectionExamples() {
  const protector = new DoubleSpendProtector('test')

  // Example 1: Risk assessment
  const txid = 'your-transaction-id-here'
  const transactionValue = 50000 // satoshis

  try {
    const riskAssessment = await protector.assessTransactionRisk(txid, transactionValue)
    
    console.log('Risk Assessment:')
    console.log(`- Risk Level: ${riskAssessment.riskLevel}`)
    console.log(`- Recommendation: ${riskAssessment.recommendation}`)
    console.log(`- Required Confirmations: ${riskAssessment.requiredConfirmations}`)
    console.log(`- Risk Factors:`)
    riskAssessment.factors.forEach(factor => console.log(`  • ${factor}`))

    // Act based on recommendation
    switch (riskAssessment.recommendation) {
      case 'accept':
        console.log('✅ Transaction accepted - low risk')
        break
      case 'wait':
        console.log('⏳ Waiting for confirmations before accepting')
        break
      case 'reject':
        console.log('❌ Transaction rejected - too risky')
        return
    }

  } catch (error) {
    console.error('Risk assessment failed:', error)
  }
}

// Example 2: Complete protection workflow
async function protectedTransactionProcessing() {
  const protector = new DoubleSpendProtector('test')
  
  // Set up double-spend alert handler
  protector['detector'].on('double-spend-detected', (event: DoubleSpendEvent) => {
    console.error('🚨 SECURITY ALERT: Double-spend detected!')
    console.error(`Affected transaction: ${event.originalTxid}`)
    console.error(`Conflicting transaction: ${event.conflictingTxid}`)
    
    // Implement your security response here
    // - Freeze related accounts
    // - Send alerts to administrators
    // - Log security incident
  })

  const transactions = [
    { txid: 'tx-1', value: 10000 },
    { txid: 'tx-2', value: 100000 },
    { txid: 'tx-3', value: 1000000 }
  ]

  for (const tx of transactions) {
    try {
      const assessment = await protector.monitorWithProtection(tx.txid, tx.value)
      
      console.log(`Transaction ${tx.txid}:`)
      console.log(`  Risk: ${assessment.riskLevel}`)
      console.log(`  Action: ${assessment.recommendation}`)
      
      if (assessment.recommendation === 'wait') {
        console.log(`  Waiting for ${assessment.requiredConfirmations} confirmations...`)
      }
      
    } catch (error) {
      console.error(`Failed to process transaction ${tx.txid}:`, error)
    }
  }

  // Cleanup after processing
  setTimeout(() => {
    protector.stopAllMonitoring()
  }, 600000) // Stop after 10 minutes
}
```

### Integration with Payment Processing

Complete integration with payment workflows:

```typescript
async function securePaymentProcessing(paymentTxid: string, paymentAmount: number) {
  const protector = new DoubleSpendProtector('main')
  
  try {
    // Step 1: Assess transaction risk
    const riskAssessment = await protector.assessTransactionRisk(paymentTxid, paymentAmount)
    
    console.log(`Payment risk assessment: ${riskAssessment.riskLevel}`)
    
    // Step 2: Handle based on risk level
    switch (riskAssessment.recommendation) {
      case 'reject':
        throw new Error('Payment rejected due to high double-spend risk')
        
      case 'wait':
        console.log(`Waiting for ${riskAssessment.requiredConfirmations} confirmations...`)
        
        // Monitor for double-spends while waiting
        await protector.monitorWithProtection(paymentTxid, paymentAmount)
        
        // Wait for required confirmations
        const confirmationMonitor = new AdvancedConfirmationMonitor('main')
        await confirmationMonitor.monitorWithValidation(paymentTxid, riskAssessment.requiredConfirmations)
        
        console.log('✅ Payment confirmed and secure')
        break
        
      case 'accept':
        console.log('✅ Payment accepted - low risk detected')
        // Still monitor for a short period as a precaution
        await protector.monitorWithProtection(paymentTxid, paymentAmount)
        break
    }
    
    return { success: true, riskLevel: riskAssessment.riskLevel }
    
  } catch (error) {
    console.error('Payment processing failed:', error)
    return { success: false, error: error.message }
  } finally {
    protector.stopAllMonitoring()
  }
}
```

## Related Resources

### SDK Documentation

- [Transaction Broadcasting](../tutorials/transaction-broadcasting.md) - Broadcasting transactions and basic monitoring patterns
- [Chain Tracking](./chain-tracking.md) - Using ChainTracker for blockchain state monitoring
- [Error Handling](../tutorials/error-handling.md) - Comprehensive error handling patterns

### API References

- [Wallet Reference](../reference/wallet.md) - Wallet integration and transaction creation
- [Network Configuration](../reference/network-config.md) - External API integration for transaction data

### External Resources

- [WhatsOnChain API Documentation](https://developers.whatsonchain.com/) - External API for transaction monitoring
- [BSV Network Documentation](https://docs.bsvblockchain.org/) - Understanding BSV transaction mechanics
- [Bitcoin Script Reference](https://wiki.bitcoinsv.io/index.php/Script) - Script validation and verification

### Advanced Topics

- [Transaction Batching](./transaction-batching.md) - Efficient batch transaction processing
- [ARC Configuration](../reference/arc-config.md) - Advanced transaction broadcasting configuration
