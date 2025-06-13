# Error Reference

Complete reference for error codes, messages, and troubleshooting in the BSV TypeScript SDK.

## Error Categories

### Transaction Errors

#### INSUFFICIENT_FUNDS
**Code**: `INSUFFICIENT_FUNDS`  
**Message**: "Insufficient funds to create transaction"  
**Cause**: Wallet doesn't have enough UTXOs to cover transaction outputs and fees  
**Solutions**:
- Check wallet balance with `listOutputs()`
- Reduce transaction amount
- Wait for pending transactions to confirm
- Use smaller fee rates

#### INVALID_TRANSACTION
**Code**: `INVALID_TRANSACTION`  
**Message**: "Transaction validation failed"  
**Cause**: Transaction structure or signatures are invalid  
**Solutions**:
- Verify all inputs are properly signed
- Check script templates are correct
- Ensure transaction format is valid
- Validate all output amounts are positive

#### TRANSACTION_TOO_LARGE
**Code**: `TRANSACTION_TOO_LARGE`  
**Message**: "Transaction exceeds maximum size limit"  
**Cause**: Transaction size exceeds network limits  
**Solutions**:
- Reduce number of inputs/outputs
- Use more efficient script templates
- Split into multiple transactions
- Optimize data storage methods

#### INVALID_SCRIPT
**Code**: `INVALID_SCRIPT`  
**Message**: "Script execution failed"  
**Cause**: Locking or unlocking script contains errors  
**Solutions**:
- Validate script syntax with `Script.fromASM()`
- Check opcode usage and limits
- Verify script template implementation
- Test script execution in isolation

### Wallet Errors

#### WALLET_NOT_CONNECTED
**Code**: `WALLET_NOT_CONNECTED`  
**Message**: "Wallet connection not established"  
**Cause**: WalletClient not connected to substrate  
**Solutions**:
- Call `await wallet.connectToSubstrate()`
- Check wallet application is running
- Verify network connectivity
- Restart wallet application if needed

#### AUTHENTICATION_FAILED
**Code**: `AUTHENTICATION_FAILED`  
**Message**: "Wallet authentication failed"  
**Cause**: User denied access or authentication expired  
**Solutions**:
- Re-authenticate with wallet
- Check originator domain permissions
- Verify wallet trust settings
- Clear cached authentication state

#### WALLET_LOCKED
**Code**: `WALLET_LOCKED`  
**Message**: "Wallet is locked or requires password"  
**Cause**: Wallet requires user authentication  
**Solutions**:
- Unlock wallet application
- Enter wallet password
- Check wallet auto-lock settings
- Verify user session is active

#### ACTION_REJECTED
**Code**: `ACTION_REJECTED`  
**Message**: "User rejected the transaction"  
**Cause**: User declined transaction in wallet UI  
**Solutions**:
- Retry transaction with user consent
- Adjust transaction parameters
- Provide clearer transaction description
- Check transaction amounts and fees

### Network Errors

#### NETWORK_ERROR
**Code**: `NETWORK_ERROR`  
**Message**: "Network request failed"  
**Cause**: Connection issues with blockchain nodes  
**Solutions**:
- Check internet connectivity
- Verify node endpoints are accessible
- Try alternative chain trackers
- Implement retry logic with exponential backoff

#### NODE_UNAVAILABLE
**Code**: `NODE_UNAVAILABLE`  
**Message**: "Blockchain node is unavailable"  
**Cause**: Target node is down or unreachable  
**Solutions**:
- Switch to backup node endpoints
- Check node status and health
- Use multiple chain tracker instances
- Implement failover mechanisms

#### BROADCAST_FAILED
**Code**: `BROADCAST_FAILED`  
**Message**: "Transaction broadcast failed"  
**Cause**: Network rejected transaction or broadcast error  
**Solutions**:
- Verify transaction is valid
- Check network fees are adequate
- Retry broadcast after delay
- Use alternative broadcast endpoints

#### TIMEOUT_ERROR
**Code**: `TIMEOUT_ERROR`  
**Message**: "Request timeout exceeded"  
**Cause**: Network request took too long  
**Solutions**:
- Increase timeout values
- Check network latency
- Use faster endpoints
- Implement request cancellation

