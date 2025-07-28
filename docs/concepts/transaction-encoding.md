# Transaction Encoding

How Bitcoin transactions are serialized and deserialized in different formats within the BSV TypeScript SDK.

## Encoding Formats

The SDK supports multiple transaction encoding formats:

### Raw Transaction Format

Standard Bitcoin transaction serialization:

```typescript
import { Transaction } from '@bsv/sdk'

// Serialize to hex
const txHex = transaction.toHex()

// Deserialize from hex
const tx = Transaction.fromHex(txHex)
```

### BEEF Format

Bitcoin Extras Extension Format for efficient data exchange:

```typescript
// Serialize to BEEF
const beefHex = transaction.toBEEF()

// Deserialize from BEEF
const tx = Transaction.fromHexBEEF(beefHex)
```

### Binary Format

Raw binary data for maximum efficiency:

```typescript
// Serialize to binary
const txBinary = transaction.toBinary()

// Deserialize from binary
const tx = Transaction.fromBinary(txBinary)
```

## Serialization Structure

### Standard Transaction

```
Version (4 bytes)
Input Count (varint)
Inputs (variable)
Output Count (varint)  
Outputs (variable)
Lock Time (4 bytes)
```

### Input Structure

```
Previous TX Hash (32 bytes)
Output Index (4 bytes)
Script Length (varint)
Unlocking Script (variable)
Sequence (4 bytes)
```

### Output Structure

```
Value (8 bytes)
Script Length (varint)
Locking Script (variable)
```

## BEEF Enhancements

BEEF format adds:

- **Merkle Paths**: Data structure for SPV verification
- **Block Headers**: Chain validation information
- **Metadata**: Additional transaction context
- **Compression**: Efficient encoding for large datasets

## Encoding Considerations

### Size Optimization

- Use BEEF for transactions with proofs
- Use raw format for minimal overhead
- Consider compression for large batches

### Compatibility

- Raw format works with all Bitcoin software
- BEEF requires compatible parsers
- Binary format is most efficient but less portable

### Performance

- Binary operations are fastest
- Hex encoding is human-readable
- BEEF provides best feature/size ratio

## Working with Encodings

### Conversion Between Formats

```typescript
// Start with a transaction
const tx = new Transaction()

// Convert between formats
const hex = tx.toHex()
const binary = tx.toBinary()
const beef = tx.toBEEF()

// All represent the same transaction
```

### Validation

```typescript
// Verify encoding integrity
const originalTx = Transaction.fromHex(hex)
const roundTripHex = originalTx.toHex()
console.log(hex === roundTripHex) // true
```

## Use Cases

### Network Transmission

- Use hex for JSON APIs
- Use binary for efficient protocols
- Use BEEF for SPV applications

### Storage

- Raw format for blockchain storage
- BEEF for application databases
- Binary for memory-constrained environments

### Interoperability

- Hex for debugging and logging
- Raw format for wallet compatibility
- BEEF for modern Bitcoin applications

## Error Handling

Common encoding issues:

- Invalid hex characters
- Truncated binary data
- Malformed BEEF structures
- Version compatibility problems

The SDK provides comprehensive validation and error reporting.

## Next Steps

- Understand [BEEF Format](./beef.md) in detail
- Learn about [SPV Verification](./spv-verification.md) with BEEF
- Explore [Transaction Structure](./transaction-structure.md) fundamentals
