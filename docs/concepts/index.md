# Concepts & Explanations

Deep dives into the architecture, design principles, and concepts behind the BSV TypeScript SDK.

## Bitcoin Protocol Fundamentals

### [BEEF (Bitcoin Extras Extension Format)](./beef.md)
- Standardized format for including additional data with Bitcoin transactions
- Integration with transaction processing in the SDK
- Use cases and implementation patterns

### [Transaction Fees](./fees.md)
- Fee calculation and management in Bitcoin
- Fee estimation strategies in the SDK
- Best practices for fee optimization

### [Script Templates](./script-templates.md)
- Standard and custom Bitcoin script templates
- Implementation of templates in the SDK
- Template selection and composition patterns

### [Transaction Signatures](./signatures.md)
- Digital signature fundamentals in Bitcoin
- SIGHASH types and their applications
- Implementation best practices in the SDK

### [Transaction Verification](./verification.md)
- Transaction validity criteria and verification levels
- SPV verification implementation
- Best practices for secure verification

## Architecture and Design Principles

### [BSV SDK Philosophy](./sdk-philosophy.md)
- Why TypeScript SDK was created
- Design principles: Zero dependencies, SPV-first, vendor neutrality
- Architecture patterns: Modular design, composable components

### [Understanding Bitcoin Script](./bitcoin-script.md)
- Script execution model
- Standard script patterns (P2PKH, P2SH, etc.)
- Custom script development philosophy
- Security considerations in script design

### [SPV Architecture Deep Dive](./spv-architecture.md)
- Why SPV matters for scaling
- Merkle proof verification tools and serialization standards
- Trade-offs: security vs efficiency
- Integration with overlay services

## Security Model and Best Practices

### [Cryptographic Foundations](./cryptographic-foundations.md)
- Sound cryptographic primitives
- Why secp256k1 for Bitcoin
- Random number generation security
- Side-channel attack prevention

### [Key Management Philosophy](./key-management.md)
- Hot vs cold storage considerations
- Key derivation strategies
- Backup and recovery approaches
- Multi-signature security models

### [Transaction Security](./transaction-security.md)
- Input validation principles
- Double-spend prevention
- Replay attack protection
- Fee manipulation defenses

## Performance and Scalability

### [Transaction Construction Efficiency](./transaction-efficiency.md)
- Memory usage optimization
- Batch processing strategies
- Large transaction handling
- UTXO selection algorithms

### [Network Performance](./network-performance.md)
- ARC integration patterns
- Connection management
- Retry and failover strategies
- Monitoring and alerting

## Integration Patterns

### [Wallet Integration](./wallet-integration.md)
- BRC-100 compliance considerations
- Vendor-neutral design principles
- Migration strategies from legacy systems

### [Application Architecture](./application-architecture.md)
- Event-driven transaction processing
- State management patterns
- Error boundary implementation
- Testing strategies

### [Ecosystem Integration](./ecosystem-integration.md)
- Template system integration
- Wallet-toolbox integration
- Overlay services integration
- Identity and authentication systems