### Cryptographic Errors

#### INVALID_PRIVATE_KEY
**Code**: `INVALID_PRIVATE_KEY`  
**Message**: "Private key is invalid or out of range"  
**Cause**: Private key doesn't meet secp256k1 requirements  
**Solutions**:
- Generate new key with `PrivateKey.fromRandom()`
- Validate key is within curve order
- Check key format and encoding
- Use proper key derivation methods

#### INVALID_PUBLIC_KEY
**Code**: `INVALID_PUBLIC_KEY`  
**Message**: "Public key is invalid or not on curve"  
**Cause**: Public key point is invalid  
**Solutions**:
- Verify key derivation from private key
- Check point coordinates are valid
- Validate key format (compressed/uncompressed)
- Use `PublicKey.fromPrivateKey()` for generation

#### SIGNATURE_VERIFICATION_FAILED
**Code**: `SIGNATURE_VERIFICATION_FAILED`  
**Message**: "Digital signature verification failed"  
**Cause**: Signature doesn't match message and public key  
**Solutions**:
- Verify message hash is correct
- Check signature format (DER encoding)
- Ensure correct public key is used
- Validate signature components (r, s values)

#### ENCRYPTION_FAILED
**Code**: `ENCRYPTION_FAILED`  
**Message**: "Symmetric encryption operation failed"  
**Cause**: AES encryption/decryption error  
**Solutions**:
- Verify encryption key is valid
- Check data format and encoding
- Ensure proper IV/nonce usage
- Validate authentication tags

### SPV Verification Errors

#### INVALID_MERKLE_PROOF
**Code**: `INVALID_MERKLE_PROOF`  
**Message**: "Merkle proof verification failed"  
**Cause**: Merkle path doesn't lead to valid root  
**Solutions**:
- Verify merkle path structure
- Check transaction hash calculation
- Validate block header merkle root
- Ensure proof completeness

#### BLOCK_HEADER_INVALID
**Code**: `BLOCK_HEADER_INVALID`  
**Message**: "Block header validation failed"  
**Cause**: Block header doesn't meet consensus rules  
**Solutions**:
- Verify header hash and difficulty
- Check timestamp validity
- Validate previous block hash
- Ensure proper header format

#### CHAIN_VALIDATION_FAILED
**Code**: `CHAIN_VALIDATION_FAILED`  
**Message**: "Blockchain validation failed"  
**Cause**: Chain doesn't follow consensus rules  
**Solutions**:
- Verify chain continuity
- Check difficulty adjustments
- Validate block timestamps
- Ensure proper chain selection

### Configuration Errors

#### INVALID_CONFIG
**Code**: `INVALID_CONFIG`  
**Message**: "SDK configuration is invalid"  
**Cause**: Configuration parameters are incorrect  
**Solutions**:
- Validate configuration schema
- Check required parameters are present
- Verify network settings
- Use default configuration as baseline

#### UNSUPPORTED_NETWORK
**Code**: `UNSUPPORTED_NETWORK`  
**Message**: "Network type is not supported"  
**Cause**: Invalid network specification  
**Solutions**:
- Use supported networks: mainnet, testnet, regtest
- Check network configuration
- Verify chain parameters
- Update SDK to latest version

## Error Interface

All SDK errors implement the `WalletErrorObject` interface:

```typescript
interface WalletErrorObject {
  code: string
  description: string
  stack?: string
  context?: Record<string, any>
}
```

## Error Handling Patterns

### Basic Error Handling

```typescript
try {
  const result = await wallet.createAction({
    description: 'Test transaction',
    outputs: [{
      satoshis: 100,
      lockingScript: Script.fromASM('OP_DUP OP_HASH160 ... OP_EQUALVERIFY OP_CHECKSIG').toHex()
    }]
  })
} catch (error) {
  if (error.code === 'INSUFFICIENT_FUNDS') {
    console.log('Not enough funds available')
    // Handle insufficient funds
  } else if (error.code === 'WALLET_NOT_CONNECTED') {
    console.log('Wallet connection required')
    await wallet.connectToSubstrate()
  } else {
    console.error('Unexpected error:', error)
  }
}
```

