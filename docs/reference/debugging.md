# Debugging Reference

Complete guide for debugging, logging, and troubleshooting applications built with the BSV TypeScript SDK.

## Debug Mode Activation

### Environment Variables

```bash
# Enable debug mode
export BSV_DEBUG=true

# Set debug level
export BSV_DEBUG_LEVEL=debug

# Enable specific debug categories
export BSV_DEBUG_CATEGORIES=wallet,transaction,network

# Enable performance debugging
export BSV_DEBUG_PERFORMANCE=true
```

### Programmatic Debug Configuration

```typescript
import { SDKConfigBuilder } from '@bsv/sdk'

const debugConfig = new SDKConfigBuilder()
  .logging({
    level: 'debug',
    outputs: [
      { type: 'console', config: {} },
      { type: 'file', config: { file: './debug.log' } }
    ],
    format: {
      timestamp: true,
      level: true,
      component: true,
      structured: true,
      colors: true
    },
    performance: {
      enabled: true,
      threshold: 100, // Log operations taking >100ms
      includeStackTrace: true
    }
  })
  .build()
```

## SDK Logging System

### Log Levels

```typescript
enum LogLevel {
  ERROR = 0,   // Critical errors only
  WARN = 1,    // Warnings and errors
  INFO = 2,    // General information
  DEBUG = 3,   // Detailed debugging info
  TRACE = 4    // Extremely verbose tracing
}
```

### Logger Interface

```typescript
interface Logger {
  error(message: string, context?: any): void
  warn(message: string, context?: any): void
  info(message: string, context?: any): void
  debug(message: string, context?: any): void
  trace(message: string, context?: any): void
  
  // Performance logging
  time(label: string): void
  timeEnd(label: string): void
  
  // Structured logging
  log(level: LogLevel, message: string, context: any): void
}
```

### Creating Custom Loggers

```typescript
class CustomLogger implements Logger {
  private level: LogLevel = LogLevel.INFO
  
  constructor(level: LogLevel = LogLevel.INFO) {
    this.level = level
  }
  
  error(message: string, context?: any): void {
    if (this.level >= LogLevel.ERROR) {
      console.error(`[ERROR] ${new Date().toISOString()} ${message}`, context)
    }
  }
  
  warn(message: string, context?: any): void {
    if (this.level >= LogLevel.WARN) {
      console.warn(`[WARN] ${new Date().toISOString()} ${message}`, context)
    }
  }
  
  info(message: string, context?: any): void {
    if (this.level >= LogLevel.INFO) {
      console.info(`[INFO] ${new Date().toISOString()} ${message}`, context)
    }
  }
  
  debug(message: string, context?: any): void {
    if (this.level >= LogLevel.DEBUG) {
      console.debug(`[DEBUG] ${new Date().toISOString()} ${message}`, context)
    }
  }
  
  trace(message: string, context?: any): void {
    if (this.level >= LogLevel.TRACE) {
      console.trace(`[TRACE] ${new Date().toISOString()} ${message}`, context)
    }
  }
  
  time(label: string): void {
    console.time(label)
  }
  
  timeEnd(label: string): void {
    console.timeEnd(label)
  }
  
  log(level: LogLevel, message: string, context: any): void {
    const methods = [this.error, this.warn, this.info, this.debug, this.trace]
    methods[level]?.call(this, message, context)
  }
}
```

## Debug Categories

### Wallet Debugging

```typescript
// Enable wallet debugging
const walletDebugger = {
  logConnection: (substrate: string, status: string) => {
    console.debug(`[WALLET] Connection ${status} to ${substrate}`)
  },
  
  logAction: (action: string, args: any) => {
    console.debug(`[WALLET] Action: ${action}`, {
      args: JSON.stringify(args, null, 2),
      timestamp: new Date().toISOString()
    })
  },
  
  logError: (error: any, context: any) => {
    console.error(`[WALLET] Error:`, {
      error: error.message,
      code: error.code,
      context,
      stack: error.stack
    })
  }
}

// Usage in wallet operations
try {
  walletDebugger.logAction('createAction', { description: 'Test transaction' })
  const result = await wallet.createAction({
    description: 'Test transaction',
    outputs: [{ satoshis: 100, lockingScript: '...' }]
  })
  walletDebugger.logConnection('substrate', 'success')
} catch (error) {
  walletDebugger.logError(error, { operation: 'createAction' })
}
```

### Transaction Debugging

