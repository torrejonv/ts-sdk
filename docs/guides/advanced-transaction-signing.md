# Advanced Transaction Signing

This guide covers advanced transaction signing techniques in the BSV TypeScript SDK, including different signature hash types (SIGHASH flags), manual signature creation, advanced verification methods, and multi-signature transaction patterns.

## Prerequisites

- Understanding of basic transaction construction
- Familiarity with Bitcoin script and cryptographic signatures
- Knowledge of private/public key pairs and digital signatures

## SIGHASH Flags Overview

SIGHASH flags determine which parts of a transaction are included in the signature hash. The BSV TypeScript SDK provides these constants in the `TransactionSignature` class:

```typescript
import { TransactionSignature } from '@bsv/sdk'

// Base signature types
TransactionSignature.SIGHASH_ALL      // 0x01 - Sign all inputs and outputs
TransactionSignature.SIGHASH_NONE     // 0x02 - Sign all inputs, no outputs
TransactionSignature.SIGHASH_SINGLE   // 0x03 - Sign all inputs, one output

// Modifiers
TransactionSignature.SIGHASH_FORKID   // 0x40 - Bitcoin SV fork identifier
TransactionSignature.SIGHASH_ANYONECANPAY // 0x80 - Sign only this input
```

## Understanding SIGHASH Types

### SIGHASH_ALL (Default)

Signs all inputs and outputs, preventing any modifications to the transaction:

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk'

const privateKey = PrivateKey.fromWif('your-private-key-wif')
const p2pkh = new P2PKH()

// Create transaction with SIGHASH_ALL (default)
const tx = new Transaction()
// ... add inputs and outputs

// Sign with SIGHASH_ALL - locks entire transaction
const unlocker = p2pkh.unlock(privateKey, 'all', false)
tx.inputs[0].unlockingScriptTemplate = unlocker
await tx.sign()
```

### SIGHASH_NONE

Signs all inputs but no outputs, allowing outputs to be modified:

```typescript
// SIGHASH_NONE allows outputs to be changed after signing
const unlocker = p2pkh.unlock(privateKey, 'none', false)
tx.inputs[0].unlockingScriptTemplate = unlocker
await tx.sign()

// Outputs can still be modified after signing
tx.addOutput({
  lockingScript: p2pkh.lock(recipientAddress),
  satoshis: 100
})
```

### SIGHASH_SINGLE

Signs all inputs and only the output at the same index as the input:

```typescript
// SIGHASH_SINGLE - signs input[0] and output[0] only
const unlocker = p2pkh.unlock(privateKey, 'single', false)
tx.inputs[0].unlockingScriptTemplate = unlocker
await tx.sign()

// Other outputs can be added/modified
tx.addOutput({
  lockingScript: p2pkh.lock(anotherAddress),
  satoshis: 200
})
```

### SIGHASH_ANYONECANPAY

Modifier that signs only the current input, allowing other inputs to be added:

```typescript
// SIGHASH_ALL | SIGHASH_ANYONECANPAY
const unlocker = p2pkh.unlock(privateKey, 'all', true)
tx.inputs[0].unlockingScriptTemplate = unlocker
await tx.sign()

// Other inputs can be added after signing
tx.addInput({
  sourceTransaction: anotherTx,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: anotherUnlocker
})
```

## Manual Signature Creation

For advanced use cases, you can manually create transaction signatures using the `TransactionSignature.format` method:

```typescript
import { 
  TransactionSignature, 
  PrivateKey, 
  Transaction, 
  LockingScript,
  Hash 
} from '@bsv/sdk'

