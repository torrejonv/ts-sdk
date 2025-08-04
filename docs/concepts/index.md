# Concepts

Essential concepts for understanding and using the BSV TypeScript SDK effectively.

## Core Bitcoin Concepts

### [Transaction Structure](./transaction-structure.md)

Understanding Bitcoin transactions, inputs, outputs, and how they work in the SDK.

### [Script Templates](./script-templates.md)

Standard and custom Bitcoin script patterns available in the SDK.

### [Digital Signatures](./signatures.md)

How digital signatures work in Bitcoin and their implementation in the SDK.

### [Transaction Verification](./verification.md)

Understanding how to verify Bitcoin transactions using the SDK.

### [SPV Verification](./spv-verification.md)

Simplified Payment Verification and merkle path concepts for lightweight clients.

### [Transaction Fees](./fees.md)

How Bitcoin transaction fees work and fee optimization strategies.

## SDK Architecture

### [SDK Design Philosophy](./sdk-philosophy.md)

Core principles: zero dependencies, SPV-first approach, and vendor neutrality.

### [Wallet Integration](./wallet-integration.md)

How the SDK connects with Bitcoin wallets and manages authentication.

### [Actions](./actions.md)

The primary abstraction for creating and managing Bitcoin transactions in the SDK.

### [Output Baskets](./output-baskets.md)

Organizational system for managing and categorizing UTXOs in your wallet.

### [Chain Tracking](./chain-tracking.md)

Understanding how the SDK interacts with the Bitcoin network for transaction data.

## Data Formats

### [BEEF Format](./beef.md)

Bitcoin Extras Extension Format for efficient transaction data exchange.

### [Transaction Encoding](./transaction-encoding.md)

How transactions are serialized and deserialized in the SDK.

## Identity and Certificates

### [Decentralized Identity](./decentralized-identity.md)

Understanding BSV's decentralized identity system and certificate-based verification.

### [Identity Certificates](./identity-certificates.md)

How cryptographic certificates work for identity claims and verification.

## Security Model

### [Key Management](./key-management.md)

How private keys, public keys, and cryptographic operations work in the SDK.

---

These concepts provide the foundational knowledge needed to build Bitcoin applications with the BSV TypeScript SDK. For deeper protocol details, refer to the [BSV Skills Center](https://docs.bsvblockchain.org/).