```typescript
class TransactionDebugger {
  static logConstruction(tx: Transaction): void {
    console.debug(`[TRANSACTION] Construction:`, {
      inputs: tx.inputs.length,
      outputs: tx.outputs.length,
      totalInput: tx.inputs.reduce((sum, input) => sum + input.satoshis, 0),
      totalOutput: tx.outputs.reduce((sum, output) => sum + output.satoshis, 0),
      fee: tx.getFee(),
      size: tx.toHex().length / 2,
      txid: Buffer.from(tx.id()).toString('hex')
    })
  }
  
  static logSigning(tx: Transaction, inputIndex: number): void {
    console.debug(`[TRANSACTION] Signing input ${inputIndex}:`, {
      txid: Buffer.from(tx.id()).toString('hex'),
      input: {
        sourceTXID: tx.inputs[inputIndex].sourceTXID,
        sourceOutputIndex: tx.inputs[inputIndex].sourceOutputIndex,
        unlockingScript: tx.inputs[inputIndex].unlockingScript?.toASM()
      }
    })
  }
  
  static logValidation(tx: Transaction, isValid: boolean, errors?: string[]): void {
    console.debug(`[TRANSACTION] Validation:`, {
      txid: Buffer.from(tx.id()).toString('hex'),
      valid: isValid,
      errors: errors || [],
      timestamp: new Date().toISOString()
    })
  }
}
```

### Network Debugging

```typescript
class NetworkDebugger {
  static logRequest(endpoint: string, method: string, data?: any): void {
    console.debug(`[NETWORK] Request:`, {
      endpoint,
      method,
      data: data ? JSON.stringify(data) : undefined,
      timestamp: new Date().toISOString()
    })
  }
  
  static logResponse(endpoint: string, status: number, data?: any): void {
    console.debug(`[NETWORK] Response:`, {
      endpoint,
      status,
      dataSize: data ? JSON.stringify(data).length : 0,
      timestamp: new Date().toISOString()
    })
  }
  
  static logError(endpoint: string, error: any): void {
    console.error(`[NETWORK] Error:`, {
      endpoint,
      error: error.message,
      code: error.code,
      stack: error.stack,
      timestamp: new Date().toISOString()
    })
  }
}
```

## Transaction Inspection Tools

### Transaction Analyzer

```typescript
class TransactionAnalyzer {
  static analyze(tx: Transaction): TransactionAnalysis {
    const analysis: TransactionAnalysis = {
      basic: this.analyzeBasic(tx),
      inputs: this.analyzeInputs(tx),
      outputs: this.analyzeOutputs(tx),
      scripts: this.analyzeScripts(tx),
      fees: this.analyzeFees(tx),
      size: this.analyzeSize(tx)
    }
    
    return analysis
  }
  
  private static analyzeBasic(tx: Transaction): BasicAnalysis {
    return {
      txid: Buffer.from(tx.id()).toString('hex'),
      version: tx.version,
      lockTime: tx.lockTime,
      inputCount: tx.inputs.length,
      outputCount: tx.outputs.length
    }
  }
  
  private static analyzeInputs(tx: Transaction): InputAnalysis[] {
    return tx.inputs.map((input, index) => ({
      index,
      sourceTXID: input.sourceTXID,
      sourceOutputIndex: input.sourceOutputIndex,
      satoshis: input.satoshis,
      unlockingScript: {
        asm: input.unlockingScript?.toASM(),
        hex: input.unlockingScript?.toHex(),
        size: input.unlockingScript?.toHex().length / 2
      },
      sequence: input.sequence
    }))
  }
  
  private static analyzeOutputs(tx: Transaction): OutputAnalysis[] {
    return tx.outputs.map((output, index) => ({
      index,
      satoshis: output.satoshis,
      lockingScript: {
        asm: output.lockingScript.toASM(),
        hex: output.lockingScript.toHex(),
        size: output.lockingScript.toHex().length / 2,
        type: this.detectScriptType(output.lockingScript)
      }
    }))
  }
  
  private static detectScriptType(script: Script): string {
    const asm = script.toASM()
    
    if (asm.includes('OP_DUP OP_HASH160') && asm.includes('OP_EQUALVERIFY OP_CHECKSIG')) {
      return 'P2PKH'
    } else if (asm.includes('OP_HASH160') && asm.includes('OP_EQUAL')) {
      return 'P2SH'
    } else if (asm.startsWith('OP_RETURN')) {
      return 'OP_RETURN'
    } else {
      return 'CUSTOM'
    }
  }
}

interface TransactionAnalysis {
  basic: BasicAnalysis
  inputs: InputAnalysis[]
  outputs: OutputAnalysis[]
  scripts: ScriptAnalysis
  fees: FeeAnalysis
  size: SizeAnalysis
}
```

### Script Debugger