async function createManualSignature(
  tx: Transaction,
  inputIndex: number,
  privateKey: PrivateKey,
  sighashType: number = TransactionSignature.SIGHASH_ALL | TransactionSignature.SIGHASH_FORKID
): Promise<TransactionSignature> {
  const input = tx.inputs[inputIndex]
  
  // Get source transaction details
  const sourceTXID = input.sourceTXID ?? input.sourceTransaction?.id('hex')
  if (!sourceTXID) {
    throw new Error('Source TXID required for signing')
  }
  
  const sourceSatoshis = input.sourceTransaction?.outputs[input.sourceOutputIndex].satoshis
  if (sourceSatoshis === undefined) {
    throw new Error('Source satoshis required for signing')
  }
  
  const lockingScript = input.sourceTransaction?.outputs[input.sourceOutputIndex].lockingScript
  if (!lockingScript) {
    throw new Error('Locking script required for signing')
  }
  
  // Create other inputs array (excluding current input)
  const otherInputs = tx.inputs.filter((_, index) => index !== inputIndex)
  
  // Create signature hash preimage
  const preimage = TransactionSignature.format({
    sourceTXID,
    sourceOutputIndex: input.sourceOutputIndex,
    sourceSatoshis,
    transactionVersion: tx.version,
    otherInputs,
    inputIndex,
    outputs: tx.outputs,
    inputSequence: input.sequence ?? 0xffffffff,
    subscript: lockingScript,
    lockTime: tx.lockTime,
    scope: sighashType
  })
  
  // Hash the preimage and sign
  const preimageHash = Hash.sha256(preimage)
  const rawSignature = privateKey.sign(preimageHash)
  
  // Create TransactionSignature with scope
  return new TransactionSignature(
    rawSignature.r,
    rawSignature.s,
    sighashType
  )
}

// Usage example
const signature = await createManualSignature(tx, 0, privateKey)
const sigBytes = signature.toChecksigFormat()
console.log('Signature:', Buffer.from(sigBytes).toString('hex'))
```

## Advanced Signature Verification

Verify signatures manually for complex validation scenarios:

```typescript
import { PublicKey, Hash } from '@bsv/sdk'

async function verifyTransactionSignature(
  tx: Transaction,
  inputIndex: number,
  signature: TransactionSignature,
  publicKey: PublicKey
): Promise<boolean> {
  const input = tx.inputs[inputIndex]
  
  // Recreate the signature hash preimage
  const sourceTXID = input.sourceTXID ?? input.sourceTransaction?.id('hex')
  const sourceSatoshis = input.sourceTransaction?.outputs[input.sourceOutputIndex].satoshis
  const lockingScript = input.sourceTransaction?.outputs[input.sourceOutputIndex].lockingScript
  
  if (!sourceTXID || sourceSatoshis === undefined || !lockingScript) {
    return false
  }
  
  const otherInputs = tx.inputs.filter((_, index) => index !== inputIndex)
  
  const preimage = TransactionSignature.format({
    sourceTXID,
    sourceOutputIndex: input.sourceOutputIndex,
    sourceSatoshis,
    transactionVersion: tx.version,
    otherInputs,
    inputIndex,
    outputs: tx.outputs,
    inputSequence: input.sequence ?? 0xffffffff,
    subscript: lockingScript,
    lockTime: tx.lockTime,
    scope: signature.scope
  })
  
  // Verify signature against preimage hash
  const preimageHash = Hash.sha256(preimage)
  return signature.verify(preimageHash, publicKey)
}

// Usage example
const isValid = await verifyTransactionSignature(tx, 0, signature, publicKey)
console.log('Signature valid:', isValid)
```

## Custom Script Template with Advanced Signing

Create a custom script template that demonstrates advanced signing patterns:

```typescript
import { 
  ScriptTemplate, 
  LockingScript, 
  UnlockingScript,
  Transaction,
  PrivateKey,
  TransactionSignature,
  Hash,
  OP
} from '@bsv/sdk'

class CustomSigningTemplate implements ScriptTemplate {
  private privateKey: PrivateKey
  private sighashType: number
  
  constructor(
    privateKey: PrivateKey, 
    sighashType: number = TransactionSignature.SIGHASH_ALL | TransactionSignature.SIGHASH_FORKID
  ) {
    this.privateKey = privateKey
    this.sighashType = sighashType
  }
  
