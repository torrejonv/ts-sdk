# Tutorials

Welcome to the BSV TypeScript SDK tutorials. These step-by-step lessons will teach you how to use the SDK through practical examples.

## Getting Started Track

### [Your First BSV Transaction](./first-transaction.md)
- **Duration**: 30 minutes
- **Prerequisites**: Node.js, basic TypeScript knowledge
- **Learning Goals**:
  - Install and configure the SDK
  - Create a simple P2PKH transaction
  - Understand BSV transaction anatomy

### [Working with Testnet Transactions](./testnet-transactions.md)
- **Duration**: 45 minutes
- **Prerequisites**: Completed "Your First BSV Transaction" tutorial
- **Learning Goals**:
  - Set up a BSV testnet environment
  - Obtain and manage testnet coins
  - Create and broadcast real transactions
  - Track and verify transaction confirmations
  - Understand transaction fees in practice

### [Key Management and Cryptography](./key-management.md)
- **Duration**: 45 minutes
- **Prerequisites**: Completed "Your First BSV Transaction" tutorial
- **Learning Goals**:
  - Generate and manage private/public keys
  - Understand ECDSA signatures
  - Create and verify digital signatures

### [Transaction Broadcasting and ARC](./transaction-broadcasting.md)
- **Duration**: 60 minutes
- **Prerequisites**: Completed "Working with Testnet Transactions" tutorial
- **Learning Goals**:
  - Configure ARC broadcaster
  - Handle transaction broadcasting
  - Monitor transaction status

## Intermediate Development Track

### [Advanced Transaction Construction](./advanced-transaction.md)
- **Duration**: 75 minutes
- **Learning Goals**:
  - Multi-input/multi-output transactions
  - Fee calculation and optimization
  - Change output handling

### [Script Construction and Custom Logic](./script-construction.md)
- **Duration**: 90 minutes
- **Learning Goals**:
  - Understand Bitcoin Script basics
  - Create custom locking scripts
  - Implement unlocking logic

### [SPV and Merkle Proof Verification](./spv-merkle-proofs.md)
- **Duration**: 90 minutes
- **Learning Goals**:
  - Understand SPV principles
  - Verify Merkle proofs
  - Implement lightweight verification

## Low-Level Cryptography Track

### [Elliptic Curve Fundamentals: Numbers & Points](./elliptic-curve-fundamentals.md)
- **Duration**: 90 minutes
- **Learning Goals**:
  - Understand the mathematical foundations of elliptic curves
  - Work with BigInteger numbers in the SDK
  - Manipulate elliptic curve points
  - Implement point addition and scalar multiplication

### [ECDH Key Exchange](./ecdh-key-exchange.md)
- **Duration**: 75 minutes
- **Learning Goals**:
  - Understand Elliptic Curve Diffie-Hellman (ECDH) key exchange principles
  - Implement secure key exchange using the SDK
  - Create shared secrets for encrypted communication
  - Apply ECDH in practical Bitcoin applications

### [AES Symmetric Encryption](./aes-encryption.md)
- **Duration**: 60 minutes
- **Learning Goals**:
  - Understand symmetric encryption principles
  - Implement AES encryption/decryption with the SDK
  - Manage encryption keys securely
  - Apply encryption in practical Bitcoin scenarios

### [Hashes and HMACs](./hashes-and-hmacs.md)
- **Duration**: 60 minutes
- **Learning Goals**:
  - Understand cryptographic hashing functions and their properties
  - Implement various hash algorithms using the SDK
  - Create and verify HMACs (Hash-based Message Authentication Codes)
  - Apply hashing techniques in practical Bitcoin scenarios

### [Type 42 Implementation](./type-42.md)
- **Duration**: 60 minutes
- **Learning Goals**:
  - Understand Type 42 protocol and its use cases
  - Implement Type 42 operations with the SDK
  - Create and parse Type 42 data
  - Apply Type 42 in practical Bitcoin applications

## Production-Ready Track

### [Error Handling and Edge Cases](./error-handling.md)
- **Duration**: 60 minutes
- **Learning Goals**:
  - Robust error handling patterns
  - Network failure recovery
  - Transaction validation edge cases

### [Performance Optimization](./performance-optimization.md)
- **Duration**: 75 minutes
- **Learning Goals**:
  - Memory management for large transactions
  - Batch processing techniques
  - Caching strategies

### [Security Best Practices](./security-best-practices.md)
- **Duration**: 90 minutes
- **Learning Goals**:
  - Secure key storage patterns
  - Input validation and sanitization
  - Side-channel attack prevention
