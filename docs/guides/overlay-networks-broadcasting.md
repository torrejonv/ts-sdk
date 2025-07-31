# Overlay Networks and Topic Broadcasting

Learn how to broadcast transactions to overlay networks and work with topic-based communication channels.

## Overview

Overlay networks in BSV provide specialized communication channels for different types of applications. This guide demonstrates how to broadcast transactions to specific topics and integrate with overlay services.

## Basic Topic Broadcasting

### Simple Transaction Broadcasting

```typescript
import { WalletClient, TopicBroadcaster, Transaction, Utils, Script } from '@bsv/sdk'

const wallet = new WalletClient('auto', 'localhost')

// Create a transaction
const response = await wallet.createAction({
  description: 'create hello world token',
  outputs: [{
    satoshis: 1,
    lockingScript: Script.fromASM('OP_NOP').toHex(),
    basket: 'hello world tokens',
    outputDescription: 'hello world token'
  }]
})

// Convert response to Transaction object for broadcasting
const tx = Transaction.fromBEEF(response.tx as number[])

const broadcaster = new TopicBroadcaster(['tm_helloworld'])

await broadcaster.broadcast(tx)
```

### Broadcasting with Custom Topics

```typescript
// Broadcast to application-specific topics
const topics = [
  'social_media_posts',
  'marketplace_listings', 
  'identity_certificates',
  'file_storage_refs'
]

for (const topic of topics) {
  const broadcaster = new TopicBroadcaster([topic])
  
  await broadcaster.broadcast(Transaction.fromBEEF(transactionBEEF))
}
```

## Advanced Broadcasting Patterns

### Token Creation with Overlay Broadcasting

```typescript
import { WalletClient, Script, TopicBroadcaster } from '@bsv/sdk'

async function createAndBroadcastToken(message: string, topic: string) {
  const wallet = new WalletClient('auto', 'localhost')
  
  // Create a token with embedded message data
  const messageHex = Buffer.from(message, 'utf8').toString('hex')
  const lockingScript = Script.fromASM(`OP_RETURN ${messageHex}`).toHex()

  // Create the transaction
  const response = await wallet.createAction({
    description: 'create and broadcast token',
    outputs: [{
      satoshis: 1,
      lockingScript: lockingScript,
      basket: 'broadcast tokens',
      outputDescription: 'broadcast token'
    }]
  })

  // Broadcast to overlay network
  const broadcaster = new TopicBroadcaster([topic])
  
  await broadcaster.broadcast(Transaction.fromBEEF(response.tx as number[]))
  
  return response
}

// Usage
await createAndBroadcastToken('Hello BSV Network!', 'tm_helloworld')
```

### Multi-Topic Broadcasting

```typescript
async function broadcastToMultipleTopics(tx: Transaction, topics: string[]) {
  const results = []
  
  for (const topic of topics) {
    try {
      const broadcaster = new TopicBroadcaster([topic])
      const result = await broadcaster.broadcast(tx)
      results.push({ topic, success: true, result })
    } catch (error: any) {
      results.push({ topic, success: false, error: error.message })
    }
  }
  
  return results
}

// Broadcast transaction to multiple overlay networks
const topics = ['tm_social', 'tm_marketplace', 'tm_notifications']
const tx = Transaction.fromBEEF(response.tx as number[])
const results = await broadcastToMultipleTopics(tx, topics)
```

## Topic Configuration

### Custom Topic Broadcaster Setup

```typescript
class CustomTopicBroadcaster extends TopicBroadcaster {
  constructor(topic: string, config?: {
    retryAttempts?: number
    timeout?: number
    endpoints?: string[]
  }) {
    super({
      topic,
      ...config
    })
  }
  
  async broadcastWithRetry(beef: number[], maxRetries = 3): Promise<any> {
    let lastError
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await this.broadcast(beef)
      } catch (error) {
        lastError = error
        if (attempt < maxRetries) {
          await new Promise(resolve => setTimeout(resolve, 1000 * attempt))
        }
      }
    }
    
    throw new Error(`Failed to broadcast after ${maxRetries} attempts: ${lastError.message}`)
  }
}
```

### Topic-Specific Broadcasting

```typescript
// Configure broadcasters for different application domains
const broadcasters = {
  social: new TopicBroadcaster(['tm_social_media']),
  marketplace: new TopicBroadcaster(['tm_marketplace']),
  identity: new TopicBroadcaster(['tm_identity']),
  storage: new TopicBroadcaster(['tm_file_storage'])
}

async function broadcastByCategory(category: string, tx: Transaction) {
  const broadcaster = broadcasters[category]
  if (!broadcaster) {
    throw new Error(`Unknown category: ${category}`)
  }
  
  return await broadcaster.broadcast(tx)
}
```

## Overlay Network Integration

### Application-Specific Overlays

```typescript
// Create application-specific overlay integration
class ApplicationOverlay {
  private wallet: WalletClient
  private topic: string
  private broadcaster: TopicBroadcaster
  
  constructor(applicationId: string) {
    this.wallet = new WalletClient('auto', 'localhost')
    this.topic = `app_${applicationId}`
    this.broadcaster = new TopicBroadcaster([this.topic])
  }
  
  async publishData(data: any, description: string): Promise<any> {
    // Create transaction with application data
    const response = await this.wallet.createAction({
      description,
      outputs: [{
        satoshis: 1,
        lockingScript: this.createDataScript(data),
        basket: `${this.topic}_data`,
        outputDescription: description
      }]
    })
    
    // Broadcast to application overlay
    await this.broadcaster.broadcast(Transaction.fromBEEF(response.tx as number[]))
    
    return response
  }
  
  private createDataScript(data: any): string {
    // Implement custom data encoding for your application
    const encodedData = JSON.stringify(data)
    return Script.fromASM(`OP_RETURN ${Buffer.from(encodedData).toString('hex')}`).toHex()
  }
}

// Usage
const myApp = new ApplicationOverlay('social_network')
await myApp.publishData({ post: 'Hello BSV!', timestamp: Date.now() }, 'social media post')
```

