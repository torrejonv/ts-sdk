# How-To Guides

Practical, problem-oriented guides to help you accomplish specific tasks with the BSV TypeScript SDK.

## Transaction Management

### [Transaction Signing Methods](./transaction-signing-methods.md)
- WalletClient approach for secure key management
- Low-level API approach for direct control
- Comparison of different signing methods
- Best practices for transaction signing

### [Advanced Transaction Signing](./advanced-transaction-signing.md)
- Different signature hash types (SIGHASH flags)
- Manual signature creation
- Advanced verification techniques
- Multi-signature implementation

### [Creating Multi-signature Transactions](./multisig-transactions.md)
- Step-by-step multisig implementation
- Threshold signature schemes
- Key ceremony management

### [Implementing Transaction Batching](./transaction-batching.md)
- Batch multiple payments efficiently
- Fee optimization strategies
- Error handling for batch failures

### [Handling Large Transactions](./large-transactions.md)
- Memory management techniques
- Streaming transaction construction
- UTXO selection algorithms

## Cryptographic Operations

### [Security Best Practices](./security-best-practices.md)
- Private key management and protection
- Secure transaction construction
- Cryptographic operation security
- Wallet integration security patterns
- Production security checklist

### [Setting up Development Wallets](./development-wallet-setup.md)
- ProtoWallet configuration for development and testing
- Mock transaction creation and testing workflows
- Multi-wallet development environments
- Key management for development scenarios

### [Implementing Custom Key Derivation](./custom-key-derivation.md)
- BIP32-style hierarchical keys
- Custom derivation paths
- Key backup and recovery

### [Creating Encrypted Messages](./encrypted-messages.md)
- ECIES implementation
- Message encryption/decryption
- Key exchange protocols

### [Verifying Complex Signatures](./complex-signatures.md)
- Batch signature verification with performance optimization
- Threshold signature validation using polynomial interpolation
- Multi-context signature validation workflows
- Time-locked and conditional signature scenarios
- Comprehensive error handling and recovery strategies
- Security considerations for complex verification patterns

## Network Integration

### [Setting up Authenticated API Communication](./authenticated-api-communication.md)
- BRC-103/104 authentication implementation
- Certificate-based API security
- Session management and retry logic
- Secure peer-to-peer communication

### [Setting Up Chain Tracking](./chain-tracking.md)
- Configuring chain trackers for blockchain data access
- Using WhatsOnChain and other providers
- SPV verification with chain trackers
- Error handling and fallback strategies

### [Configuring HTTP Clients](./http-client-configuration.md)
- Axios integration and setup
- Custom request timeout configuration
- Error handling and retries
- Alternative HTTP client options

### [Creating Custom Broadcasters](./custom-broadcasters.md)
- Implementing custom broadcaster interfaces
- HTTP-based broadcaster patterns
- Retry logic and error handling
- Multi-service failover strategies

### [Implementing Transaction Monitoring](./transaction-monitoring.md)
- Real-time transaction tracking
- Confirmation monitoring
- Double-spend detection

## File and Data Management

### [Implementing File Upload/Download Features](./file-upload-download.md)
- UHRP-based decentralized file storage
- File integrity verification and validation
- Batch file operations and management
- File retention and renewal strategies

## Identity and Access Management

### [Building Identity Verification Systems](./identity-verification-systems.md)
- Decentralized identity verification workflows
- Trust scoring and certificate validation
- Identity-based access control systems
- Verification history and audit trails

## Cross-Platform Integration

### [Working with React](./react-integration.md)
- Setting up the SDK in React projects
- State management for keys and transactions
- React component patterns for BSV applications
- React Native considerations