  lock(publicKeyHash: number[]): LockingScript {
    // Custom locking script: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
    // Note: For standard P2PKH scripts, use the built-in P2PKH class:
    // import { P2PKH } from '@bsv/sdk'
    // const p2pkh = new P2PKH()
    // return p2pkh.lock(publicKeyHash)
    
    // This example shows manual construction for educational purposes:
    return new LockingScript([
      { op: OP.OP_DUP },
      { op: OP.OP_HASH160 },
      { op: publicKeyHash.length, data: publicKeyHash },
      { op: OP.OP_EQUALVERIFY },
      { op: OP.OP_CHECKSIG }
    ])
  }
  
  unlock(): {
    sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>
    estimateLength: () => Promise<number>
  } {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        const input = tx.inputs[inputIndex]
        const sourceTXID = input.sourceTXID ?? input.sourceTransaction?.id('hex')
        const sourceSatoshis = input.sourceTransaction?.outputs[input.sourceOutputIndex].satoshis
        const lockingScript = input.sourceTransaction?.outputs[input.sourceOutputIndex].lockingScript
        
        if (!sourceTXID || sourceSatoshis === undefined || !lockingScript) {
          throw new Error('Missing required input data for signing')
        }
        
        const otherInputs = tx.inputs.filter((_, index) => index !== inputIndex)
        
        // Create signature with custom SIGHASH type
        const preimage = TransactionSignature.format({
          sourceTXID,
          sourceOutputIndex: input.sourceOutputIndex,
          sourceSatoshis,
          transactionVersion: tx.version,
          otherInputs,
          inputIndex,
          outputs: tx.outputs,
          inputSequence: input.sequence ?? 0xffffffff,
          subscript: lockingScript,
          lockTime: tx.lockTime,
          scope: this.sighashType
        })
        
        const rawSignature = this.privateKey.sign(Hash.sha256(preimage))
        const signature = new TransactionSignature(
          rawSignature.r,
          rawSignature.s,
          this.sighashType
        )
        
        const sigBytes = signature.toChecksigFormat()
        const pubKeyBytes = this.privateKey.toPublicKey().encode(true)
        
        return new UnlockingScript([
          { op: sigBytes.length, data: sigBytes as unknown as number[] },
          { op: pubKeyBytes.length, data: pubKeyBytes as unknown as number[] }
        ])
      },
      
      estimateLength: async () => 108 // signature + pubkey + opcodes
    }
  }
}

// Usage example
const customTemplate = new CustomSigningTemplate(
  privateKey,
  TransactionSignature.SIGHASH_SINGLE | TransactionSignature.SIGHASH_FORKID
)

const tx = new Transaction()
tx.addInput({
  sourceTransaction: sourceTx,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: customTemplate.unlock()
})
```

## Multi-Signature Advanced Patterns

For complex multi-signature scenarios, see the [Multi-Signature Transactions Guide](./multisig-transactions.md). Here's an advanced pattern for threshold signatures:

```typescript
import { PrivateKey, PublicKey, TransactionSignature } from '@bsv/sdk'

class ThresholdSigner {
  private requiredSignatures: number
  private privateKeys: PrivateKey[]
  
  constructor(requiredSignatures: number, privateKeys: PrivateKey[]) {
    this.requiredSignatures = requiredSignatures
    this.privateKeys = privateKeys
  }
  
  async signTransaction(
    tx: Transaction,
    inputIndex: number,
    sighashType: number = TransactionSignature.SIGHASH_ALL | TransactionSignature.SIGHASH_FORKID
  ): Promise<TransactionSignature[]> {
    const signatures: TransactionSignature[] = []
    
    // Sign with required number of keys
    for (let i = 0; i < this.requiredSignatures && i < this.privateKeys.length; i++) {
      const signature = await createManualSignature(
        tx, 
        inputIndex, 
        this.privateKeys[i], 
        sighashType
      )
      signatures.push(signature)
    }
    
    return signatures
  }
}

