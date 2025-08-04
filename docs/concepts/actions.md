# Actions

## Overview

Actions are the primary abstraction layer for creating and managing Bitcoin transactions in the BSV SDK. Rather than dealing directly with the complexities of transaction construction, Actions provide a simplified, wallet-integrated approach to building, signing, and broadcasting transactions.

## Why Actions?

Traditional Bitcoin transaction construction requires developers to:

- Manually select and manage UTXOs
- Calculate appropriate transaction fees
- Handle change outputs
- Manage signature generation and placement
- Coordinate transaction broadcasting
- Track transaction status through confirmation

Actions abstract away these complexities while maintaining full control over transaction specifics when needed.

## The Action Lifecycle

An Action progresses through several distinct stages from creation to confirmation:

### 1. Creation Phase

When you call `createAction()`, the wallet:

- Evaluates your output requirements
- Automatically selects appropriate UTXOs as inputs
- Calculates optimal transaction fees
- Creates change outputs if necessary
- Returns an unsigned transaction with metadata

```typescript
const action = await wallet.createAction({
  description: 'Payment for services',
  outputs: [{
    to: recipientAddress,
    satoshis: 50000,
    description: 'Service payment'
  }]
})
```

### 2. Signing Phase

The wallet coordinates signature generation:

- Identifies which inputs require signatures
- Requests signatures from the appropriate key sources
- Places signatures in the correct script locations
- Validates the transaction is properly signed

### 3. Processing Phase

Once signed, the Action enters processing:

- Transaction is broadcast to the Bitcoin network
- Status updates are tracked automatically
- SPV proof data is collected when available
- Transaction is stored with organizational metadata

### 4. Completion Phase

An Action is complete when:

- The transaction is accepted by the network
- Merkle proof data is available (for SPV)
- All outputs are spendable by their recipients
- Transaction data is indexed for future reference

## Action Status States

Actions progress through specific status states:

- **`unsigned`**: Created but awaiting signatures
- **`unprocessed`**: Signed but not yet broadcast
- **`sending`**: Currently being broadcast to the network
- **`unproven`**: Broadcast but awaiting merkle proof
- **`completed`**: Fully processed with proof data
- **`failed`**: Processing failed at some stage
- **`nosend`**: Created with `noSend` option, awaiting manual broadcast
- **`nonfinal`**: Transaction uses nLockTime or nSequence for delayed finality

## Action Options

Actions support various processing options:

### Immediate Processing

```typescript
const action = await wallet.createAction({
  description: 'Immediate payment',
  outputs: [...],
  signAndProcess: true  // Sign and broadcast immediately
})
```

### Delayed Broadcasting

```typescript
const action = await wallet.createAction({
  description: 'Delayed broadcast',
  outputs: [...],
  noSend: true  // Create and sign, but don't broadcast
})

// Broadcast later
await wallet.processTransaction({
  txid: action.txid,
  rawTx: action.rawTx
})
```

### Background Processing

```typescript
const action = await wallet.createAction({
  description: 'Background processing',
  outputs: [...],
  acceptDelayedBroadcast: true  // Return immediately, process in background
})
```

## Actions vs Direct Transactions

### When to Use Actions

- **Most application development**: Actions handle the complexity
- **Wallet-integrated applications**: Automatic UTXO and key management
- **Multi-party transactions**: Coordination through wallet infrastructure
- **Output organization**: Automatic basket assignment and tagging

### When to Use Direct Transactions

- **Custom UTXO selection**: Specific input requirements
- **Non-standard scripts**: Complex smart contract interactions
- **External key management**: Keys not managed by wallet
- **Transaction templates**: Pre-built transaction structures

## Best Practices

### 1. Descriptive Metadata

Always provide clear descriptions for actions and outputs:

```typescript
await wallet.createAction({
  description: 'Monthly subscription payment - January 2024',
  outputs: [{
    satoshis: 10000,
    to: merchantAddress,
    description: 'Subscription #12345'
  }]
})
```

### 2. Error Handling

Actions can fail at various stages:

```typescript
try {
  const action = await wallet.createAction({...})
} catch (error) {
  if (error.code === 'insufficient-funds') {
    // Handle insufficient balance
  } else if (error.code === 'signing-failed') {
    // Handle signing errors
  }
}
```

### 3. Status Monitoring

For long-running operations:

```typescript
const action = await wallet.createAction({
  outputs: [...],
  acceptDelayedBroadcast: true
})

// Check status later
const status = await wallet.listActions({
  txids: [action.txid]
})
```

### 4. Batch Operations

Create multiple related transactions efficiently:

```typescript
const actions = await Promise.all(
  recipients.map(recipient => 
    wallet.createAction({
      description: `Batch payment to ${recipient.name}`,
      outputs: [{
        to: recipient.address,
        satoshis: recipient.amount
      }]
    })
  )
)
```

## Advanced Patterns

### Custom Input Selection

While Actions handle UTXO selection automatically, you can provide specific inputs:

```typescript
await wallet.createAction({
  description: 'Spend specific UTXOs',
  inputs: [{
    txid: 'abc123...',
    vout: 0
  }],
  outputs: [...]
})
```

### Tag-Based Organization

Use tags for transaction categorization:

```typescript
await wallet.createAction({
  description: 'Customer payment',
  outputs: [{
    satoshis: 25000,
    to: customerAddress,
    tags: ['customer-payment', 'invoice-789']
  }],
  labels: ['revenue', 'Q1-2024']
})
```

### Multi-Signature Coordination

Actions simplify multi-party transactions:

```typescript
// Create unsigned action
const action = await wallet.createAction({
  description: 'Multi-sig transaction',
  outputs: [...],
  signAndProcess: false
})

// Coordinate signatures from multiple parties
// ... signature collection process ...

// Process once all signatures are collected
await wallet.processTransaction({
  txid: action.txid,
  rawTx: signedTransaction
})
```

## Related Concepts

- [Output Baskets](./output-baskets.md) - Organizing outputs created by actions
- [Transaction Structure](./transaction-structure.md) - Understanding the underlying transaction format
- [Wallet Integration](./wallet-integration.md) - How actions interact with the wallet
- [SPV Verification](./spv-verification.md) - Proof data collected during action processing