### Retry Logic with Exponential Backoff

```typescript
async function retryOperation<T>(
  operation: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await operation()
    } catch (error) {
      if (attempt === maxRetries) throw error
      
      // Only retry on network errors
      if (!['NETWORK_ERROR', 'TIMEOUT_ERROR', 'NODE_UNAVAILABLE'].includes(error.code)) {
        throw error
      }
      
      const delay = baseDelay * Math.pow(2, attempt)
      await new Promise(resolve => setTimeout(resolve, delay))
    }
  }
  throw new Error('Max retries exceeded')
}
```

### Comprehensive Error Categorization

```typescript
function categorizeError(error: WalletErrorObject): 'user' | 'network' | 'system' | 'config' {
  const userErrors = ['ACTION_REJECTED', 'AUTHENTICATION_FAILED', 'WALLET_LOCKED']
  const networkErrors = ['NETWORK_ERROR', 'NODE_UNAVAILABLE', 'BROADCAST_FAILED', 'TIMEOUT_ERROR']
  const systemErrors = ['INVALID_TRANSACTION', 'SIGNATURE_VERIFICATION_FAILED', 'ENCRYPTION_FAILED']
  const configErrors = ['INVALID_CONFIG', 'UNSUPPORTED_NETWORK']
  
  if (userErrors.includes(error.code)) return 'user'
  if (networkErrors.includes(error.code)) return 'network'
  if (systemErrors.includes(error.code)) return 'system'
  if (configErrors.includes(error.code)) return 'config'
  
  return 'system' // Default category
}
```

## Troubleshooting Guide

### Common Issues and Solutions

#### "RPC Error: no header should have returned false"
**Symptoms**: Error during `createAction` calls  
**Cause**: Wallet input selection issues  
**Solutions**:
1. Restart wallet application
2. Ensure wallet is fully synced
3. Use slightly larger amounts (200-500 satoshis)
4. Wait for wallet synchronization
5. Check for sufficient confirmed UTXOs

#### "Insufficient funds" with Available Balance
**Symptoms**: Error despite wallet showing balance  
**Cause**: UTXOs not properly selected by wallet  
**Solutions**:
1. Check UTXO status with `listOutputs()`
2. Verify UTXOs are confirmed
3. Restart wallet to refresh UTXO cache
4. Use manual input selection if supported

#### Transaction Broadcast Failures
**Symptoms**: Valid transactions rejected by network  
**Cause**: Various network or validation issues  
**Solutions**:
1. Verify transaction format and signatures
2. Check fee rates are adequate
3. Ensure inputs are unspent
4. Try alternative broadcast endpoints

#### Wallet Connection Issues
**Symptoms**: Cannot connect to wallet substrate  
**Cause**: Wallet not running or permission issues  
**Solutions**:
1. Ensure wallet application is running
2. Check originator domain permissions
3. Clear browser cache and cookies
4. Verify wallet trust settings

### Diagnostic Tools

#### Transaction Validation

```typescript
function validateTransaction(tx: Transaction): string[] {
  const errors: string[] = []
  
  if (tx.inputs.length === 0) {
    errors.push('Transaction must have at least one input')
  }
  
  if (tx.outputs.length === 0) {
    errors.push('Transaction must have at least one output')
  }
  
  const totalInput = tx.inputs.reduce((sum, input) => sum + input.satoshis, 0)
  const totalOutput = tx.outputs.reduce((sum, output) => sum + output.satoshis, 0)
  
  if (totalInput <= totalOutput) {
    errors.push('Insufficient input value to cover outputs and fees')
  }
  
  return errors
}
```

#### Network Connectivity Test

```typescript
async function testNetworkConnectivity(chainTracker: ChainTracker): Promise<boolean> {
  try {
    const height = await chainTracker.getHeight()
    return height > 0
  } catch (error) {
    console.error('Network connectivity test failed:', error)
    return false
  }
}
```