// Usage
const thresholdSigner = new ThresholdSigner(2, [privateKey1, privateKey2, privateKey3])
const signatures = await thresholdSigner.signTransaction(tx, 0)
```

## Signature Hash Analysis

Analyze what parts of a transaction are covered by different SIGHASH types:

```typescript
function analyzeSighashCoverage(sighashType: number): {
  inputs: string
  outputs: string
  modifier: string
} {
  const baseType = sighashType & 0x1f
  const hasAnyoneCanPay = (sighashType & TransactionSignature.SIGHASH_ANYONECANPAY) !== 0
  
  let inputs: string
  let outputs: string
  
  if (hasAnyoneCanPay) {
    inputs = 'Current input only'
  } else {
    inputs = 'All inputs'
  }
  
  switch (baseType) {
    case TransactionSignature.SIGHASH_ALL:
      outputs = 'All outputs'
      break
    case TransactionSignature.SIGHASH_NONE:
      outputs = 'No outputs'
      break
    case TransactionSignature.SIGHASH_SINGLE:
      outputs = 'Corresponding output only'
      break
    default:
      outputs = 'Unknown'
  }
  
  return {
    inputs,
    outputs,
    modifier: hasAnyoneCanPay ? 'ANYONECANPAY' : 'Standard'
  }
}

// Usage
const coverage = analyzeSighashCoverage(
  TransactionSignature.SIGHASH_SINGLE | TransactionSignature.SIGHASH_ANYONECANPAY
)
console.log('Signature covers:', coverage)
// Output: { inputs: 'Current input only', outputs: 'Corresponding output only', modifier: 'ANYONECANPAY' }
```

## Best Practices

### Security Considerations

1. **Always use SIGHASH_FORKID**: Required for Bitcoin SV transactions
2. **Validate inputs**: Ensure all required transaction data is present before signing
3. **Use appropriate SIGHASH types**: Choose the most restrictive type that meets your needs
4. **Verify signatures**: Always verify signatures in critical applications

```typescript
// Secure signing pattern
function secureSign(
  tx: Transaction,
  inputIndex: number,
  privateKey: PrivateKey,
  sighashType: number = TransactionSignature.SIGHASH_ALL | TransactionSignature.SIGHASH_FORKID
): Promise<TransactionSignature> {
  // Validate SIGHASH_FORKID is present
  if ((sighashType & TransactionSignature.SIGHASH_FORKID) === 0) {
    throw new Error('SIGHASH_FORKID is required for BSV transactions')
  }
  
  // Validate input exists
  if (inputIndex >= tx.inputs.length) {
    throw new Error('Input index out of range')
  }
  
  return createManualSignature(tx, inputIndex, privateKey, sighashType)
}
```

## Error Handling

Handle common signing errors gracefully:

```typescript
async function robustSign(
  tx: Transaction,
  inputIndex: number,
  privateKey: PrivateKey
): Promise<TransactionSignature> {
  try {
    return await createManualSignature(tx, inputIndex, privateKey)
  } catch (error) {
    if (error.message.includes('Source TXID required')) {
      throw new Error('Transaction input missing source transaction reference')
    }
    if (error.message.includes('Source satoshis required')) {
      throw new Error('Cannot determine input value for signature creation')
    }
    if (error.message.includes('Locking script required')) {
      throw new Error('Source output locking script not available')
    }
    
    // Re-throw unknown errors
    throw error
  }
}
```

## Related Resources

- [Transaction Signing Methods](./transaction-signing-methods.md) - Basic signing approaches
- [Multi-Signature Transactions](./multisig-transactions.md) - Multi-signature implementation
- [Transaction Signatures Reference](../reference/transaction-signatures.md) - Detailed API reference
- [Signatures Concepts](../concepts/signatures.md) - Conceptual overview

## Summary

This guide covered advanced transaction signing techniques including:

- Understanding and using different SIGHASH flags
- Manual signature creation and verification
- Custom script templates with advanced signing
- Multi-signature patterns and threshold signing
- Security best practices and error handling

These advanced techniques provide fine-grained control over transaction signing behavior, enabling sophisticated Bitcoin applications with complex signing requirements.