```typescript
class ScriptDebugger {
  static debugScript(script: Script): ScriptDebugInfo {
    const chunks = script.chunks
    const debugInfo: ScriptDebugInfo = {
      chunks: [],
      opcodes: [],
      data: [],
      analysis: {
        isValid: true,
        errors: [],
        warnings: []
      }
    }
    
    chunks.forEach((chunk, index) => {
      if (chunk.opcode !== undefined) {
        debugInfo.opcodes.push({
          index,
          opcode: chunk.opcode,
          name: this.getOpcodeName(chunk.opcode),
          description: this.getOpcodeDescription(chunk.opcode)
        })
      }
      
      if (chunk.data) {
        debugInfo.data.push({
          index,
          data: chunk.data,
          hex: Buffer.from(chunk.data).toString('hex'),
          size: chunk.data.length
        })
      }
    })
    
    return debugInfo
  }
  
  private static getOpcodeName(opcode: number): string {
    const opcodeMap: Record<number, string> = {
      0: 'OP_0',
      76: 'OP_PUSHDATA1',
      77: 'OP_PUSHDATA2',
      78: 'OP_PUSHDATA4',
      79: 'OP_1NEGATE',
      81: 'OP_1',
      82: 'OP_2',
      // ... add more opcodes as needed
      118: 'OP_DUP',
      169: 'OP_HASH160',
      136: 'OP_EQUALVERIFY',
      172: 'OP_CHECKSIG'
    }
    
    return opcodeMap[opcode] || `OP_UNKNOWN_${opcode}`
  }
}
```

## Performance Debugging

### Performance Monitor

```typescript
class PerformanceMonitor {
  private static timers = new Map<string, number>()
  private static metrics = new Map<string, PerformanceMetric>()
  
  static startTimer(label: string): void {
    this.timers.set(label, performance.now())
  }
  
  static endTimer(label: string): number {
    const startTime = this.timers.get(label)
    if (!startTime) {
      console.warn(`Timer '${label}' was not started`)
      return 0
    }
    
    const duration = performance.now() - startTime
    this.timers.delete(label)
    
    // Update metrics
    const metric = this.metrics.get(label) || {
      count: 0,
      totalTime: 0,
      avgTime: 0,
      minTime: Infinity,
      maxTime: 0
    }
    
    metric.count++
    metric.totalTime += duration
    metric.avgTime = metric.totalTime / metric.count
    metric.minTime = Math.min(metric.minTime, duration)
    metric.maxTime = Math.max(metric.maxTime, duration)
    
    this.metrics.set(label, metric)
    
    console.debug(`[PERFORMANCE] ${label}: ${duration.toFixed(2)}ms`)
    return duration
  }
  
  static getMetrics(): Map<string, PerformanceMetric> {
    return new Map(this.metrics)
  }
  
  static resetMetrics(): void {
    this.metrics.clear()
    this.timers.clear()
  }
}

interface PerformanceMetric {
  count: number
  totalTime: number
  avgTime: number
  minTime: number
  maxTime: number
}
```

### Memory Usage Monitoring

```typescript
class MemoryMonitor {
  static logMemoryUsage(label: string): void {
    if (typeof process !== 'undefined' && process.memoryUsage) {
      const usage = process.memoryUsage()
      console.debug(`[MEMORY] ${label}:`, {
        rss: `${Math.round(usage.rss / 1024 / 1024)}MB`,
        heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)}MB`,
        heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)}MB`,
        external: `${Math.round(usage.external / 1024 / 1024)}MB`
      })
    } else if (typeof performance !== 'undefined' && (performance as any).memory) {
      const memory = (performance as any).memory
      console.debug(`[MEMORY] ${label}:`, {
        used: `${Math.round(memory.usedJSHeapSize / 1024 / 1024)}MB`,
        total: `${Math.round(memory.totalJSHeapSize / 1024 / 1024)}MB`,
        limit: `${Math.round(memory.jsHeapSizeLimit / 1024 / 1024)}MB`
      })
    }
  }
}
```

## Debug Utilities

### Hex Dump Utility

```typescript
class HexDump {
  static dump(data: Uint8Array | number[], bytesPerLine: number = 16): string {
    const bytes = Array.from(data)
    let result = ''
    
    for (let i = 0; i < bytes.length; i += bytesPerLine) {
      const line = bytes.slice(i, i + bytesPerLine)
      const offset = i.toString(16).padStart(8, '0')
      const hex = line.map(b => b.toString(16).padStart(2, '0')).join(' ')
      const ascii = line.map(b => (b >= 32 && b <= 126) ? String.fromCharCode(b) : '.').join('')
      
      result += `${offset}: ${hex.padEnd(bytesPerLine * 3 - 1)} |${ascii}|\n`
    }
    
    return result
  }
  
  static logDump(data: Uint8Array | number[], label: string): void {
    console.debug(`[HEX DUMP] ${label}:\n${this.dump(data)}`)
  }
}
```

