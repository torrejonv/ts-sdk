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

### [Implementing Custom Key Derivation](./custom-key-derivation.md)
- BIP32-style hierarchical keys
- Custom derivation paths
- Key backup and recovery

### [Creating Encrypted Messages](./encrypted-messages.md)
- ECIES implementation
- Message encryption/decryption
- Key exchange protocols

### [Verifying Complex Signatures](./complex-signatures.md)
- Multi-party signatures
- Threshold signatures
- Time-based signatures

## Network Integration

### [Configuring Custom ARC Endpoints](./custom-arc-endpoints.md)
- ARC service configuration
- Load balancing across miners
- Failover and retry logic

### [Configuring HTTP Clients](./http-client-configuration.md)
- Axios integration and setup
- Custom request timeout configuration
- Error handling and retries
- Alternative HTTP client options

### [Implementing Transaction Monitoring](./transaction-monitoring.md)
- Real-time transaction tracking
- Confirmation monitoring
- Double-spend detection

### [Optimizing Network Performance](./network-performance.md)
- Connection pooling
- Request batching
- Caching strategies

## Cross-Platform Integration

### [Working with React](./react-integration.md)
- Setting up the SDK in React projects
- State management for keys and transactions
- React component patterns for BSV applications
- React Native considerations

## Alternative Programming Languages 

### [Working with Go SDK](./go-sdk-integration.md)
- TypeScript to Go SDK interoperability
- Cross-language data format handling
- Building hybrid TypeScript/Go applications
- Migration strategies between SDKs

### [Working with Python SDK](./python-sdk-integration.md)
- TypeScript to Python SDK interoperability
- Data science and analytics integration
- Cross-language API design patterns
- Python-specific use cases and optimizations

## Development Workflows

### [Debugging Transaction Construction](./debugging-transactions.md)
- Transaction visualization tools
- Script debugging techniques
- Common error patterns

### [Testing SDK Integrations](./testing-integrations.md)
- Unit testing patterns
- Integration test setup
- Mock ARC services

### [Migrating from Legacy BSV Libraries](./migration-guide.md)
- Migration from bsv-js
- API mapping guide
- Breaking changes handling
