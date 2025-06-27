# Reference Documentation

Complete technical specifications and API documentation for the BSV TypeScript SDK.

## Standards and Interfaces

### [BRC-100 Wallet Interface](./brc-100.md)
- Unified wallet-to-application interface standard
- WalletClient implementation details
- JSON API specifications

## Core Classes

### [Transaction Class](./transaction.md)
- Constructor options and methods
- Input/output management
- Serialization formats

### [PrivateKey/PublicKey Classes](./primitives.md)
- Key generation methods
- Import/export formats
- Cryptographic operations

### [Transaction Signatures Reference](./transaction-signatures.md)
- ECDSA signature components
- DER encoding format
- Signature hash types (SIGHASH flags)

### [Script Classes](./script.md)
- Script construction utilities
- Standard script templates
- Custom script patterns

### [OP Codes Reference](./op-codes.md)
- Complete opcode listing and descriptions
- Opcode categories and usage patterns
- Script execution examples

## Module Reference

### [Primitives Module](./primitives.md)
- Cryptographic primitives
- Hash functions
- Security implementation notes

### [Transaction Module](./transaction.md)
- Transaction lifecycle
- Fee calculation details
- Broadcasting options
- SPV verification

### [Wallet Module](./wallet.md)
- Wallet architecture patterns
- Integration guidelines
- BRC-100 compliance notes

## Configuration Reference

### [SDK Configuration Options](./configuration.md)
```typescript
interface SDKConfig {
  network: 'mainnet' | 'testnet' | 'regtest'
  arc: ARCConfig
  fees: FeeConfig
  security: SecurityConfig
}
```

### [ARC Configuration](./arc-config.md)
- Endpoint configuration
- Authentication methods
- Rate limiting options
- Failover settings

### [Network Configuration](./network-config.md)
- Mainnet vs testnet settings
- Custom network parameters
- Node endpoint configurations

## Error Reference

### [Error Codes and Messages](./errors.md)
- Transaction validation errors
- Network connectivity errors
- Cryptographic operation errors
- Troubleshooting steps

### [Debugging Guide](./debugging.md)
- SDK logging configuration
- Debug mode activation
- Transaction inspection tools


## Swagger

[BRC-100](https://brc.dev/100) defines a Unified, Vendor-Neutral, Unchanging, and Open BSV Blockchain Standard Wallet-to-Application Interface which is implemented in this library within the WalletClient class. The API is laid out here as a swagger openapi document to offer a fast-track to understanding the interface which is implemented across multiple substrates. The JSON api is generally considered a developer friendly introduction to the WalletClient, where an binary equivalent ABI may be preferred for production use cases.

- [Wallet JSON API Swagger](../swagger)