### Network Request Inspector

```typescript
class NetworkInspector {
  private static requests = new Map<string, NetworkRequest>()
  
  static startRequest(id: string, url: string, method: string, data?: any): void {
    this.requests.set(id, {
      id,
      url,
      method,
      data,
      startTime: Date.now(),
      status: 'pending'
    })
    
    console.debug(`[NETWORK] Starting request ${id}:`, {
      url,
      method,
      dataSize: data ? JSON.stringify(data).length : 0
    })
  }
  
  static endRequest(id: string, status: number, response?: any): void {
    const request = this.requests.get(id)
    if (!request) return
    
    const duration = Date.now() - request.startTime
    request.status = 'completed'
    request.duration = duration
    request.response = response
    
    console.debug(`[NETWORK] Completed request ${id}:`, {
      status,
      duration: `${duration}ms`,
      responseSize: response ? JSON.stringify(response).length : 0
    })
    
    // Clean up old requests
    setTimeout(() => this.requests.delete(id), 60000)
  }
  
  static failRequest(id: string, error: any): void {
    const request = this.requests.get(id)
    if (!request) return
    
    const duration = Date.now() - request.startTime
    request.status = 'failed'
    request.duration = duration
    request.error = error
    
    console.error(`[NETWORK] Failed request ${id}:`, {
      error: error.message,
      duration: `${duration}ms`
    })
  }
  
  static getActiveRequests(): NetworkRequest[] {
    return Array.from(this.requests.values()).filter(r => r.status === 'pending')
  }
}

interface NetworkRequest {
  id: string
  url: string
  method: string
  data?: any
  startTime: number
  duration?: number
  status: 'pending' | 'completed' | 'failed'
  response?: any
  error?: any
}
```

## Debug Configuration Examples

### Development Debug Setup

```typescript
// Enable comprehensive debugging for development
const devDebugConfig = {
  logging: {
    level: 'debug' as LogLevel,
    outputs: [
      { type: 'console', config: { colors: true } },
      { type: 'file', config: { file: './dev-debug.log' } }
    ],
    format: {
      timestamp: true,
      level: true,
      component: true,
      structured: true,
      colors: true
    },
    performance: {
      enabled: true,
      threshold: 50,
      includeStackTrace: true
    }
  }
}
```

### Production Debug Setup

```typescript
// Minimal debugging for production
const prodDebugConfig = {
  logging: {
    level: 'warn' as LogLevel,
    outputs: [
      { type: 'file', config: { file: '/var/log/bsv-app.log', rotation: true } }
    ],
    format: {
      timestamp: true,
      level: true,
      component: false,
      structured: true,
      colors: false
    },
    performance: {
      enabled: true,
      threshold: 1000,
      includeStackTrace: false
    }
  }
}
```

### Testing Debug Setup

```typescript
// Debug configuration for testing
const testDebugConfig = {
  logging: {
    level: 'trace' as LogLevel,
    outputs: [
      { type: 'console', config: {} }
    ],
    format: {
      timestamp: false,
      level: true,
      component: true,
      structured: false,
      colors: false
    },
    performance: {
      enabled: false
    }
  }
}
```

## Troubleshooting Common Issues

### Debug Checklist

1. **Enable appropriate logging level**
   ```typescript
   // Set debug level based on issue severity
   const config = { logging: { level: 'debug' } }
   ```

2. **Check network connectivity**
   ```typescript
   NetworkInspector.startRequest('health-check', 'https://api.whatsonchain.com/v1/bsv/main/chain/info', 'GET')
   ```

3. **Validate transaction structure**
   ```typescript
   const analysis = TransactionAnalyzer.analyze(transaction)
   console.log('Transaction Analysis:', analysis)
   ```

4. **Monitor performance**
   ```typescript
   PerformanceMonitor.startTimer('wallet-operation')
   // ... perform operation
   PerformanceMonitor.endTimer('wallet-operation')
   ```

5. **Check memory usage**
   ```typescript
   MemoryMonitor.logMemoryUsage('before-operation')
   // ... perform operation
   MemoryMonitor.logMemoryUsage('after-operation')
   ```

This comprehensive debugging reference provides developers with all the tools and techniques needed to effectively debug and troubleshoot applications built with the BSV TypeScript SDK.
