# Key Management

Understanding how private keys, public keys, and cryptographic operations work in the BSV TypeScript SDK.

## Cryptographic Keys

Bitcoin uses elliptic curve cryptography (secp256k1) for key management:

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

// Generate a new private key
const privateKey = PrivateKey.fromRandom()

// Derive the corresponding public key
const publicKey = privateKey.toPublicKey()
```

## Private Keys

Private keys are 256-bit numbers that control Bitcoin funds:

### Generation

```typescript
// Secure random generation
const privKey = PrivateKey.fromRandom()

// From existing data (use carefully)
const privKey2 = PrivateKey.fromString('hex_string')
```

### Formats

```typescript
// WIF (Wallet Import Format)
const wif = privateKey.toWif()

// Hex string
const hex = privateKey.toString()

// DER encoding
const der = privateKey.toDER()
```

## Public Keys

Public keys are derived from private keys and can be shared safely:

### Derivation

```typescript
// Always derive from private key
const publicKey = privateKey.toPublicKey()

// Cannot go backwards (public -> private)
```

### Formats

```typescript
// Compressed (33 bytes) - preferred
const compressed = publicKey.toString()

// Uncompressed (65 bytes) - legacy
const uncompressed = publicKey.toString(false)

// DER encoding
const der = publicKey.toDER()
```

## Digital Signatures

Private keys create signatures that prove ownership:

```typescript
// Sign a message
const message = 'Hello Bitcoin'
const signature = privateKey.sign(message)

// Verify with public key
const isValid = publicKey.verify(message, signature)
```

## Key Derivation

The SDK supports hierarchical key derivation:

```typescript
// Derive child keys (simplified example)
const childKey = privateKey.deriveChild(0)
const childPubKey = childKey.toPublicKey()
```

## Security Considerations

### Private Key Security

- **Never expose**: Private keys should never be logged or transmitted
- **Secure storage**: Use encrypted storage for private keys
- **Random generation**: Always use cryptographically secure randomness
- **Access control**: Limit who can access private key operations

### Best Practices

```typescript
// Good: Generate securely
const key = PrivateKey.fromRandom()

// Bad: Predictable generation
const badKey = PrivateKey.fromString('1234567890abcdef...')

// Good: Derive public key when needed
const pubKey = key.toPublicKey()

// Bad: Store private key unnecessarily
localStorage.setItem('privateKey', key.toString())
```

## Wallet Integration

In most applications, wallets handle key management:

The `WalletClient` provides high-level key management through wallet integration:

```typescript
// Wallet manages keys securely
const wallet = new WalletClient()

// Application doesn't see private keys
const action = await wallet.createAction({
  outputs: [/* transaction outputs */]
})
```

When using the `WalletClient`, keys are managed by the connected wallet service:

The `WalletClient` approach is recommended for production applications as it provides:

## Key Recovery

Keys can be recovered from various formats:

```typescript
// From WIF format
const key1 = PrivateKey.fromWif(wifString)

// From hex string
const key2 = PrivateKey.fromString(hexString)

// From DER encoding
const key3 = PrivateKey.fromDER(derBytes)
```

## Cryptographic Operations

The SDK provides secure implementations of:

- **ECDSA**: Digital signature algorithm
- **ECDH**: Key exchange protocol
- **Hash Functions**: SHA-256, RIPEMD-160
- **AES**: Symmetric encryption

## Memory Management

Sensitive key data should be cleared when no longer needed:

- Use secure memory practices
- Clear variables containing key data
- Avoid keeping keys in memory longer than necessary

## Testing and Development

For development and testing:

- Use testnet for experiments
- Generate new keys for each test
- Never use mainnet keys in test code
- Implement proper key rotation

## Next Steps

- Understand [Digital Signatures](./signatures.md) in detail
- Explore [Wallet Integration](./wallet-integration.md) patterns
