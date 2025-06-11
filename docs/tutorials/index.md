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

### [Transaction Types and Data](./transaction-types.md)
- **Duration**: 30 minutes
- **Prerequisites**: Completed "Your First BSV Transaction" tutorial
- **Learning Goals**:
  - Create transactions with multiple outputs
  - Add data to transactions
  - Work with different output types
  - Use advanced WalletClient features

### [Key Management and Cryptography](./key-management.md)
- **Duration**: 45 minutes
- **Prerequisites**: Completed "Your First BSV Transaction" tutorial
- **Learning Goals**:
  - Generate and manage private/public keys
  - Understand ECDSA signatures
  - Create and verify digital signatures

### [Transaction Broadcasting](./transaction-broadcasting.md)
- **Duration**: 25 minutes
- **Prerequisites**: Completed "Your First BSV Transaction" tutorial, Node.js, basic TypeScript knowledge
- **Learning Goals**:
  - Understand WalletClient vs direct broadcasting approaches
  - Configure broadcasting for testnet vs mainnet
  - Implement custom broadcasters (ARC, WhatsOnChain)
  - Handle broadcasting errors and implement retry logic
  - Monitor and verify transaction acceptance

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

### [Error Handling and Edge Cases](./error-handling.md)
- **Duration**: 60 minutes
- **Learning Goals**:
  - Robust error handling patterns
  - Network failure recovery
  - Transaction validation edge cases

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
  - Understand AES-GCM symmetric encryption principles
  - Use the `SymmetricKey` class for encryption and decryption
  - Implement secure key generation and management
  - Apply AES encryption in practical Bitcoin applications
  - Combine AES with ECDH for secure communication
  - Handle different data formats and encoding

### [Hashes and HMACs](./hashes-and-hmacs.md)
- **Duration**: 75 minutes
- **Learning Goals**:
  - Understand cryptographic hash functions and their properties
  - Master the Hash module classes and helper functions in the BSV TypeScript SDK
  - Implement various hash algorithms (SHA-256, SHA-512, SHA-1, RIPEMD-160)
  - Create and verify HMACs for message authentication
  - Apply Bitcoin-specific hashing patterns (hash256, hash160)
  - Build practical applications using hashing for data integrity and authentication
  - Understand performance considerations and security best practices

### [Type-42 Key Derivation](./type-42.md)
- **Duration**: 75 minutes
- **Prerequisites**: Basic TypeScript knowledge, Elliptic Curve Fundamentals tutorial completed, ECDH Key Exchange tutorial completed
- **Learning Goals**:
  - Understand Type-42 key derivation protocol and its use cases
  - Implement Type-42 operations with the BSV TypeScript SDK
  - Create shared key universes between two parties
  - Apply Type-42 in practical Bitcoin applications like message signing and encryption
  - Understand the "anyone key" concept and its applications


## Alternative Low-Level Transaction API Track

These tutorials demonstrate how to use the lower-level APIs of the BSV TypeScript SDK for more direct control over transaction creation and management.

### [Your First BSV Transaction (Low-level API)](./first-transaction-low-level.md)
- **Duration**: 45 minutes
- **Prerequisites**: Node.js, basic TypeScript knowledge
- **Learning Goals**:
  - Work with low-level transaction APIs
  - Create transactions without WalletClient abstraction
  - Understand transaction construction internals
  - Manually manage inputs, outputs, and signing

### [Working with Testnet Transactions (Low-level API)](./testnet-transactions-low-level.md)
- **Duration**: 60 minutes
- **Prerequisites**: Completed "Your First BSV Transaction (Low-level API)" tutorial
- **Learning Goals**:
  - Set up a BSV testnet environment with low-level APIs
  - Manually handle testnet transactions
  - Understand UTXO management without WalletClient
  - Implement custom transaction workflows
