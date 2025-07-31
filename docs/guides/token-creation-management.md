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
import { WalletClient, Script } from '@bsv/sdk'

const wallet = new WalletClient('auto', 'localhost')

// Create a token with embedded custom data
const customData = 'Hello World'
const dataHex = Buffer.from(customData, 'utf8').toString('hex')
const lockingScript = Script.fromASM(`OP_RETURN ${dataHex}`).toHex()

const response = await wallet.createAction({
  description: 'create hello world token',
  outputs: [{
    satoshis: 1,
    lockingScript: lockingScript,
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
  taggedOutputs: {
    tag: 'VIP',
    value: 'true'
  }
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
    lockingScript: Script.fromASM('OP_DUP OP_HASH160 ' + recipientAddress + ' OP_EQUALVERIFY OP_CHECKSIG').toHex(),
    outputDescription: 'token transfer'
  }]
})
```

## Basket Management

### Creating Organized Token Systems

```typescript
// Group related tokens in baskets
const ticketBaskets = [
  'event tickets',
  'concert tickets', 
  'sports tickets',
  'travel tickets'
]

// Create tokens in different categories
for (const basket of ticketBaskets) {
  await wallet.createAction({
    description: `create ${basket.replace(' tickets', '')} ticket`,
    outputs: [{
      satoshis: 1,
      lockingScript: Script.fromASM('OP_NOP').toHex(),
      basket: basket,
      outputDescription: `${basket.replace(' tickets', '')} ticket`
    }]
  })
}
```

### Cross-Basket Operations

```typescript
// Move tokens between baskets by spending and recreating
const sourceTokens = await wallet.listOutputs({
  basket: 'draft tickets',
  include: 'entire transactions'
})

const response = await wallet.createAction({
  description: 'publish tickets',
  inputBEEF: sourceTokens.BEEF,
  inputs: sourceTokens.outputs.map(output => ({
    outpoint: output.outpoint,
    unlockingScript: Script.fromASM('OP_TRUE').toHex(),
    inputDescription: 'draft ticket'
  })),
  outputs: sourceTokens.outputs.map(output => ({
    satoshis: 1,
    lockingScript: Script.fromASM('OP_NOP').toHex(),
    basket: 'published tickets',
    outputDescription: 'published ticket'
  }))
})
```

## Advanced Patterns

### Token Metadata Management

```typescript
// Create tokens with rich metadata
const response = await wallet.createAction({
  description: 'create premium event ticket',
  outputs: [{
    satoshis: 1,
    lockingScript: Script.fromASM('OP_NOP').toHex(),
    basket: 'event tickets',
    outputDescription: 'premium event ticket',
    tags: {
      'event': 'BSV Conference 2024',
      'tier': 'VIP',
      'section': 'A',
      'row': '1',
      'seat': '5'
    }
  }]
})
```

### Batch Token Operations

```typescript
// Create multiple tokens in a single transaction
const outputs = Array.from({ length: 100 }, (_, i) => ({
  satoshis: 1,
  lockingScript: Script.fromASM('OP_NOP').toHex(),
  basket: 'event tickets',
  outputDescription: `ticket ${i + 1}`,
  tags: {
    'ticketNumber': (i + 1).toString(),
    'batch': 'batch-001'
  }
}))

const response = await wallet.createAction({
  description: 'create ticket batch',
  outputs
})
```

## Error Handling

### Robust Token Management

```typescript
async function safeTokenOperation(operation: string, params: any) {
  try {
    const wallet = new WalletClient('auto', 'localhost')
    
    switch (operation) {
      case 'create':
        return await wallet.createAction(params)
      case 'list':
        return await wallet.listOutputs(params)
      default:
        throw new Error(`Unknown operation: ${operation}`)
    }
  } catch (error) {
    if (error.message.includes('insufficient funds')) {
      throw new Error('Not enough funds to create token')
    } else if (error.message.includes('basket not found')) {
      throw new Error(`Basket "${params.basket}" does not exist`)
    } else {
      throw new Error(`Token operation failed: ${error.message}`)
    }
  }
}
```

## Best Practices

1. **Use Descriptive Basket Names**: Choose clear, hierarchical basket names for easy organization
2. **Include Meaningful Descriptions**: Always provide clear descriptions for your token operations
3. **Handle Errors Gracefully**: Implement proper error handling for network and wallet issues
4. **Use Tags for Metadata**: Leverage the tagging system for rich token metadata
5. **Batch Operations**: Create multiple tokens in single transactions when possible for efficiency
6. **Verify Before Redeeming**: Always check token availability before attempting redemption

## Related Guides

- [Transaction Signing Methods](./transaction-signing-methods.md)
- [Direct Transaction Creation](./direct-transaction-creation.md)
- [Security Best Practices](./security-best-practices.md)
