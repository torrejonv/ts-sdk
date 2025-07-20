# Multi-Signature Transactions

Multi-signature (multisig) transactions require multiple signatures to spend funds, providing enhanced security and shared control over Bitcoin outputs. This guide covers implementing multisig transactions using the BSV TypeScript SDK, from basic 2-of-3 setups to advanced patterns.

## Table of Contents

1. [Multi-Signature Fundamentals](#multi-signature-fundamentals)
2. [Basic 2-of-3 Implementation](#basic-2-of-3-implementation)
3. [Funding the Multisig Address](#funding-the-multisig-address)
4. [Spending from Multisig](#spending-from-multisig)
5. [Advanced Multisig Patterns](#advanced-multisig-patterns)
6. [Error Handling and Validation](#error-handling-and-validation)
7. [Best Practices](#best-practices)
8. [Testing Multisig Implementation](#testing-multisig-implementation)
9. [Troubleshooting Common Issues](#troubleshooting-common-issues)

## Multi-Signature Fundamentals

Multi-signature transactions use the `OP_CHECKMULTISIG` opcode to require multiple valid signatures from a set of public keys. Common patterns include:

- **2-of-2**: Both parties must sign (joint accounts)
- **2-of-3**: Any 2 of 3 parties must sign (escrow with arbiter)
- **3-of-5**: Any 3 of 5 parties must sign (corporate governance)

### Key Concepts

- **Threshold**: Minimum number of required signatures
- **Public Key Set**: All possible signers
- **Signature Ordering**: Signatures must match public key order
- **OP_0 Bug**: Required extra OP_0 due to Bitcoin's OP_CHECKMULTISIG implementation

## Basic 2-of-3 Implementation

### Step 1: Generate Key Pairs

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

// Generate three key pairs for the multisig
const key1 = PrivateKey.fromRandom()
const key2 = PrivateKey.fromRandom()
const key3 = PrivateKey.fromRandom()

const pubKey1 = key1.toPublicKey()
const pubKey2 = key2.toPublicKey()
const pubKey3 = key3.toPublicKey()

console.log('Generated 3 key pairs for 2-of-3 multisig')
```

### Step 2: Create Multisig Script Template

```typescript
import { Script, OP, ScriptTemplate, Transaction, UnlockingScript, LockingScript, ScriptTemplateUnlock } from '@bsv/sdk'

class MultiSigTemplate implements ScriptTemplate {
  private threshold: number
  private publicKeys: PublicKey[]

  constructor(threshold: number, publicKeys: PublicKey[]) {
    if (threshold > publicKeys.length) {
      throw new Error('Threshold cannot exceed number of public keys')
    }
    if (threshold < 1) {
      throw new Error('Threshold must be at least 1')
    }
    if (publicKeys.length > 16) {
      throw new Error('Maximum 16 public keys allowed')
    }

    this.threshold = threshold
    this.publicKeys = publicKeys.sort((a, b) => {
      const aStr = a.toString()
      const bStr = b.toString()
      return aStr.localeCompare(bStr)
    })
  }

  lock(): LockingScript {
    const script = new Script()
    
    // Push threshold (OP_1 through OP_16)
    if (this.threshold <= 16) {
      script.writeOpCode(OP.OP_1 + this.threshold - 1)
    } else {
      script.writeNumber(this.threshold)
    }
    
    // Push all public keys
    for (const pubKey of this.publicKeys) {
      const pubKeyHex = pubKey.toString()
      const pubKeyBytes = Array.from(Buffer.from(pubKeyHex, 'hex'))
      script.writeBin(pubKeyBytes)
    }
    
    // Push number of public keys
    if (this.publicKeys.length <= 16) {
      script.writeOpCode(OP.OP_1 + this.publicKeys.length - 1)
    } else {
      script.writeNumber(this.publicKeys.length)
    }
    
    // Add OP_CHECKMULTISIG
    script.writeOpCode(OP.OP_CHECKMULTISIG)
    
    return new LockingScript(script.chunks)
  }

  unlock(privateKeys: PrivateKey[]): ScriptTemplateUnlock {
    if (privateKeys.length < this.threshold) {
      throw new Error(`Need at least ${this.threshold} private keys`)
    }

    return {
      sign: async (tx: Transaction, inputIndex: number): Promise<UnlockingScript> => {
        const script = new Script()
        
        // Add OP_0 (required due to OP_CHECKMULTISIG bug)
        script.writeOpCode(OP.OP_0)
        
        // Create signatures with the first 'threshold' keys
        const signingKeys = privateKeys.slice(0, this.threshold)
        
        // Note: In a real implementation, you would create proper signatures here
        // This is a simplified example for demonstration
        for (let i = 0; i < this.threshold; i++) {
          // Placeholder for signature - in real implementation would sign transaction
          const dummySig = new Array(72).fill(0x30) // Dummy DER signature
          script.writeBin(dummySig)
        }
        
        return new UnlockingScript(script.chunks)
      },
      estimateLength: async (): Promise<number> => {
        // OP_0 + (threshold * signature_length)
        return 1 + (this.threshold * 73)
      }
    }
}
```

### Step 3: Create Multisig Address

```typescript
import { Hash } from '@bsv/sdk'

// Create 2-of-3 multisig template
const multisigTemplate = new MultiSigTemplate(2, [pubKey1, pubKey2, pubKey3])
const lockingScript = multisigTemplate.lock()

// Create script hash using available Hash methods
const scriptBytes = lockingScript.toBinary()
const scriptHash = Hash.sha256(Array.from(scriptBytes))

console.log('Multisig locking script:', lockingScript.toASM())
console.log('Script hash:', Buffer.from(scriptHash).toString('hex'))
```

## Funding the Multisig Address

### Using WalletClient

```typescript
import { WalletClient } from '@bsv/sdk'

async function fundMultisig(wallet: WalletClient, scriptHash: number[], amount: number) {
  try {
    const actionResult = await wallet.createAction({
      description: 'Fund 2-of-3 multisig address',
      outputs: [
        {
          satoshis: amount,
          lockingScript: Buffer.from(scriptHash).toString('hex'),
          outputDescription: 'Multisig funding'
        }
      ],
    })

    if (actionResult.txid) {
      console.log('Multisig funded with transaction:', actionResult.txid)
      return actionResult.txid
    } else {
      throw new Error('Failed to fund multisig address')
    }
  } catch (error) {
    console.error('Error funding multisig:', error)
    throw error
  }
}

// Example usage
const wallet = new WalletClient('https://staging-dojo.babbage.systems')
await wallet.authenticate()
const fundingTxid = await fundMultisig(wallet, scriptHash, 100) // 100 satoshis
```

## Spending from Multisig

### Step 1: Create Spending Transaction

```typescript
import { Transaction } from '@bsv/sdk'

async function createMultisigSpendingTx(
  fundingTxid: string,
  outputIndex: number,
  amount: number,
  recipientScript: Script,
  multisigTemplate: MultiSigTemplate
): Promise<Transaction> {
  
  const tx = new Transaction()
  
  // Add input from multisig funding transaction
  tx.inputs = [{
    sourceTXID: fundingTxid,
    sourceOutputIndex: outputIndex,
    unlockingScript: new Script(), // Will be filled later
    sequence: 0xffffffff
  }] as any
  
  // Add output (subtract small fee)
  tx.outputs = [{
    satoshis: amount - 100, // 100 satoshi fee
    lockingScript: recipientScript
  }] as any
  
  return tx
}
```

### Step 2: Generate Signatures

```typescript
function signMultisigTransaction(
  transaction: Transaction,
  inputIndex: number,
  privateKeys: PrivateKey[],
  multisigScript: Script,
  inputAmount: number
): Buffer[] {
  
  const signatures: Buffer[] = []
  
  for (const privateKey of privateKeys) {
    try {
      // Create a signature for the transaction
      // Note: This is a simplified example - in production you would use proper signature hash
      const testMessage = Array.from(Buffer.from('transaction_signature_data'))
      const signature = privateKey.sign(testMessage)
      
      // Create signature buffer with SIGHASH flag
      const sigBytes = Array.from(Buffer.from(signature.toDER(), 'hex'))
      sigBytes.push(0x41) // SIGHASH_ALL | SIGHASH_FORKID
      
      signatures.push(Buffer.from(sigBytes))
    } catch (error) {
      console.error('Error signing with key:', error)
      throw error
    }
  }
  
  return signatures
}
```

### Step 3: Complete Transaction

```typescript
async function spendFromMultisig(
  fundingTxid: string,
  outputIndex: number,
  amount: number,
  recipientAddress: string,
  signingKeys: PrivateKey[], // 2 keys for 2-of-3
  multisigTemplate: MultiSigTemplate
): Promise<string> {
  
  try {
    // Create recipient script (P2PKH for simplicity)
    const recipientScript = new Script()
    recipientScript.writeOpCode(OP.OP_DUP)
    recipientScript.writeOpCode(OP.OP_HASH160)
    recipientScript.writeBin(Buffer.from(recipientAddress, 'hex'))
    recipientScript.writeOpCode(OP.OP_EQUALVERIFY)
    recipientScript.writeOpCode(OP.OP_CHECKSIG)
    
    // Create spending transaction
    const tx = await createMultisigSpendingTx(
      fundingTxid,
      outputIndex,
      amount,
      recipientScript,
      multisigTemplate
    )
    
    // Get multisig locking script
    const multisigScript = multisigTemplate.lock()
    
    // Generate signatures
    const signatures = signMultisigTransaction(
      tx,
      0, // First input
      signingKeys,
      multisigScript,
      amount
    )
    
    // Create unlocking script
    const unlockingScript = multisigTemplate.unlock(signatures, tx, 0)
    tx.inputs[0].unlockingScript = unlockingScript.sign()
    
    // Verify transaction
    const isValid = tx.verify()
    if (!isValid) {
      throw new Error('Transaction verification failed')
    }
    
    console.log('Multisig transaction created successfully')
    console.log('Transaction hex:', tx.toHex())
    
    return tx.toHex()
    
  } catch (error) {
    console.error('Error spending from multisig:', error)
    throw error
  }
}

// Example usage
const spendingTx = await spendFromMultisig(
  fundingTxid,
  0, // Output index
  1000, // Amount
  'recipient_address_hash160',
  [key1, key2], // 2 signatures for 2-of-3
  multisigTemplate
)
```

## Advanced Multisig Patterns

### Threshold Signature Coordination

```typescript
class MultisigCoordinator {
  private template: MultiSigTemplate
  private participants: Map<string, PublicKey>
  private signatures: Map<string, Buffer>
  
  constructor(template: MultiSigTemplate, participants: PublicKey[]) {
    this.template = template
    this.participants = new Map()
    this.signatures = new Map()
    
    participants.forEach((pubKey, index) => {
      this.participants.set(`participant_${index}`, pubKey)
    })
  }
  
  addSignature(participantId: string, signature: Buffer): void {
    if (!this.participants.has(participantId)) {
      throw new Error('Unknown participant')
    }
    
    this.signatures.set(participantId, signature)
    console.log(`Signature added for ${participantId}`)
  }
  
  hasEnoughSignatures(): boolean {
    return this.signatures.size >= this.template['threshold']
  }
  
  getSignatures(): Buffer[] {
    const sigs = Array.from(this.signatures.values())
    return sigs.slice(0, this.template['threshold'])
  }
  
  createUnlockingScript(transaction: Transaction, inputIndex: number): UnlockingScript {
    if (!this.hasEnoughSignatures()) {
      throw new Error('Insufficient signatures')
    }
    
    return this.template.unlock(this.getSignatures(), transaction, inputIndex)
  }
}
```

### Time-Locked Multisig

```typescript
class TimeLockMultiSig extends MultiSigTemplate {
  private lockTime: number
  
  constructor(threshold: number, publicKeys: PublicKey[], lockTime: number) {
    super(threshold, publicKeys)
    this.lockTime = lockTime
  }
  
  lock(): Script {
    const script = new Script()
    
    // Add time lock (simplified - just add the number and drop it)
    script.writeNumber(this.lockTime)
    script.writeOpCode(OP.OP_DROP)
    
    // Add standard multisig
    const multisigScript = super.lock()
    const scriptBytes = multisigScript.toBinary()
    script.writeBin(Array.from(scriptBytes))
    
    return script
  }
}

// Usage
const timeLockMultisig = new TimeLockMultiSig(
  2, // 2-of-3
  [pubKey1, pubKey2, pubKey3],
  1640995200 // Unix timestamp
)
```

## Error Handling and Validation

### Comprehensive Error Handling

```typescript
class MultisigError extends Error {
  constructor(message: string, public code: string) {
    super(message)
    this.name = 'MultisigError'
  }
}

function validateMultisigSetup(
  threshold: number,
  publicKeys: PublicKey[]
): void {
  if (threshold < 1) {
    throw new MultisigError('Threshold must be at least 1', 'INVALID_THRESHOLD')
  }
  
  if (threshold > publicKeys.length) {
    throw new MultisigError(
      'Threshold cannot exceed number of public keys',
      'THRESHOLD_TOO_HIGH'
    )
  }
  
  if (publicKeys.length > 16) {
    throw new MultisigError(
      'Maximum 16 public keys allowed',
      'TOO_MANY_KEYS'
    )
  }
  
  // Check for duplicate keys
  const keySet = new Set(publicKeys.map(k => k.toString()))
  if (keySet.size !== publicKeys.length) {
    throw new MultisigError('Duplicate public keys detected', 'DUPLICATE_KEYS')
  }
}

function validateSignatures(
  signatures: Buffer[],
  expectedCount: number
): void {
  if (signatures.length !== expectedCount) {
    throw new MultisigError(
      `Expected ${expectedCount} signatures, got ${signatures.length}`,
      'INVALID_SIGNATURE_COUNT'
    )
  }
  
  for (let i = 0; i < signatures.length; i++) {
    if (signatures[i].length < 70 || signatures[i].length > 73) {
      throw new MultisigError(
        `Invalid signature length at index ${i}`,
        'INVALID_SIGNATURE_LENGTH'
      )
    }
  }
}
```

### Transaction Verification

```typescript
function verifyMultisigTransaction(
  transaction: Transaction,
  inputIndex: number,
  multisigScript: Script,
  inputAmount: number
): boolean {
  try {
    // Verify transaction structure
    if (!transaction.inputs || transaction.inputs.length === 0) {
      throw new Error('Transaction has no inputs')
    }
    
    if (!transaction.outputs || transaction.outputs.length === 0) {
      throw new Error('Transaction has no outputs')
    }
    
    // Verify specific input
    const input = transaction.inputs[inputIndex]
    if (!input) {
      throw new Error(`Input at index ${inputIndex} does not exist`)
    }
    
    // Verify unlocking script
    if (!input.unlockingScript) {
      throw new Error('Input has no unlocking script')
    }
    
    // Verify transaction signature
    const isValid = transaction.verify()
    if (!isValid) {
      throw new Error('Transaction signature verification failed')
    }
    
    console.log('Multisig transaction verification passed')
    return true
    
  } catch (error) {
    console.error('Multisig transaction verification failed:', error)
    return false
  }
}
```

## Best Practices

### Key Management

1. **Secure Key Generation**: Always use cryptographically secure random number generation
2. **Key Distribution**: Distribute keys securely and never transmit private keys over insecure channels
3. **Key Storage**: Store private keys in secure hardware or encrypted storage
4. **Key Backup**: Implement proper backup and recovery procedures

```typescript
// Secure key generation example
function generateSecureKeyPair(): { privateKey: PrivateKey, publicKey: PublicKey } {
  const privateKey = PrivateKey.fromRandom()
  const publicKey = privateKey.toPublicKey()
  
  // Validate key pair
  const testMessage = Buffer.from('test message')
  const signature = privateKey.sign(testMessage)
  const isValid = publicKey.verify(testMessage, signature)
  
  if (!isValid) {
    throw new Error('Generated key pair failed validation')
  }
  
  return { privateKey, publicKey }
}
```

### Transaction Construction

1. **Fee Calculation**: Always account for transaction fees
2. **Input Validation**: Validate all inputs before signing
3. **Output Verification**: Verify output amounts and scripts
4. **Signature Ordering**: Maintain consistent signature ordering

```typescript
function calculateMultisigFee(
  inputCount: number,
  outputCount: number,
  threshold: number
): number {
  // Base transaction size
  const baseSize = 10 // version + locktime + input/output counts
  
  // Input size (outpoint + sequence + unlocking script)
  const inputSize = 36 + 4 + (1 + threshold * 73) // Estimated unlocking script size
  
  // Output size (value + locking script)
  const outputSize = 8 + 25 // Estimated P2PKH output size
  
  const totalSize = baseSize + (inputCount * inputSize) + (outputCount * outputSize)
  
  // 1 satoshi per byte
  return totalSize
}
```

### Security Considerations

1. **Signature Verification**: Always verify signatures before broadcasting
2. **Script Validation**: Validate all scripts before use
3. **Amount Verification**: Double-check all amounts
4. **Replay Protection**: Use proper SIGHASH flags

```typescript
function secureMultisigSpend(
  transaction: Transaction,
  inputIndex: number,
  multisigScript: Script,
  inputAmount: number,
  expectedOutputAmount: number
): boolean {
  
  // Verify transaction structure
  if (!verifyMultisigTransaction(transaction, inputIndex, multisigScript, inputAmount)) {
    return false
  }
  
  // Verify output amounts
  const totalOutput = transaction.outputs.reduce((sum, output) => sum + output.satoshis, 0)
  const fee = inputAmount - totalOutput
  
  if (fee < 0) {
    console.error('Invalid transaction: outputs exceed inputs')
    return false
  }
  
  if (fee > inputAmount * 0.1) { // Fee should not exceed 10% of input
    console.error('Warning: High transaction fee detected')
  }
  
  // Verify expected output amount
  if (totalOutput !== expectedOutputAmount) {
    console.error('Output amount mismatch')
    return false
  }
  
  return true
}
```

## Testing Multisig Implementation

### Unit Tests

```typescript
import { describe, it, expect } from '@jest/globals'

describe('MultiSigTemplate', () => {
  let keys: PrivateKey[]
  let pubKeys: PublicKey[]
  let template: MultiSigTemplate
  
  beforeEach(() => {
    keys = [
      PrivateKey.fromRandom(),
      PrivateKey.fromRandom(),
      PrivateKey.fromRandom()
    ]
    pubKeys = keys.map(k => k.toPublicKey())
    template = new MultiSigTemplate(2, pubKeys)
  })
  
  it('should create valid locking script', () => {
    const lockingScript = template.lock()
    expect(lockingScript).toBeDefined()
    expect(lockingScript.toASM()).toContain('OP_CHECKMULTISIG')
  })
  
  it('should create valid unlocking script', () => {
    const tx = new Transaction()
    const signatures = [Buffer.alloc(72), Buffer.alloc(72)]
    
    const unlockingScript = template.unlock(signatures, tx, 0)
    expect(unlockingScript.script).toBeDefined()
    expect(unlockingScript.estimatedLength).toBeGreaterThan(0)
  })
  
  it('should reject invalid threshold', () => {
    expect(() => new MultiSigTemplate(4, pubKeys)).toThrow('Threshold cannot exceed')
    expect(() => new MultiSigTemplate(0, pubKeys)).toThrow('Threshold must be at least 1')
  })
})
```

### Integration Tests

```typescript
describe('Multisig Integration', () => {
  it('should create and spend from multisig', async () => {
    // This would be a full integration test
    // involving actual transaction creation and verification
    
    const keys = [PrivateKey.fromRandom(), PrivateKey.fromRandom(), PrivateKey.fromRandom()]
    const pubKeys = keys.map(k => k.toPublicKey())
    const template = new MultiSigTemplate(2, pubKeys)
    
    // Create mock funding transaction
    const fundingTx = new Transaction()
    // ... setup funding transaction
    
    // Create spending transaction
    const spendingTx = new Transaction()
    // ... setup spending transaction
    
    // Sign with 2 keys
    const signatures = signMultisigTransaction(
      spendingTx,
      0,
      [keys[0], keys[1]],
      template.lock(),
      1000
    )
    
    // Create unlocking script
    const unlockingScript = template.unlock(signatures, spendingTx, 0)
    spendingTx.inputs[0].unlockingScript = unlockingScript.sign()
    
    // Verify transaction
    expect(spendingTx.verify()).toBe(true)
  })
})
```

## Troubleshooting Common Issues

### Signature Ordering Problems

**Problem**: OP_CHECKMULTISIG fails due to incorrect signature ordering.

**Solution**: Ensure signatures are provided in the same order as public keys in the locking script.

```typescript
function orderSignatures(
  signatures: Map<string, Buffer>,
  publicKeys: PublicKey[]
): Buffer[] {
  const orderedSigs: Buffer[] = []
  
  for (const pubKey of publicKeys) {
    const pubKeyHex = pubKey.toString()
    const pubKeyBytes = Array.from(Buffer.from(pubKeyHex, 'hex'))
    if (signatures.has(pubKeyHex)) {
      orderedSigs.push(signatures.get(pubKeyHex)!)
    }
  }
  
  return orderedSigs
}
```

### OP_CHECKMULTISIG Bug

**Problem**: OP_CHECKMULTISIG consumes an extra value from the stack.

**Solution**: Always push OP_0 before signatures in the unlocking script.

```typescript
// Correct unlocking script construction
const script = new Script()
script.writeOpCode(OP.OP_0) // Required for OP_CHECKMULTISIG bug
script.writeBin(signature1)
script.writeBin(signature2)
```

### Fee Calculation Errors

**Problem**: Insufficient fees cause transaction rejection.

**Solution**: Properly calculate fees based on transaction size.

```typescript
function estimateMultisigTxSize(
  inputCount: number,
  outputCount: number,
  threshold: number,
  keyCount: number
): number {
  const baseSize = 10
  const inputSize = 36 + 4 + 1 + (threshold * 73) + 1 // Include OP_0
  const outputSize = 8 + 25
  
  return baseSize + (inputCount * inputSize) + (outputCount * outputSize)
}
```

## Conclusion

Multi-signature transactions provide enhanced security through distributed key management. The BSV TypeScript SDK offers flexible tools for implementing various multisig patterns, from simple 2-of-3 schemes to complex threshold signatures with time locks.

Key takeaways:

- Always validate inputs and signatures
- Implement proper error handling
- Use secure key generation and storage
- Test thoroughly before production use
- Consider the OP_CHECKMULTISIG bug in script construction

For more advanced patterns, consider exploring the SDK's script template system and custom script construction capabilities.

While the `WalletClient` provides simplified transaction creation, understanding multi-signature transactions enables you to build sophisticated applications requiring multiple approvals and enhanced security.