### Cross-Overlay Communication

```typescript
// Implement cross-overlay message routing
class OverlayRouter {
  private broadcasters: Map<string, TopicBroadcaster> = new Map()
  
  addOverlay(name: string, topic: string) {
    this.broadcasters.set(name, new TopicBroadcaster([topic]))
  }
  
  async routeMessage(message: any, targets: string[]): Promise<void> {
    const routingTransaction = await this.createRoutingTransaction(message, targets)
    
    // Broadcast to all target overlays
    const promises = targets.map(async (target) => {
      const broadcaster = this.broadcasters.get(target)
      if (broadcaster) {
        return broadcaster.broadcast(Transaction.fromBEEF(routingTransaction.tx as number[]))
      }
    })
    
    await Promise.all(promises)
  }
  
  private async createRoutingTransaction(message: any, targets: string[]): Promise<any> {
    const wallet = new WalletClient('auto', 'localhost')
    
    const response = await wallet.createAction({
      description: 'cross-overlay message routing',
      outputs: targets.map(target => ({
        satoshis: 1,
        lockingScript: this.createRoutingScript(message, target),
        basket: 'routing_messages',
        outputDescription: `message to ${target}`
      }))
    })
    
    return response
  }
  
  private createRoutingScript(message: any, target: string): string {
    const routingData = { target, message, timestamp: Date.now() }
    return Script.fromASM(`OP_RETURN ${Buffer.from(JSON.stringify(routingData)).toString('hex')}`).toHex()
  }
}
```

## Real-Time Communication

### Event-Driven Broadcasting

```typescript
class EventBroadcaster {
  private wallet: WalletClient
  private eventTopic: string
  private broadcaster: TopicBroadcaster
  
  constructor(eventType: string) {
    this.wallet = new WalletClient('auto', 'localhost')
    this.eventTopic = `events_${eventType}`
    this.broadcaster = new TopicBroadcaster([this.eventTopic])
  }
  
  async broadcastEvent(eventData: {
    type: string
    payload: any
    timestamp?: number
  }): Promise<void> {
    const event = {
      ...eventData,
      timestamp: eventData.timestamp || Date.now(),
      id: this.generateEventId()
    }
    
    const response = await this.wallet.createAction({
      description: `broadcast ${event.type} event`,
      outputs: [{
        satoshis: 1,
        lockingScript: this.createEventScript(event),
        basket: 'event_broadcasts',
        outputDescription: `${event.type} event`
      }]
    })
    
    await this.broadcaster.broadcast(Transaction.fromBEEF(response.tx as number[]))
  }
  
  private createEventScript(event: any): string {
    return Script.fromASM(`OP_RETURN ${Buffer.from(JSON.stringify(event)).toString('hex')}`).toHex()
  }
  
  private generateEventId(): string {
    return `${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
  }
}

// Usage
const userEventBroadcaster = new EventBroadcaster('user_actions')
await userEventBroadcaster.broadcastEvent({
  type: 'user_login',
  payload: { userId: '12345', location: 'mobile_app' }
})
```

## Error Handling and Monitoring

### Robust Broadcasting with Fallbacks

```typescript
class ResilientBroadcaster {
  private primaryBroadcaster: TopicBroadcaster
  private fallbackBroadcasters: TopicBroadcaster[]
  
  constructor(primaryTopic: string, fallbackTopics: string[] = []) {
    this.primaryBroadcaster = new TopicBroadcaster([primaryTopic])
    this.fallbackBroadcasters = fallbackTopics.map(topic => 
      new TopicBroadcaster([topic])
    )
  }
  
  async broadcast(tx: Transaction): Promise<void> {
    try {
      await this.primaryBroadcaster.broadcast(tx)
      return
    } catch (primaryError: any) {
      console.warn('Primary broadcaster failed:', primaryError.message)
      
      // Try fallback broadcasters
      for (const fallback of this.fallbackBroadcasters) {
        try {
          await fallback.broadcast(tx)
          console.log('Successfully broadcast via fallback')
          return
        } catch (fallbackError: any) {
          console.warn('Fallback broadcaster failed:', fallbackError.message)
        }
      }
      
      throw new Error('All broadcasters failed')
    }
  }
}
```

## Best Practices

1. **Choose Appropriate Topics**: Use descriptive, hierarchical topic names that reflect your application's purpose
2. **Implement Fallbacks**: Always have backup broadcasting strategies for critical applications
3. **Monitor Performance**: Track broadcasting success rates and response times
4. **Batch When Possible**: Group related transactions for efficient overlay usage
5. **Handle Errors Gracefully**: Implement retry logic and graceful degradation
6. **Use BEEF Format**: Always convert transactions to BEEF format for optimal broadcasting
7. **Topic Namespacing**: Use consistent naming conventions for your application topics

## Related Guides

- [Token Creation and Management](./token-creation-management.md)
- [Custom Broadcasters](./custom-broadcasters.md)
- [Transaction Broadcasting](../tutorials/transaction-broadcasting.md)
- [Advanced Transaction Signing](./advanced-transaction-signing.md)
