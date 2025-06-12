# Script Templates

Standard and custom Bitcoin script patterns available in the BSV TypeScript SDK.

## What are Script Templates?

Script templates provide reusable patterns for Bitcoin transaction locking and unlocking scripts:

```typescript
import { P2PKH } from '@bsv/sdk'

// Use a standard template
const template = new P2PKH()

// Create locking script
const lockingScript = template.lock(publicKeyHash)

// Create unlocking script
const unlockingScript = template.unlock(privateKey, signature)
```

## Standard Templates

### P2PKH (Pay to Public Key Hash)
The most common Bitcoin script pattern:
```typescript
const p2pkh = new P2PKH()
const lock = p2pkh.lock(publicKeyHash)
const unlock = p2pkh.unlock(privateKey, signature)
```

### P2PK (Pay to Public Key)
Direct payment to a public key:
```typescript
const p2pk = new P2PK()
const lock = p2pk.lock(publicKey)
const unlock = p2pk.unlock(signature)
```

### OP_RETURN (Data Storage)
Store arbitrary data on-chain:
```typescript
const opReturn = new OpReturn()
const lock = opReturn.lock(data)
// No unlock needed - unspendable output
```

## Custom Templates

Create your own script templates:

```typescript
class TimeLockTemplate implements ScriptTemplate {
  lock(lockTime: number, publicKeyHash: string): Script {
    return new Script()
      .writeNumber(lockTime)
      .writeOpCode(OpCode.OP_CHECKLOCKTIMEVERIFY)
      .writeOpCode(OpCode.OP_DROP)
      .writeOpCode(OpCode.OP_DUP)
      .writeOpCode(OpCode.OP_HASH160)
      .writeBin(publicKeyHash)
      .writeOpCode(OpCode.OP_EQUALVERIFY)
      .writeOpCode(OpCode.OP_CHECKSIG)
  }

  unlock(signature: string, publicKey: string): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey)
  }
}
```

## Template Interface

All templates implement the ScriptTemplate interface:

```typescript
interface ScriptTemplate {
  lock(...args: any[]): Script
  unlock(...args: any[]): Script
  estimateLength?(args: any[]): number
}
```

## Benefits

### Reusability
- Standard patterns for common use cases
- Consistent implementation across applications
- Reduced development time

### Security
- Well-tested script patterns
- Reduced risk of script errors
- Best practice implementations

### Maintainability
- Clear separation of script logic
- Easy to update and modify
- Testable components

## Working with Templates

### Transaction Integration
```typescript
// Use template in transaction
const output = {
  satoshis: 1000,
  lockingScript: template.lock(publicKeyHash)
}

const input = {
  sourceTransaction: prevTx,
  sourceOutputIndex: 0,
  unlockingScript: template.unlock(privateKey, signature)
}
```

### Fee Estimation
```typescript
// Estimate script size for fee calculation
const estimatedSize = template.estimateLength([publicKeyHash])
const fee = estimatedSize * feePerByte
```

## Advanced Patterns

### Multi-Signature
```typescript
class MultiSigTemplate implements ScriptTemplate {
  lock(threshold: number, publicKeys: string[]): Script {
    // Implementation for m-of-n multisig
  }
  
  unlock(signatures: string[]): Script {
    // Implementation for signature collection
  }
}
```

### Conditional Scripts
```typescript
class ConditionalTemplate implements ScriptTemplate {
  lock(condition: Script, trueScript: Script, falseScript: Script): Script {
    // Implementation for IF/ELSE logic
  }
}
```

## Best Practices

- Use standard templates when possible
- Test custom templates thoroughly
- Document template parameters clearly
- Consider script size and complexity
- Validate inputs before script creation

## Next Steps

- Learn about [Digital Signatures](./signatures.md) for script authorization
- Understand [Transaction Structure](./transaction-structure.md) integration
- Explore [Key Management](./key-management.md) for script security
