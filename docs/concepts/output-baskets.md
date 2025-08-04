# Output Baskets

## Overview

Output Baskets are an organizational system for managing UTXOs (Unspent Transaction Outputs) in the BSV SDK wallet. Think of baskets as labeled containers that group related outputs together, making it easier to track, query, and spend specific sets of UTXOs for different purposes.

## The Need for UTXO Organization

In Bitcoin applications, you often manage many different types of outputs:

- Payment outputs waiting to be spent
- Data storage outputs containing application state
- Token outputs representing digital assets
- Change outputs from previous transactions
- Specialized outputs for specific protocols

Without organization, finding and managing the right outputs becomes increasingly complex as your application scales. Output Baskets solve this problem by providing a built-in categorization system.

## How Baskets Work

### Creating Outputs with Baskets

When creating outputs through Actions, you can assign each output to a specific basket:

```typescript
await wallet.createAction({
  description: 'Organize different output types',
  outputs: [
    {
      satoshis: 10000,
      script: paymentScript,
      basket: 'pending-payments',
      description: 'Payment to supplier'
    },
    {
      satoshis: 1,
      script: tokenScript,
      basket: 'app-tokens',
      description: 'User reward token'
    },
    {
      satoshis: 5000,
      script: dataScript,
      basket: 'app-data',
      description: 'User profile data'
    }
  ]
})
```

### Retrieving Outputs by Basket

Query specific baskets to find relevant outputs:

```typescript
// Get all tokens
const { outputs: tokens } = await wallet.listOutputs({
  basket: 'app-tokens',
  includeSpent: false
})

// Get pending payments
const { outputs: payments } = await wallet.listOutputs({
  basket: 'pending-payments',
  spendable: true
})
```

### The Default Basket

Outputs created without an explicit basket assignment go to the default basket. This includes:

- Change outputs automatically created by the wallet
- Outputs where no basket is specified
- Incoming payments from external sources

```typescript
// This output goes to the default basket
await wallet.createAction({
  outputs: [{
    satoshis: 1000,
    script: someScript
    // No basket specified
  }]
})

// Query the default basket
const { outputs } = await wallet.listOutputs({
  basket: 'default'
})
```

## Basket Design Strategies

### 1. Purpose-Based Baskets

Organize by the intended use of outputs:

```typescript
const baskets = {
  'hot-wallet': 'Immediately spendable funds',
  'cold-storage': 'Long-term storage',
  'operating-funds': 'Business operations',
  'customer-deposits': 'Held for customers'
}
```

### 2. Application-Specific Baskets

Group outputs by application features:

```typescript
const gameBaskets = {
  'player-items': 'In-game items and weapons',
  'player-currency': 'In-game money',
  'achievement-badges': 'Earned achievements',
  'trade-escrow': 'Items in trading'
}
```

### 3. Time-Based Baskets

Organize by temporal characteristics:

```typescript
const timeBaskets = {
  'daily-operations': 'Today\'s transactions',
  'weekly-batch': 'This week\'s accumulation',
  'archive-2024-q1': 'Historical Q1 data'
}
```

### 4. Protocol-Specific Baskets

Separate outputs by the protocols they implement:

```typescript
const protocolBaskets = {
  'run-tokens': 'RUN protocol tokens',
  'stas-tokens': 'STAS token outputs',
  'custom-nfts': 'Custom NFT implementations'
}
```

## Advanced Basket Patterns

### Dynamic Basket Creation

Create baskets based on runtime conditions:

```typescript
async function createUserBasket(userId: string, type: string) {
  const basketName = `user-${userId}-${type}`
  
  await wallet.createAction({
    outputs: [{
      satoshis: 1000,
      script: userScript,
      basket: basketName,
      description: `Initial ${type} for user ${userId}`
    }]
  })
  
  return basketName
}
```

### Basket Migration

Move outputs between baskets by spending and recreating:

```typescript
async function migrateOutputs(fromBasket: string, toBasket: string) {
  // Get outputs from source basket
  const { outputs } = await wallet.listOutputs({
    basket: fromBasket,
    spendable: true
  })
  
  // Create action to move them
  await wallet.createAction({
    description: `Migrate ${fromBasket} to ${toBasket}`,
    inputs: outputs.map(o => ({
      txid: o.txid,
      vout: o.vout
    })),
    outputs: outputs.map(o => ({
      satoshis: o.satoshis,
      script: o.lockingScript,
      basket: toBasket,
      description: `Migrated: ${o.description}`
    }))
  })
}
```

### Basket Analytics

Track basket statistics for insights:

