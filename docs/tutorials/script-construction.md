# Script Construction and Custom Logic

Learn how to create, manipulate, and implement custom Bitcoin scripts using the BSV TypeScript SDK. This tutorial covers both basic script operations and advanced script template creation.

## Prerequisites

- Completed "Your First BSV Transaction" tutorial
- Basic understanding of Bitcoin script operations
- Node.js and TypeScript knowledge

> **📚 Related Concepts**: Review [Script Templates](../concepts/script-templates.md) and [Transaction Structure](../concepts/transaction-structure.md) for foundational understanding. See [OP Codes Reference](../reference/op-codes.md) for complete opcode documentation.

## Learning Goals

- Understand Bitcoin script fundamentals
- Create and serialize scripts in different formats
- Build custom script templates
- Implement locking and unlocking logic
- Work with opcodes and script chunks

> **💡 Try It Interactive**: Test script construction and custom templates in our [Interactive BSV Coding Environment](https://fast.brc.dev/) - experiment with opcodes and script logic in real-time!

## Duration

60 minutes

---

## Part 1: Script Fundamentals

### Understanding Bitcoin Scripts

Bitcoin scripts are small programs that define the conditions under which bitcoins can be spent. They consist of:

- **Locking Scripts**: Define spending conditions (found in transaction outputs)
- **Unlocking Scripts**: Provide evidence to satisfy conditions (found in transaction inputs)

### Setting Up

First, install the SDK and import the necessary modules:

```bash
npm install @bsv/sdk
```

```typescript
import { 
  Script, 
  LockingScript, 
  UnlockingScript, 
  OP, 
  PrivateKey, 
  P2PKH,
  ScriptTemplate,
  Transaction,
  Hash
} from '@bsv/sdk'
```

### Creating Scripts from Different Formats

The SDK supports creating scripts from various formats:

#### From ASM (Assembly)

```typescript
// Create a P2PKH script from ASM
const scriptFromASM = Script.fromASM(
  'OP_DUP OP_HASH160 1451baa3aad777144a0759998a03538018dd7b4b OP_EQUALVERIFY OP_CHECKSIG'
)

console.log('Script from ASM:', scriptFromASM.toHex())
```

#### From Hexadecimal

```typescript
// Create script from hex
const scriptFromHex = Script.fromHex('76a9141451baa3aad777144a0759998a03538018dd7b4b88ac')

console.log('Script from hex:', scriptFromHex.toASM())
```

#### From Binary Array

```typescript
// Create script from binary data
const binaryData = [OP.OP_TRUE, OP.OP_RETURN, 4, 0x74, 0x65, 0x73, 0x74]
const scriptFromBinary = Script.fromBinary(binaryData)

console.log('Script from binary:', scriptFromBinary.toASM())
```

### Script Serialization

Convert scripts between different formats for storage or transmission:

```typescript
const script = Script.fromASM('OP_DUP OP_HASH160 1451baa3aad777144a0759998a03538018dd7b4b OP_EQUALVERIFY OP_CHECKSIG')

// Serialize to different formats
const scriptAsHex = script.toHex()
const scriptAsASM = script.toASM()
const scriptAsBinary = script.toBinary()

console.log('Hex:', scriptAsHex)
console.log('ASM:', scriptAsASM)
console.log('Binary length:', scriptAsBinary.length)
```

---

## Part 2: Working with Script Chunks

### Understanding Script Chunks

Scripts are composed of chunks, each containing either an opcode or data:

```typescript
// Create a script with mixed opcodes and data
const script = new Script([
  { op: OP.OP_DUP },
  { op: OP.OP_HASH160 },
  { op: 20, data: [0x14, 0x51, 0xba, 0xa3, 0xaa, 0xd7, 0x77, 0x14, 0x4a, 0x07, 0x59, 0x99, 0x8a, 0x03, 0x53, 0x80, 0x18, 0xdd, 0x7b, 0x4b] },
  { op: OP.OP_EQUALVERIFY },
  { op: OP.OP_CHECKSIG }
])

console.log('Script chunks:', script.chunks.length)
console.log('Script ASM:', script.toASM())
```

### Building Scripts Programmatically

```typescript
// Build a script step by step
const script = new Script()

// Add opcodes
script.writeOpCode(OP.OP_DUP)
script.writeOpCode(OP.OP_HASH160)

// Add data
const pubkeyHash = [0x14, 0x51, 0xba, 0xa3, 0xaa, 0xd7, 0x77, 0x14, 0x4a, 0x07, 0x59, 0x99, 0x8a, 0x03, 0x53, 0x80, 0x18, 0xdd, 0x7b, 0x4b]
script.writeBin(pubkeyHash)

// Add more opcodes
script.writeOpCode(OP.OP_EQUALVERIFY)
script.writeOpCode(OP.OP_CHECKSIG)

console.log('Built script:', script.toASM())
```

### Adding Numbers and Data

```typescript
const script = new Script()

// Add numbers (automatically encoded)
script.writeNumber(42)
script.writeNumber(1000)

// Add binary data
script.writeBin([0x48, 0x65, 0x6c, 0x6c, 0x6f]) // "Hello"

// Add another script
const dataScript = Script.fromASM('OP_TRUE OP_FALSE')
script.writeScript(dataScript)

console.log('Data script:', script.toASM())
```

---

## Part 3: Standard Script Templates

### Using P2PKH Template

The P2PKH (Pay to Public Key Hash) template is the most common script type:

```typescript
async function runP2PKHExample() {
  // Generate a key pair
  const privateKey = PrivateKey.fromRandom()
  const publicKey = privateKey.toPublicKey()
  const pubkeyHash = publicKey.toHash()

  // Create P2PKH template
  const p2pkh = new P2PKH()

  // Create locking script
  const lockingScript = p2pkh.lock(pubkeyHash)
  console.log('P2PKH locking script:', lockingScript.toASM())

  // Create unlocking script
  const unlockingTemplate = p2pkh.unlock(privateKey)
  console.log('Unlocking script estimate:', await unlockingTemplate.estimateLength())
}

// Run the example
runP2PKHExample().catch(console.error)
```

### Data Storage Scripts

Create scripts that store arbitrary data:

```typescript
// Create a simple data storage script
const dataScript = new Script()
dataScript.writeOpCode(OP.OP_FALSE)
dataScript.writeOpCode(OP.OP_RETURN)
dataScript.writeBin([0x48, 0x65, 0x6c, 0x6c, 0x6f, 0x20, 0x57, 0x6f, 0x72, 0x6c, 0x64]) // "Hello World"

console.log('Data script:', dataScript.toASM())
console.log('Data script hex:', dataScript.toHex())
```

---

## Part 4: Custom Script Templates

### Creating a Simple Custom Template

Let's create a custom script template for a simple puzzle:

```typescript
import { ScriptTemplate, LockingScript, UnlockingScript, OP, Transaction } from '@bsv/sdk'

class SimplePuzzle implements ScriptTemplate {
  /**
   * Creates a locking script that requires a specific number to unlock
   */
  lock(secretNumber: number): LockingScript {
    const script = new LockingScript()
    script.writeNumber(secretNumber)
    script.writeOpCode(OP.OP_EQUAL)
    return script
  }

  /**
   * Creates an unlocking script with the secret number
   */
  unlock(secretNumber: number) {
    return {
      sign: async (tx: Transaction, inputIndex: number): Promise<UnlockingScript> => {
        const script = new UnlockingScript()
        script.writeNumber(secretNumber)
        return script
      },
      estimateLength: async (): Promise<number> => {
        // Estimate: number encoding (1-5 bytes) + push opcode (1 byte)
        return 6
      }
    }
  }
}

async function runSimplePuzzleExample() {
  // Usage example
  const puzzle = new SimplePuzzle()
  const secretNumber = 42

  const lockingScript = puzzle.lock(secretNumber)
  console.log('Puzzle locking script:', lockingScript.toASM())

  const unlockingTemplate = puzzle.unlock(secretNumber)
  console.log('Estimated unlock length:', await unlockingTemplate.estimateLength())
}

// Run the example
runSimplePuzzleExample().catch(console.error)
```

### Advanced Custom Template: Hash Puzzle

Create a more sophisticated template that uses hash functions:

```typescript
class HashPuzzle implements ScriptTemplate {
  /**
   * Creates a locking script that requires the preimage of a hash
   */
  lock(hash: number[]): LockingScript {
    const script = new LockingScript()
    script.writeOpCode(OP.OP_SHA256)
    script.writeBin(hash)
    script.writeOpCode(OP.OP_EQUAL)
    return script
  }

  /**
   * Creates an unlocking script with the preimage
   */
  unlock(preimage: number[]) {
    return {
      sign: async (tx: Transaction, inputIndex: number): Promise<UnlockingScript> => {
        const script = new UnlockingScript()
        script.writeBin(preimage)
        return script
      },
      estimateLength: async (): Promise<number> => {
        // Estimate: preimage length + push opcodes
        return preimage.length + 5
      }
    }
  }
}

async function runHashPuzzleExample() {
  // Usage example
  const hashPuzzle = new HashPuzzle()
  const preimage = [0x48, 0x65, 0x6c, 0x6c, 0x6f] // "Hello"
  const hash = Hash.sha256(preimage)

  const lockingScript = hashPuzzle.lock(hash)
  console.log('Hash puzzle locking script:', lockingScript.toASM())

  const unlockingTemplate = hashPuzzle.unlock(preimage)
  console.log('Hash puzzle unlock estimate:', await unlockingTemplate.estimateLength())
}

// Run the example
runHashPuzzleExample().catch(console.error)
```

---

## Part 5: Multi-Signature Scripts

### Creating Multi-Sig Templates

```typescript
class MultiSig implements ScriptTemplate {
  /**
   * Creates a multi-signature locking script
   */
  lock(requiredSigs: number, publicKeys: number[][]): LockingScript {
    const script = new LockingScript()
    
    // Required signatures count
    script.writeNumber(requiredSigs)
    
    // Add public keys
    for (const pubkey of publicKeys) {
      script.writeBin(pubkey)
    }
    
    // Total public keys count
    script.writeNumber(publicKeys.length)
    script.writeOpCode(OP.OP_CHECKMULTISIG)
    
    return script
  }

  /**
   * Creates an unlocking script for multi-sig
   */
  unlock(signatures: number[][]) {
    return {
      sign: async (tx: Transaction, inputIndex: number): Promise<UnlockingScript> => {
        const script = new UnlockingScript()
        
        // OP_0 due to CHECKMULTISIG bug
        script.writeOpCode(OP.OP_0)
        
        // Add signatures
        for (const sig of signatures) {
          script.writeBin(sig)
        }
        
        return script
      },
      estimateLength: async (): Promise<number> => {
        // Estimate: OP_0 + signatures with push opcodes
        return 1 + signatures.reduce((total, sig) => total + sig.length + 1, 0)
      }
    }
  }
}

async function runMultiSigExample() {
  // Usage example
  const multiSig = new MultiSig()
  const requiredSigs = 2
  const publicKeys = [
    [0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18, 0x19, 0x20, 0x21, 0x22, 0x23, 0x24, 0x25, 0x26, 0x27, 0x28, 0x29, 0x30, 0x31, 0x32, 0x33],
    [0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x40, 0x41, 0x42, 0x43, 0x44, 0x45, 0x46, 0x47, 0x48, 0x49, 0x50, 0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57, 0x58, 0x59, 0x60, 0x61, 0x62, 0x63, 0x64, 0x65]
  ]

  const lockingScript = multiSig.lock(requiredSigs, publicKeys)
  console.log('Multi-sig locking script:', lockingScript.toASM())

  const unlockingTemplate = multiSig.unlock([
    [0x66, 0x67, 0x68, 0x69, 0x70, 0x71, 0x72, 0x73, 0x74, 0x75, 0x76, 0x77, 0x78, 0x79, 0x80, 0x81, 0x82, 0x83, 0x84, 0x85, 0x86, 0x87, 0x88, 0x89, 0x90, 0x91, 0x92, 0x93, 0x94, 0x95, 0x96, 0x97],
    [0x98, 0x99, 0x100, 0x101, 0x102, 0x103, 0x104, 0x105, 0x106, 0x107, 0x108, 0x109, 0x110, 0x111, 0x112, 0x113, 0x114, 0x115, 0x116, 0x117, 0x118, 0x119, 0x120, 0x121, 0x122, 0x123, 0x124, 0x125, 0x126, 0x127, 0x128, 0x129]
  ])
  console.log('Multi-sig unlock estimate:', await unlockingTemplate.estimateLength())
}

// Run the example
runMultiSigExample().catch(console.error)
```

---

## Part 6: Script Validation and Testing

### Testing Script Templates

```typescript
// Test the simple puzzle template
async function testSimplePuzzle() {
  const puzzle = new SimplePuzzle()
  const secretNumber = 123
  
  // Create locking script
  const lockingScript = puzzle.lock(secretNumber)
  
  // Create unlocking script
  const unlockingTemplate = puzzle.unlock(secretNumber)
  
  // In a real scenario, you would use this with a transaction
  console.log('Puzzle test:')
  console.log('Locking script:', lockingScript.toASM())
  console.log('Estimated unlock length:', await unlockingTemplate.estimateLength())
  
  // Test with wrong number
  const wrongUnlock = puzzle.unlock(456)
  console.log('Wrong unlock estimate:', await wrongUnlock.estimateLength())
}

testSimplePuzzle()
```

### Script Analysis

```typescript
// Analyze script properties
function analyzeScript(script: Script) {
  console.log('Script Analysis:')
  console.log('- Chunks:', script.chunks.length)
  console.log('- ASM:', script.toASM())
  console.log('- Hex:', script.toHex())
  console.log('- Binary length:', script.toBinary().length)
  console.log('- Is push-only:', script.isPushOnly())
}

const testScript = Script.fromASM('OP_DUP OP_HASH160 1451baa3aad777144a0759998a03538018dd7b4b OP_EQUALVERIFY OP_CHECKSIG')
analyzeScript(testScript)
```

---

## Part 7: Advanced Script Operations

### Script Manipulation

```typescript
// Modify existing scripts
const script = Script.fromASM('OP_DUP OP_HASH160 1451baa3aad777144a0759998a03538018dd7b4b OP_EQUALVERIFY OP_CHECKSIG')

// Remove code separators (if any)
script.removeCodeseparators()

// Modify specific chunks
script.setChunkOpCode(0, OP.OP_2DUP) // Change first OP_DUP to OP_2DUP

console.log('Modified script:', script.toASM())
```

### Combining Scripts

```typescript
// Combine multiple scripts
const script1 = Script.fromASM('OP_TRUE')
const script2 = Script.fromASM('OP_FALSE')
const script3 = Script.fromASM('OP_ADD')

const combinedScript = new Script()
combinedScript.writeScript(script1)
combinedScript.writeScript(script2)
combinedScript.writeScript(script3)

console.log('Combined script:', combinedScript.toASM())
```

---

## Part 8: Real-World Example

### Creating a Time-Locked Script Template

```typescript
class TimeLock implements ScriptTemplate {
  /**
   * Creates a time-locked script that can only be spent after a certain time
   */
  lock(lockTime: number, pubkeyHash: number[]): LockingScript {
    const script = new LockingScript()
    
    // Push the lock time
    script.writeNumber(lockTime)
    script.writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
    script.writeOpCode(OP.OP_DROP)
    
    // Standard P2PKH after time check
    script.writeOpCode(OP.OP_DUP)
    script.writeOpCode(OP.OP_HASH160)
    script.writeBin(pubkeyHash)
    script.writeOpCode(OP.OP_EQUALVERIFY)
    script.writeOpCode(OP.OP_CHECKSIG)
    
    return script
  }

  /**
   * Creates an unlocking script for time-locked output
   */
  unlock(privateKey: PrivateKey) {
    return {
      sign: async (tx: Transaction, inputIndex: number): Promise<UnlockingScript> => {
        // This would need proper signature creation in a real implementation
        const script = new UnlockingScript()
        
        // Add signature (simplified - would need proper SIGHASH implementation)
        const dummySig = [0x30, 0x44, 0x02, 0x20] // Placeholder signature
        script.writeBin(dummySig)
        
        // Add public key
        const pubkey = privateKey.toPublicKey().encode(true) as number[]
        script.writeBin(pubkey)
        
        return script
      },
      estimateLength: async (): Promise<number> => {
        // Signature (~71 bytes) + public key (33 bytes) + push opcodes
        return 106
      }
    }
  }
}

// Usage
const timeLock = new TimeLock()
const lockTime = Math.floor(Date.now() / 1000) + 3600 // 1 hour from now
const privateKey = PrivateKey.fromRandom()
const pubkeyHash = privateKey.toPublicKey().toHash()

const lockingScript = timeLock.lock(lockTime, pubkeyHash)
console.log('Time-locked script:', lockingScript.toASM())
```

---

## Best Practices

### 1. Script Efficiency

- Keep scripts as small as possible to minimize fees
- Use standard templates when possible
- Avoid unnecessary operations

### 2. Security Considerations

- Validate all inputs in custom templates
- Test scripts thoroughly before mainnet use

### 3. Testing

```typescript
// Always test your custom templates
async function testTemplate(template: ScriptTemplate, ...lockParams: any[]) {
  try {
    const lockingScript = template.lock(...lockParams)
    console.log('✓ Locking script created:', lockingScript.toASM())
    
    // Test unlocking if parameters are available
    // const unlockingTemplate = template.unlock(...unlockParams)
    // const estimate = await unlockingTemplate.estimateLength()
    // console.log('✓ Unlock estimate:', estimate)
    
  } catch (error) {
    console.error('✗ Template test failed:', error.message)
  }
}
```

---

## Common Patterns

### 1. Data Storage Pattern

```typescript
// Store data with OP_RETURN
const dataScript = new Script()
dataScript.writeOpCode(OP.OP_FALSE)
dataScript.writeOpCode(OP.OP_RETURN)
dataScript.writeBin([/* your data */])
```

### 2. Conditional Spending Pattern

```typescript
// IF-ELSE conditional spending
const conditionalScript = new Script()
conditionalScript.writeOpCode(OP.OP_IF)
// ... condition true path
conditionalScript.writeOpCode(OP.OP_ELSE)
// ... condition false path
conditionalScript.writeOpCode(OP.OP_ENDIF)
```

### 3. Hash Verification Pattern

```typescript
// Verify hash preimage
const hashScript = new Script()
hashScript.writeOpCode(OP.OP_SHA256)
hashScript.writeBin([/* expected hash */])
hashScript.writeOpCode(OP.OP_EQUAL)
```

---

## Integration with `WalletClient`

Your custom script templates can be used with the `WalletClient` for production applications:

```typescript
// Create a wallet client instance
const walletClient = new WalletClient({
  // ... wallet client options
})

// Create a custom script template
const customTemplate = new SimplePuzzle()

// Create a locking script
const lockingScript = customTemplate.lock(42)

// Create a transaction with the custom script
const tx = walletClient.createTransaction({
  // ... transaction options
  lockingScript,
})

// Send the transaction
walletClient.sendTransaction(tx)
```

---

## Summary

In this tutorial, you learned:

- ✅ Bitcoin script fundamentals and structure
- ✅ Creating scripts from different formats (ASM, hex, binary)
- ✅ Working with script chunks and opcodes
- ✅ Using standard script templates like P2PKH
- ✅ Building custom script templates
- ✅ Implementing locking and unlocking logic
- ✅ Advanced script operations and combinations
- ✅ Best practices for script development

You now have the knowledge to create sophisticated Bitcoin scripts and custom templates for your applications. Remember to test thoroughly and consider security implications when deploying custom scripts.

## Next Steps

- Explore the [Advanced Transaction Construction](./advanced-transaction.md) tutorial
- Learn about [Transaction Broadcasting](./transaction-broadcasting.md)
- Study the SDK's built-in script templates for more examples
- Practice with testnet before deploying to mainnet

## Troubleshooting

### Common Issues

1. **Script too large**: Minimize operations and data size
2. **Invalid opcodes**: Check opcode values against the OP enum
3. **Serialization errors**: Ensure proper data encoding
4. **Template errors**: Verify unlock function returns correct structure

### Getting Help

- Check the [BSV TypeScript SDK documentation](https://docs.bsvblockchain.org/)
- Review existing script templates in the SDK source
- Test scripts on testnet before mainnet deployment
