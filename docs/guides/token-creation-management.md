# Token Creation and Management

Learn how to create, manage, and redeem tokens using baskets and the WalletClient interface.

## Overview

Tokens in the BSV ecosystem represent digital assets that can be created, transferred, and redeemed. This guide demonstrates how to use the SDK's token management features with baskets for organized token storage.

## Creating Tokens

### Basic Token Creation

```typescript
import { WalletClient, Script } from '@bsv/sdk'

const wallet = new WalletClient('auto', 'localhost')

// Create a simple token (e.g., event ticket)
const response = await wallet.createAction({
  description: 'create an event ticket',
  outputs: [{
    satoshis: 1,
    lockingScript: Script.fromASM('OP_NOP').toHex(),
    basket: 'event tickets',
    outputDescription: 'event ticket'
  }]
})

console.log('Token created:', response)
```

### Token with Custom Data

```typescript
import { WalletClient, Script, OP, PrivateKey } from '@bsv/sdk'

const wallet = new WalletClient('auto', 'localhost')

// Create a token with embedded custom data
const data = Array.from('Hello World', char => char.charCodeAt(0))

// Generate recipient key (in practice, you'd get this from the actual recipient)
const recipientPrivKey = new PrivateKey(1)
const recipientPubKey = recipientPrivKey.toPublicKey()
const recipientPubKeyHash = recipientPubKey.toHash()

const lockingScript = new Script([
  { op: data.length, data },
  { op: OP.OP_DROP },
  { op: OP.OP_DUP },
  { op: OP.OP_HASH160 },
  { op: recipientPubKeyHash.length, data: recipientPubKeyHash },
  { op: OP.OP_EQUALVERIFY },
  { op: OP.OP_CHECKSIG }
])

const response = await wallet.createAction({
  description: 'create hello world token',
  outputs: [{
    satoshis: 1000,
    lockingScript: lockingScript.toHex(),
    basket: 'hello world tokens',
    outputDescription: 'hello world token'
  }]
})
```

## Managing Tokens

### Listing Tokens in a Basket

```typescript
const wallet = new WalletClient('auto', 'localhost')

// List all tokens in a specific basket
const response = await wallet.listOutputs({
  basket: 'event tickets'
})

console.log('Available tokens:', response.outputs)
```

### Listing Tokens with Full Transaction Data

```typescript
// Include complete transaction data for redemption
const response = await wallet.listOutputs({
  basket: 'event tickets',
  include: 'entire transactions'
})

console.log('Tokens with BEEF data:', response)
```

### Filtering Tokens

```typescript
// List tokens with specific criteria
const response = await wallet.listOutputs({
  basket: 'event tickets',
  tags: ['VIP', 'true']
})
```

## Redeeming Tokens

### Basic Token Redemption

```typescript
const wallet = new WalletClient('auto', 'localhost')

// First, get the token to redeem
const list = await wallet.listOutputs({
  basket: 'event tickets',
  include: 'entire transactions'
})

// Redeem the first available token
const response = await wallet.createAction({
  description: 'redeem an event ticket',
  inputBEEF: list.BEEF,
  inputs: [{
    outpoint: list.outputs[0].outpoint,
    unlockingScript: Script.fromASM('OP_TRUE').toHex(),
    inputDescription: 'event ticket'
  }]
})

console.log('Token redeemed:', response)
```

### Conditional Token Redemption

```typescript
// Redeem token with custom unlocking conditions
import { P2PKH } from '@bsv/sdk'

const response = await wallet.createAction({
  description: 'redeem premium token',
  inputBEEF: list.BEEF,
  inputs: [{
    outpoint: selectedToken.outpoint,
    unlockingScript: customUnlockingScript.toHex(),
    inputDescription: 'premium token'
  }],
  outputs: [{
    satoshis: 1,
    lockingScript: new P2PKH().lock(recipientAddress).toHex(),
    outputDescription: 'token transfer'
  }]
})
```

## Related Guides

- [Transaction Signing Methods](./transaction-signing-methods.md)
- [Direct Transaction Creation](./direct-transaction-creation.md)
