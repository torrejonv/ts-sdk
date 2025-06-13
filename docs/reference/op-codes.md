# OP Codes

*Understanding Bitcoin script opcodes and their usage in the BSV TypeScript SDK.*

## What are OP Codes?

OP codes (operation codes) are the fundamental building blocks of Bitcoin's scripting language. They are operational instructions that manipulate data on the script execution stack to define the conditions under which Bitcoin can be spent. Each opcode performs a specific function such as arithmetic operations, logical comparisons, cryptographic operations, or stack manipulation.

## Script Execution Environment

Bitcoin scripts operate in a **stack-based execution environment** with two primary data structures:

- **Main Stack**: Where most operations are performed
- **Alt Stack**: Provides additional stack flexibility for complex operations

Scripts consist of two parts:
1. **Unlocking Script**: Provided by the spender, supplies data to satisfy locking conditions
2. **Locking Script**: Defines the conditions that must be met to spend the output

The execution begins with the unlocking script placing data on the stack, followed by the locking script operating on the same stack. A spend is valid if the top of the stack contains a "true" value (non-zero) after execution.

## SDK Integration

The BSV TypeScript SDK provides comprehensive opcode support through the `OP` object and script execution engine:

```typescript
import { OP, Script, LockingScript, UnlockingScript } from '@bsv/sdk'

// Access opcodes by name
console.log(OP.OP_DUP)        // 0x76
console.log(OP.OP_HASH160)    // 0xa9
console.log(OP.OP_CHECKSIG)   // 0xac

// Create scripts using opcodes
const lockingScript = new LockingScript([
  { op: OP.OP_DUP },
  { op: OP.OP_HASH160 },
  { op: 20, data: publicKeyHash }, // Push 20 bytes
  { op: OP.OP_EQUALVERIFY },
  { op: OP.OP_CHECKSIG }
])

// Create from ASM (Assembly) format
const script = Script.fromASM('OP_DUP OP_HASH160 ' + publicKeyHash + ' OP_EQUALVERIFY OP_CHECKSIG')
```

## Opcode Categories

### Push Operations
Push data onto the stack:

```typescript
// Push small numbers (0-16)
OP.OP_0        // Push empty array (false)
OP.OP_1        // Push number 1 (true)
OP.OP_2        // Push number 2
// ... up to OP_16

// Push data of various sizes
OP.OP_PUSHDATA1  // Push up to 255 bytes
OP.OP_PUSHDATA2  // Push up to 65,535 bytes  
OP.OP_PUSHDATA4  // Push up to 4,294,967,295 bytes
```

### Stack Operations
Manipulate stack contents:

```typescript
OP.OP_DUP        // Duplicate top stack item
OP.OP_DROP       // Remove top stack item
OP.OP_SWAP       // Swap top two stack items
OP.OP_ROT        // Rotate top three stack items
OP.OP_2DUP       // Duplicate top two stack items
OP.OP_TOALTSTACK // Move item to alt stack
```

### Arithmetic Operations
Perform mathematical operations:

```typescript
OP.OP_ADD        // Add top two stack items
OP.OP_SUB        // Subtract second from top
OP.OP_MUL        // Multiply top two items
OP.OP_DIV        // Divide second by top
OP.OP_MOD        // Modulo operation
OP.OP_MIN        // Return minimum of two values
OP.OP_MAX        // Return maximum of two values
```

### Comparison Operations
Compare values and return boolean results:

```typescript
OP.OP_EQUAL         // Check if top two items are equal
OP.OP_EQUALVERIFY   // Equal check + verify (fail if false)
OP.OP_NUMEQUAL      // Numeric equality check
OP.OP_LESSTHAN      // Check if second < top
OP.OP_GREATERTHAN   // Check if second > top
OP.OP_WITHIN        // Check if value is within range
```

### Cryptographic Operations
Perform cryptographic functions:

```typescript
OP.OP_SHA1          // SHA-1 hash
OP.OP_SHA256        // SHA-256 hash
OP.OP_HASH160       // RIPEMD160(SHA256(x))
OP.OP_HASH256       // SHA256(SHA256(x))
OP.OP_CHECKSIG      // Verify signature
OP.OP_CHECKMULTISIG // Verify multiple signatures
```

### Control Flow Operations
Control script execution:

```typescript
OP.OP_IF         // Conditional execution
OP.OP_ELSE       // Alternative branch
OP.OP_ENDIF      // End conditional
OP.OP_VERIFY     // Fail if top stack item is false
OP.OP_RETURN     // Mark output as unspendable
```

## Common Script Patterns

### Pay-to-Public-Key-Hash (P2PKH)
The most common Bitcoin script pattern:

```typescript
// Locking script: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
const p2pkhLock = new LockingScript([
  { op: OP.OP_DUP },
  { op: OP.OP_HASH160 },
  { op: 20, data: publicKeyHash },
  { op: OP.OP_EQUALVERIFY },
  { op: OP.OP_CHECKSIG }
])

// Unlocking script: <signature> <publicKey>
const p2pkhUnlock = new UnlockingScript([
  { op: signature.length, data: signature },
  { op: publicKey.length, data: publicKey }
])
```

### Data Storage Script
Store arbitrary data on the blockchain:

```typescript
// OP_RETURN <data>
const dataScript = new LockingScript([
  { op: OP.OP_RETURN },
  { op: data.length, data: data }
])
```

### Multi-Signature Script
Require multiple signatures:

```typescript
// 2-of-3 multisig: OP_2 <pubKey1> <pubKey2> <pubKey3> OP_3 OP_CHECKMULTISIG
const multisigScript = new LockingScript([
  { op: OP.OP_2 },
  { op: pubKey1.length, data: pubKey1 },
  { op: pubKey2.length, data: pubKey2 },
  { op: pubKey3.length, data: pubKey3 },
  { op: OP.OP_3 },
  { op: OP.OP_CHECKMULTISIG }
])
```


## Security Considerations

### Disabled Opcodes
Some opcodes are disabled for security reasons:

```typescript
// These opcodes will cause script failure
OP.OP_2MUL      // Disabled: multiply by 2
OP.OP_2DIV      // Disabled: divide by 2  
OP.OP_VERIF     // Disabled: conditional verification
OP.OP_VERNOTIF  // Disabled: inverse conditional verification
```

## Best Practices

### 1. Use Script Templates
Leverage SDK script templates for common patterns:

```typescript
import { P2PKH, MultiSig, RPuzzle } from '@bsv/sdk'

// Use templates instead of manual opcode construction
const p2pkh = new P2PKH()
const lockingScript = p2pkh.lock(publicKeyHash)
```

### 2. Validate Scripts
Always validate scripts before use:

```typescript
try {
  const script = Script.fromASM(asmString)
  // Script is valid
} catch (error) {
  console.error('Invalid script:', error.message)
}
```

### 3. Handle Execution Errors
Implement proper error handling:

```typescript
const spend = new Spend(params)
try {
  while (!spend.isFinished()) {
    if (!spend.step()) {
      throw new Error(`Script execution failed: ${spend.getDebugString()}`)
    }
  }
} catch (error) {
  console.error('Script execution error:', error.message)
}
```

## Common Use Cases

### 1. Payment Scripts
Standard payment to public key hash:

```typescript
const paymentScript = Script.fromASM(`OP_DUP OP_HASH160 ${pubKeyHash} OP_EQUALVERIFY OP_CHECKSIG`)
```

### 2. Data Storage
Store application data on-chain:

```typescript
const dataScript = Script.fromASM(`OP_RETURN ${Buffer.from(jsonData).toString('hex')}`)
```

### 3. Smart Contracts
Create conditional spending logic:

```typescript
const contractScript = Script.fromASM(`
  OP_IF
    ${timelock} OP_CHECKLOCKTIMEVERIFY OP_DROP
    OP_DUP OP_HASH160 ${ownerHash} OP_EQUALVERIFY OP_CHECKSIG
  OP_ELSE
    OP_DUP OP_HASH160 ${beneficiaryHash} OP_EQUALVERIFY OP_CHECKSIG
  OP_ENDIF
`)
```

### 4. Puzzle Scripts
Create cryptographic puzzles:

```typescript
// Hash puzzle: provide preimage to unlock
const puzzleScript = Script.fromASM(`OP_HASH256 ${targetHash} OP_EQUAL`)
```

## Debugging Scripts

### Script Execution Tracing
Monitor script execution step by step:

```typescript
const spend = new Spend(params)
console.log('Initial stack:', spend.getDebugString())

while (!spend.isFinished()) {
  const success = spend.step()
  console.log('After step:', spend.getDebugString())
  
  if (!success) {
    console.log('Execution failed at:', spend.getCurrentOpcode())
    break
  }
}
```

### Common Debugging Patterns
Identify and fix common script issues:

```typescript
// Check stack state
if (spend.getStackSize() === 0) {
  console.log('Stack is empty - missing data?')
}

// Check for script errors
if (spend.hasError()) {
  console.log('Script error:', spend.getErrorMessage())
}

// Verify final state
if (spend.isSuccess() && spend.getStackTop() !== 1) {
  console.log('Script succeeded but top stack value is not true')
}
```

Understanding OP codes enables you to create sophisticated Bitcoin applications with custom spending conditions, smart contracts, and advanced cryptographic protocols using the BSV TypeScript SDK.