#### Wallet Health Check

```typescript
async function checkWalletHealth(wallet: WalletClient): Promise<{
  connected: boolean
  authenticated: boolean
  balance: number
  errors: string[]
}> {
  const result = {
    connected: false,
    authenticated: false,
    balance: 0,
    errors: [] as string[]
  }
  
  try {
    await wallet.connectToSubstrate()
    result.connected = true
  } catch (error) {
    result.errors.push(`Connection failed: ${error.message}`)
  }
  
  try {
    const outputs = await wallet.listOutputs({ limit: 1 })
    result.authenticated = true
    result.balance = outputs.totalValue || 0
  } catch (error) {
    result.errors.push(`Authentication failed: ${error.message}`)
  }
  
  return result
}
```

## Error Recovery Strategies

### Automatic Recovery

```typescript
class ErrorRecoveryManager {
  private retryAttempts = new Map<string, number>()
  private maxRetries = 3
  
  async executeWithRecovery<T>(
    operation: () => Promise<T>,
    operationId: string
  ): Promise<T> {
    try {
      const result = await operation()
      this.retryAttempts.delete(operationId)
      return result
    } catch (error) {
      return this.handleError(error, operation, operationId)
    }
  }
  
  private async handleError<T>(
    error: WalletErrorObject,
    operation: () => Promise<T>,
    operationId: string
  ): Promise<T> {
    const attempts = this.retryAttempts.get(operationId) || 0
    
    if (attempts >= this.maxRetries) {
      this.retryAttempts.delete(operationId)
      throw error
    }
    
    // Implement recovery strategies based on error type
    switch (error.code) {
      case 'WALLET_NOT_CONNECTED':
        await this.reconnectWallet()
        break
      case 'NETWORK_ERROR':
        await this.switchToBackupNode()
        break
      case 'INSUFFICIENT_FUNDS':
        await this.waitForConfirmations()
        break
      default:
        throw error // No recovery strategy available
    }
    
    this.retryAttempts.set(operationId, attempts + 1)
    return this.executeWithRecovery(operation, operationId)
  }
  
  private async reconnectWallet(): Promise<void> {
    // Implement wallet reconnection logic
  }
  
  private async switchToBackupNode(): Promise<void> {
    // Implement node failover logic
  }
  
  private async waitForConfirmations(): Promise<void> {
    // Wait for pending transactions to confirm
    await new Promise(resolve => setTimeout(resolve, 30000))
  }
}
```

## Best Practices

### Error Logging

```typescript
class ErrorLogger {
  static log(error: WalletErrorObject, context?: Record<string, any>): void {
    const logEntry = {
      timestamp: new Date().toISOString(),
      code: error.code,
      message: error.description,
      context: { ...error.context, ...context },
      stack: error.stack
    }
    
    // Log to appropriate destination based on severity
    if (this.isCriticalError(error.code)) {
      console.error('CRITICAL ERROR:', logEntry)
    } else {
      console.warn('ERROR:', logEntry)
    }
  }
  
  private static isCriticalError(code: string): boolean {
    return [
      'INVALID_PRIVATE_KEY',
      'SIGNATURE_VERIFICATION_FAILED',
      'CHAIN_VALIDATION_FAILED'
    ].includes(code)
  }
}
```

### User-Friendly Error Messages

```typescript
function getUserFriendlyMessage(error: WalletErrorObject): string {
  const messages: Record<string, string> = {
    'INSUFFICIENT_FUNDS': 'You don\'t have enough funds for this transaction.',
    'ACTION_REJECTED': 'Transaction was cancelled.',
    'WALLET_NOT_CONNECTED': 'Please connect your wallet to continue.',
    'NETWORK_ERROR': 'Network connection issue. Please try again.',
    'INVALID_TRANSACTION': 'Transaction format is invalid. Please check your inputs.'
  }
  
  return messages[error.code] || 'An unexpected error occurred. Please try again.'
}
```

This comprehensive error reference provides developers with the tools and knowledge needed to handle all types of errors that may occur when using the BSV TypeScript SDK.