```typescript
async function analyzeBaskets() {
  const baskets = ['payments', 'tokens', 'data']
  const analysis = {}
  
  for (const basket of baskets) {
    const { outputs } = await wallet.listOutputs({ basket })
    
    analysis[basket] = {
      count: outputs.length,
      totalValue: outputs.reduce((sum, o) => sum + o.satoshis, 0),
      oldestOutput: outputs.sort((a, b) => a.createdAt - b.createdAt)[0]
    }
  }
  
  return analysis
}
```

## Best Practices

### 1. Consistent Naming Conventions

Establish clear naming patterns:

- Use lowercase with hyphens: `user-tokens`, not `UserTokens`
- Be descriptive but concise: `pending-withdrawals`, not `pw`
- Include context when helpful: `exchange-btc-reserves`

### 2. Document Basket Purposes

Maintain a basket registry in your application:

```typescript
const BASKETS = {
  PAYMENTS: {
    name: 'customer-payments',
    description: 'Incoming customer payments awaiting processing',
    retention: '30 days'
  },
  ESCROW: {
    name: 'trade-escrow',
    description: 'Funds held in escrow for active trades',
    retention: 'Until trade completion'
  }
} as const
```

### 3. Regular Maintenance

Implement basket cleanup strategies:

```typescript
async function cleanupOldOutputs(basket: string, daysOld: number) {
  const cutoffTime = Date.now() - (daysOld * 24 * 60 * 60 * 1000)
  
  const { outputs } = await wallet.listOutputs({
    basket,
    spendable: true
  })
  
  const oldOutputs = outputs.filter(o => o.createdAt < cutoffTime)
  
  if (oldOutputs.length > 0) {
    // Consolidate old outputs
    await wallet.createAction({
      description: `Consolidate old ${basket} outputs`,
      inputs: oldOutputs.map(o => ({ txid: o.txid, vout: o.vout })),
      outputs: [{
        satoshis: oldOutputs.reduce((sum, o) => sum + o.satoshis, 0) - 200, // Leave fee
        script: consolidationScript,
        basket: `${basket}-consolidated`,
        description: `Consolidated from ${oldOutputs.length} outputs`
      }]
    })
  }
}
```

### 4. Security Considerations

Use baskets to enhance security:

- **Segregate high-value outputs**: Keep large amounts in separate baskets
- **Implement access controls**: Different baskets for different permission levels
- **Audit trails**: Use basket names to track fund sources and destinations

## Common Use Cases

### E-commerce Platform

```typescript
const ecommerceBaskets = {
  'customer-payments': 'Incoming customer payments',
  'refund-pool': 'Funds reserved for potential refunds',
  'merchant-payouts': 'Pending merchant settlements',
  'platform-fees': 'Collected platform fees',
  'dispute-escrow': 'Funds held during disputes'
}
```

### Gaming Application

```typescript
const gamingBaskets = {
  'player-wallets': 'Individual player balances',
  'tournament-prizes': 'Prize pools for tournaments',
  'item-marketplace': 'NFT game items for sale',
  'staking-rewards': 'Rewards awaiting distribution',
  'game-treasury': 'Game operator funds'
}
```

### DeFi Application

```typescript
const defiBaskets = {
  'liquidity-pools': 'AMM liquidity provisions',
  'collateral-vaults': 'Locked collateral for loans',
  'yield-farming': 'Staked tokens earning yield',
  'governance-tokens': 'Voting power tokens',
  'protocol-treasury': 'Protocol-owned liquidity'
}
```

## Performance Considerations

### Basket Sizing

- **Small baskets** (< 100 outputs): Fast queries, ideal for hot wallets
- **Medium baskets** (100-1000 outputs): Good for active operations
- **Large baskets** (> 1000 outputs): Consider partitioning or archiving

### Query Optimization

```typescript
// Efficient: Query specific basket with filters
const activeTokens = await wallet.listOutputs({
  basket: 'app-tokens',
  spendable: true,
  limit: 50
})

// Inefficient: Query all then filter
const allOutputs = await wallet.listOutputs({})
const tokens = allOutputs.outputs.filter(o => o.basket === 'app-tokens')
```

### Basket Partitioning

For high-volume applications:

```typescript
function getPartitionedBasket(baseBasket: string, partitionKey: string): string {
  const partition = hashToPartition(partitionKey, 10) // 10 partitions
  return `${baseBasket}-p${partition}`
}

// Usage
const userBasket = getPartitionedBasket('user-funds', userId)
```

## Related Concepts

- [Actions](./actions.md) - Creating outputs with basket assignments
- [Transaction Structure](./transaction-structure.md) - Understanding outputs in transactions
- [Wallet Integration](./wallet-integration.md) - How baskets integrate with wallet functionality
- [BEEF Format](./beef.md) - How basket metadata is preserved in transaction data
