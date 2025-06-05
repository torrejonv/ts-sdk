# BSV TypeScript SDK Documentation

> **Note:** We're migrating to a new documentation structure. Please see our [main documentation page](./index.md) for the new organization.


## API Reference Documentation

The following reference documentation pages cover specific functional areas of the SDK:

- [Primitives](./reference/primitives.md) — Public and private keys, key derivation, digital signatures, symmetric keys, and low-level operations
- [Script](./reference/script.md) — Bitcoin scripts, templating system, serialization, and script interpreter
- [Transaction](./reference/transaction.md) — Transaction construction, signing, broadcasters, fee models, merkle proofs, and SPV structures
- [Messages](./reference/messages.md) — Message signing, verification, encryption and decryption
- [TOTP](./reference/totp.md) - Time-based One Time Password for secure counterparty validation
- [Wallet](./reference/wallet.md) - Wallet interface for standardized communication between applications and wallets
- [Overlay Tools](./reference/overlay-tools.md) - Overlays for transaction broadcasting and token lookup
- [Auth](./reference/auth.md) - Mutual Authentication and Service Monetization Framework
- [Storage](./reference/storage.md) — UHRP client for distributed data storage by hash
- [Registry](./reference/registry.md) — Distributed protocol and certificate registration
- [Compat](./reference/compat.md) — Deprecated functionality for legacy systems like BIP32 and ECIES

## Swagger

[BRC-100](https://brc.dev/100) defines a Unified, Vendor-Neutral, Unchanging, and Open BSV Blockchain Standard Wallet-to-Application Interface which is implemented in this library within the WalletClient class. The API is laid out here as a swagger openapi document to offer a fast-track to understanding the interface which is implemented across multiple substrates. The JSON api is generally considered a developer friendly introduction to the WalletClient, where an binary equivalent ABI may be preferred for production use cases.

- [Wallet JSON API Swagger](./swagger)
