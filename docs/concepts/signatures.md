# Digital Signatures

How digital signatures work in Bitcoin and their implementation in the BSV TypeScript SDK.

## What are Digital Signatures?

Digital signatures prove ownership and authorize Bitcoin transactions:

```typescript
import { PrivateKey, Transaction } from '@bsv/sdk'

// Create a signature
const privateKey = PrivateKey.fromRandom()
const message = 'transaction data'
const signature = privateKey.sign(message)

// Verify the signature
const publicKey = privateKey.toPublicKey()
const isValid = publicKey.verify(message, signature)
```

## Bitcoin Signatures

Bitcoin uses ECDSA (Elliptic Curve Digital Signature Algorithm):
- **secp256k1**: The elliptic curve used by Bitcoin
- **SHA-256**: Hash function for message digests
- **DER Encoding**: Standard format for signature serialization

## SIGHASH Types

SIGHASH flags determine what parts of a transaction are signed:

### SIGHASH_ALL (Default)
Signs all inputs and outputs:
```typescript
const signature = privateKey.sign(txHash, 'all')
```

### SIGHASH_NONE
Signs all inputs but no outputs:
```typescript
const signature = privateKey.sign(txHash, 'none')
```

### SIGHASH_SINGLE
Signs all inputs and one corresponding output:
```typescript
const signature = privateKey.sign(txHash, 'single')
```

### SIGHASH_ANYONECANPAY
Can be combined with other flags to sign only one input:
```typescript
const signature = privateKey.sign(txHash, 'all|anyonecanpay')
```

## Transaction Signing

The SDK handles transaction signing automatically:

```typescript
// Manual signing (low-level)
const tx = new Transaction()
const signature = tx.sign(privateKey, inputIndex, sighashType)

// Wallet signing (recommended)
const wallet = new WalletClient()
const action = await wallet.createAction({
  outputs: [/* outputs */]
})
// Wallet handles signing internally
```

## Signature Verification

Verify signatures to ensure transaction validity:

```typescript
// Verify a specific signature
const isValid = publicKey.verify(messageHash, signature)

// Verify entire transaction
const txValid = await transaction.verify(chainTracker)
```

## DER Encoding

Signatures are encoded in DER format:

```typescript
// Get DER-encoded signature
const derSignature = signature.toDER()

// Parse DER signature
const sig = Signature.fromDER(derBytes)

// Get r and s components
const r = signature.r
const s = signature.s
```

## Security Considerations

### Nonce Security
- Each signature must use a unique, random nonce
- Reusing nonces can leak private keys
- The SDK handles nonce generation securely

### Signature Malleability
- Bitcoin signatures can be modified without invalidating them
- Use canonical signatures to prevent malleability
- The SDK produces canonical signatures by default

### Hash Types
- Choose appropriate SIGHASH types for your use case
- SIGHASH_ALL is safest for most applications
- Other types enable advanced transaction patterns

## Common Patterns

### Multi-Input Signing
```typescript
// Sign multiple inputs in a transaction
for (let i = 0; i < transaction.inputs.length; i++) {
  const signature = privateKey.sign(transaction.getSignatureHash(i))
  transaction.inputs[i].unlockingScript = createUnlockingScript(signature)
}
```

### Conditional Signatures
```typescript
// Different signatures for different conditions
const signature1 = privateKey1.sign(txHash, 'all')
const signature2 = privateKey2.sign(txHash, 'single')
```

## Error Handling

Common signature issues:
- Invalid private key format
- Incorrect message hash
- Malformed signature data
- Verification failures

```typescript
try {
  const signature = privateKey.sign(message)
} catch (error) {
  console.error('Signing failed:', error.message)
}
```

## Best Practices

- Always use secure random number generation
- Verify signatures before trusting them
- Use appropriate SIGHASH types for your use case
- Store signatures in DER format for interoperability
- Never reuse nonces across signatures

## Wallet Integration

Most applications use wallets for signing:

```typescript
// Wallet handles signature creation
const wallet = new WalletClient()
const result = await wallet.createAction({
  description: 'Payment transaction',
  outputs: [/* outputs */]
})
// Signatures created automatically
```

## Next Steps

- Understand [Key Management](./key-management.md) for signature security
- Learn about [Script Templates](./script-templates.md) for signature usage
- Explore [Transaction Structure](./transaction-structure.md) for signature placement
