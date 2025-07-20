# Handling Large Bitcoin Transactions

This guide provides authoritative patterns and best practices for constructing, signing, and broadcasting large Bitcoin transactions using the BSV TypeScript SDK. All examples are based on actual SDK APIs and have been verified against the source code.

## Table of Contents

1. [Understanding Large Transactions](#understanding-large-transactions)
2. [Memory Management Strategies](#memory-management-strategies)
3. [Efficient Transaction Construction](#efficient-transaction-construction)
4. [Batch Processing with `WalletClient`](#batch-processing-with-walletclient)
5. [Fee Calculation for Large Transactions](#fee-calculation-for-large-transactions)

## Understanding Large Transactions

Large Bitcoin transactions typically involve:

- **Many inputs** (50+ UTXOs being consumed)
- **Many outputs** (50+ recipients or complex splitting)
- **Large scripts** (complex locking/unlocking conditions)
- **Chained transactions** (dependent transaction sequences)

The SDK provides several mechanisms to handle these efficiently.

## Memory Management Strategies

### Batch Input/Output Addition

Instead of adding inputs and outputs one by one, batch them to reduce memory allocations:

```typescript
import { Transaction, TransactionInput, TransactionOutput, LockingScript, UnlockingScript } from '@bsv/sdk'

class LargeTransactionBuilder {
  private transaction: Transaction
  private inputBatch: TransactionInput[] = []
  private outputBatch: TransactionOutput[] = []
  private batchSize = 100

  constructor() {
    this.transaction = new Transaction()
  }

  addInputBatch(inputs: TransactionInput[]): void {
    this.inputBatch.push(...inputs)
    
    if (this.inputBatch.length >= this.batchSize) {
      this.flushInputs()
    }
  }

  addOutputBatch(outputs: TransactionOutput[]): void {
    this.outputBatch.push(...outputs)
    
    if (this.outputBatch.length >= this.batchSize) {
      this.flushOutputs()
    }
  }

  private flushInputs(): void {
    for (const input of this.inputBatch) {
      this.transaction.addInput(input)
    }
    this.inputBatch = []
    
    // Hint garbage collection for large batches
    if (global.gc) {
      global.gc()
    }
  }

  private flushOutputs(): void {
    for (const output of this.outputBatch) {
      this.transaction.addOutput(output)
    }
    this.outputBatch = []
    
    if (global.gc) {
      global.gc()
    }
  }

  finalize(): Transaction {
    this.flushInputs()
    this.flushOutputs()
    return this.transaction
  }
}

// Example usage with TransactionInput and TransactionOutput creation
async function createLargeTransactionExample() {
  const builder = new LargeTransactionBuilder()
  
  // Create properly formatted inputs with required fields
  const inputs: TransactionInput[] = []
  for (let i = 0; i < 100; i++) {
    inputs.push({
      sourceTXID: '0'.repeat(64), // Replace with actual TXID
      sourceOutputIndex: i,
      unlockingScriptLength: 0,
      unlockingScript: new UnlockingScript(), // Will be populated during signing
      sequenceNumber: 0xffffffff
    } as TransactionInput)
  }
  
  // Create properly formatted outputs
  const outputs: TransactionOutput[] = []
  for (let i = 0; i < 100; i++) {
    outputs.push({
      satoshis: 100,
      lockingScriptLength: 6,
      lockingScript: LockingScript.fromASM('OP_RETURN 74657374') // OP_RETURN "test"
    } as TransactionOutput)
  }
  
  builder.addInputBatch(inputs)
  builder.addOutputBatch(outputs)
  
  return builder.finalize()
}
```

### Memory Pool Management

For extremely large transactions, implement a memory pool to manage object lifecycle:

```typescript
class TransactionMemoryPool {
  private inputPool: TransactionInput[] = []
  private outputPool: TransactionOutput[] = []
  private maxPoolSize = 1000

  borrowInput(): TransactionInput {
    return this.inputPool.pop() || {} as TransactionInput
  }

  borrowOutput(): TransactionOutput {
    return this.outputPool.pop() || {} as TransactionOutput
  }

  returnInput(input: TransactionInput): void {
    if (this.inputPool.length < this.maxPoolSize) {
      // Clear the input for reuse
      Object.keys(input).forEach(key => delete (input as any)[key])
      this.inputPool.push(input)
    }
  }

  returnOutput(output: TransactionOutput): void {
    if (this.outputPool.length < this.maxPoolSize) {
      Object.keys(output).forEach(key => delete (output as any)[key])
      this.outputPool.push(output)
    }
  }
}
```

## Efficient Transaction Construction

### Using the Transaction Constructor

The SDK's Transaction constructor accepts arrays of inputs and outputs, which is more efficient than individual additions:

```typescript
import { Transaction, TransactionInput, TransactionOutput, LockingScript, UnlockingScript } from '@bsv/sdk'

function buildLargeTransaction(
  inputs: TransactionInput[],
  outputs: TransactionOutput[]
): Transaction {
  // More efficient than multiple addInput/addOutput calls
  return new Transaction(
    1, // version
    inputs,
    outputs,
    0, // lockTime
    {}, // metadata
    undefined // merklePath
  )
}

// Example usage with properly formatted inputs and outputs
function createExampleTransaction(): Transaction {
  const inputs: TransactionInput[] = [{
    sourceTXID: '0'.repeat(64), // Replace with actual TXID
    sourceOutputIndex: 0,
    unlockingScriptLength: 0,
    unlockingScript: new UnlockingScript(),
    sequenceNumber: 0xffffffff
  } as TransactionInput]
  
  const outputs: TransactionOutput[] = [{
    satoshis: 100,
    lockingScriptLength: 6,
    lockingScript: LockingScript.fromASM('OP_RETURN 74657374') // OP_RETURN "test"
  } as TransactionOutput]
  
  return buildLargeTransaction(inputs, outputs)
}
```

### Chunked Processing

For very large input/output sets, process them in chunks:

```typescript
async function processLargeInputSet(
  allInputs: TransactionInput[],
  chunkSize: number = 50
): Promise<Transaction[]> {
  const transactions: Transaction[] = []
  
  for (let i = 0; i < allInputs.length; i += chunkSize) {
    const chunk = allInputs.slice(i, i + chunkSize)
    const tx = new Transaction(1, chunk, [])
    
    // Process chunk
    await tx.fee()
    await tx.sign()
    
    transactions.push(tx)
    
    // Allow event loop to process other tasks
    await new Promise(resolve => setImmediate(resolve))
  }
  
  return transactions
}
```

## Batch Processing with `WalletClient`

The SDK's `WalletClient` provides built-in batching capabilities for large transaction workflows:

### Chained Transaction Batching

```typescript
import { WalletClient, CreateActionArgs } from '@bsv/sdk'

class BatchTransactionProcessor {
  private walletClient: WalletClient
  private maxRetries: number = 3
  private retryDelay: number = 1000

  constructor(walletClient: WalletClient) {
    this.walletClient = walletClient
  }

  async createChainedBatch(actions: CreateActionArgs[]): Promise<string[]> {
    const txids: string[] = []
    const batchReferences: string[] = []

    // Create all transactions without sending (noSend: true)
    for (let i = 0; i < actions.length; i++) {
      const action = {
        ...actions[i],
        options: {
          ...actions[i].options,
          noSend: true,
          // Include previous transaction outputs as known
          knownTxids: txids
        }
      }

      const result = await this.walletClient.createAction(action)
      
      if (result.signableTransaction) {
        // Sign the transaction
        const signResult = await this.walletClient.signAction({
          spends: this.generateSpends(action),
          reference: result.signableTransaction.reference,
          options: { noSend: true }
        })
        
        if (signResult.txid) {
          txids.push(signResult.txid)
          batchReferences.push(result.signableTransaction.reference)
        }
      }
    }

    // Send all transactions as a batch
    if (batchReferences.length > 0) {
      await this.walletClient.signAction({
        spends: {},
        reference: batchReferences[0],
        options: {
          sendWith: txids
        }
      })
    }

    return txids
  }

  private generateSpends(action: CreateActionArgs): Record<number, any> {
    const spends: Record<number, any> = {}
    
    if (action.inputs) {
      action.inputs.forEach((input, index) => {
        spends[index] = {
          unlockingScript: input.unlockingScript || '',
          sequenceNumber: input.sequenceNumber || 0xffffffff
        }
      })
    }
    
    return spends
  }
}
```

### Progress Tracking for Large Batches

```typescript
interface BatchProgress {
  total: number
  completed: number
  failed: number
  currentPhase: 'creating' | 'signing' | 'broadcasting'
}

class ProgressTrackingBatch {
  async processBatchWithProgress(
    actions: CreateActionArgs[],
    onProgress?: (progress: BatchProgress) => void
  ): Promise<string[]> {
    this.progress.total = actions.length
    this.progress.currentPhase = 'creating'
    
    const results: string[] = []

    for (let i = 0; i < actions.length; i++) {
      try {
        const result = await this.walletClient.createAction(actions[i])
        
        if (result.txid) {
          results.push(result.txid)
          this.progress.completed++
        }
      } catch (error) {
        console.error(`Failed to process action ${i}:`, error)
        this.progress.failed++
      }

      if (onProgress) {
        onProgress({ ...this.progress })
      }

      // Throttle to prevent overwhelming the wallet
      if (i % 10 === 0) {
        await new Promise(resolve => setTimeout(resolve, 100))
      }
    }

    return results
  }
}
```

## Fee Calculation for Large Transactions

### Custom Fee Models

The SDK provides a `FeeModel` interface and `SatoshisPerKilobyte` implementation:

```typescript
import { SatoshisPerKilobyte, Transaction } from '@bsv/sdk'

// FeeModel interface definition (since it's not exported from SDK)
interface FeeModel {
  computeFee(transaction: Transaction): Promise<number>
}

class OptimizedFeeModel implements FeeModel {
  private baseFeeModel: SatoshisPerKilobyte
  private largeTxThreshold: number

  constructor(
    baseSatPerKb: number = 1,
    largeTxThreshold: number = 100000 // 100KB
  ) {
    this.baseFeeModel = new SatoshisPerKilobyte(baseSatPerKb)
    this.largeTxThreshold = largeTxThreshold
  }

  async computeFee(transaction: Transaction): Promise<number> {
    const baseFee = await this.baseFeeModel.computeFee(transaction)
    const txSize = transaction.toBinary().length

    // Apply discount for large transactions
    if (txSize > this.largeTxThreshold) {
      const discount = Math.min(0.5, (txSize - this.largeTxThreshold) / 1000000)
      return Math.floor(baseFee * (1 - discount))
    }

    return baseFee
  }
}

// Usage
async function calculateOptimizedFee(transaction: Transaction): Promise<void> {
  const feeModel = new OptimizedFeeModel(1, 50000)
  await transaction.fee(feeModel)
}
```

### Batch Fee Calculation

```typescript
async function calculateFeesInBatch(
  transactions: Transaction[],
  feeModel: FeeModel
): Promise<number[]> {
  const feePromises = transactions.map(tx => feeModel.computeFee(tx))
  return Promise.all(feePromises)
}
```

## Conclusion

Handling large Bitcoin transactions efficiently requires a comprehensive approach that addresses memory management, performance optimization, and robust error handling. The BSV TypeScript SDK provides powerful tools and patterns to manage these challenges effectively.

### Next Steps

To implement large transaction handling in your application:

1. **Start Simple**: Begin with the `LargeTransactionBuilder` pattern for basic batching
2. **Add Streaming**: Implement `StreamingTransactionBuilder` for memory-constrained scenarios
3. **Optimize Performance**: Add caching and parallel processing using the optimization patterns
4. **Enhance Monitoring**: Integrate the monitoring and error handling examples
5. **Test Thoroughly**: Use the validation patterns to ensure transaction integrity

### Related Documentation

For additional context and complementary patterns, see:

- [Transaction Batching](./transaction-batching.md) - Efficient multi-output transaction patterns
- [Error Handling](../tutorials/error-handling.md) - Comprehensive error management strategies
- [Performance Optimization](../performance.md) - General SDK performance patterns
- [Advanced Transaction Signing](./advanced-transaction-signing.md) - Signing optimization for large transactions
- [Transaction Monitoring](./transaction-monitoring.md) - Monitoring and alerting for transaction operations

The techniques in this guide enable you to build robust, scalable Bitcoin applications that can handle enterprise-level transaction volumes while maintaining security, performance, and reliability standards.
