# Message Box Integration

Learn patterns for secure, encrypted communication that can be adapted for message box services using the BSV SDK.

## Overview

This guide demonstrates secure, encrypted communication using message box services with the BSV SDK and message box client.

> **Installation**: These examples require the message box client package:
>
> ```bash
> npm install @bsv/message-box-client
> ```

## Basic Encrypted Messaging Patterns

### Encryption and Decryption Setup

```typescript
import { WalletClient, ProtoWallet, Utils } from '@bsv/sdk'

// Initialize wallet and protowallet for encryption
const wallet = new WalletClient('auto', 'localhost')
const proto = new ProtoWallet()

// Example: Encrypt a message for a recipient
async function encryptMessage(message: string, recipientPublicKey: string): Promise<Uint8Array> {
  return await proto.encrypt({
    plaintext: Utils.toArray(message, 'utf8'),
    protocolID: [0, 'messaging'],
    keyID: '1',
    counterparty: recipientPublicKey
  })
}

// Example: Decrypt a received message
async function decryptMessage(encryptedData: Uint8Array, senderPublicKey: string): Promise<string> {
  const decrypted = await proto.decrypt({
    ciphertext: encryptedData,
    protocolID: [0, 'messaging'],
    keyID: '1',
    counterparty: senderPublicKey
  })
  
  return Utils.toString(decrypted, 'utf8')
}
```

### Simple Message Sending

```typescript
import { WalletClient, ProtoWallet, Utils } from '@bsv/sdk'
import { MessageBoxClient } from '@bsv/message-box-client'

async function sendMessage(recipientPublicKey: string, message: string): Promise<void> {
  const wallet = new WalletClient('auto', 'localhost')
  const proto = new ProtoWallet()
  const mbc = new MessageBoxClient({
    endpoint: 'https://messagebox.example.com'
  })
  
  // Get sender's public key
  const senderPublicKey = await wallet.getPublicKey({
    protocolID: [0, 'sendMessage'],
    keyID: '1',
    counterparty: 'self'
  })
  
  // Encrypt the message
  const encryptedMessage = await proto.encrypt({
    plaintext: Utils.toArray(message, 'utf8'),
    protocolID: [0, 'sendMessage'],
    keyID: '1',
    counterparty: recipientPublicKey
  })
  
  // Send encrypted message to message box
  await mbc.sendMessage({
    recipient: recipientPublicKey,
    body: Utils.toBase64(encryptedMessage)
  })
  
  console.log('Message sent successfully')
}
```

## Advanced Message Patterns

### Message with Acknowledgment

```typescript
async function sendMessageWithAck(recipientPublicKey: string, message: string): Promise<void> {
  const wallet = new WalletClient('auto', 'localhost')
  const proto = new ProtoWallet()
  const mbc = new MessageBoxClient()
  
  // Create message with acknowledgment request
  const messageData = {
    content: message,
    timestamp: Date.now(),
    requiresAck: true,
    messageId: generateMessageId()
  }
  
  // Encrypt the message
  const encryptedMessage = await proto.encrypt({
    plaintext: Utils.toArray(JSON.stringify(messageData), 'utf8'),
    protocolID: [0, 'messaging'],
    keyID: '1',
    counterparty: recipientPublicKey
  })
  
  // Send message
  await mbc.sendMessage({
    recipient: recipientPublicKey,
    body: Utils.toBase64(encryptedMessage),
    messageType: 'message_with_ack'
  })
}

function generateMessageId(): string {
  return `msg_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
}
```

### Receiving and Processing Messages

```typescript
async function receiveMessages(): Promise<any[]> {
  const wallet = new WalletClient('auto', 'localhost')
  const proto = new ProtoWallet()
  const mbc = new MessageBoxClient()
  
  // Get messages from message box
  const messages = await mbc.getMessages()
  const decryptedMessages = []
  
  for (const message of messages) {
    try {
      // Decrypt each message
      const decryptedData = await proto.decrypt({
        ciphertext: Utils.fromBase64(message.body),
        protocolID: [0, 'messaging'],
        keyID: '1',
        counterparty: message.sender
      })
      
      const messageContent = JSON.parse(Utils.toString(decryptedData, 'utf8'))
      decryptedMessages.push({
        ...messageContent,
        sender: message.sender,
        receivedAt: Date.now()
      })
      
      // Send acknowledgment if required
      if (messageContent.requiresAck) {
        await sendAcknowledgment(message.sender, messageContent.messageId)
      }
      
    } catch (error) {
      console.error('Failed to decrypt message:', error)
    }
  }
  
  return decryptedMessages
}

async function sendAcknowledgment(recipientPublicKey: string, messageId: string): Promise<void> {
  const ackData = {
    type: 'acknowledgment',
    originalMessageId: messageId,
    timestamp: Date.now()
  }
  
  await sendMessage(recipientPublicKey, JSON.stringify(ackData))
}
```

## Message Box Service Integration

### Custom Message Box Client

```typescript
class EnhancedMessageBoxClient extends MessageBoxClient {
  private retryAttempts: number
  private retryDelay: number
  
  constructor(config: {
    endpoint: string
    retryAttempts?: number
    retryDelay?: number
  }) {
    super(config)
    this.retryAttempts = config.retryAttempts || 3
    this.retryDelay = config.retryDelay || 1000
  }
  
  async sendMessageWithRetry(messageData: any): Promise<void> {
    let lastError
    
    for (let attempt = 1; attempt <= this.retryAttempts; attempt++) {
      try {
        await this.sendMessage(messageData)
        return
      } catch (error) {
        lastError = error
        if (attempt < this.retryAttempts) {
          await new Promise(resolve => setTimeout(resolve, this.retryDelay * attempt))
        }
      }
    }
    
    throw new Error(`Failed to send message after ${this.retryAttempts} attempts: ${lastError.message}`)
  }
  
  async getMessagesWithPagination(options: {
    limit?: number
    offset?: number
    filter?: string
  } = {}): Promise<any[]> {
    try {
      return await this.getMessages(options)
    } catch (error) {
      console.error('Failed to retrieve messages:', error)
      return []
    }
  }
}
```

### Multi-Service Message Routing

```typescript
class MessageRouter {
  private services: Map<string, MessageBoxClient> = new Map()
  private defaultService: string
  
  constructor(defaultServiceName: string) {
    this.defaultService = defaultServiceName
  }
  
  addService(name: string, client: MessageBoxClient): void {
    this.services.set(name, client)
  }
  
  async sendMessage(
    recipient: string,
    message: string,
    serviceName?: string
  ): Promise<void> {
    const service = this.services.get(serviceName || this.defaultService)
    if (!service) {
      throw new Error(`Message service not found: ${serviceName || this.defaultService}`)
    }
    
    const wallet = new WalletClient('auto', 'localhost')
    const proto = new ProtoWallet()
    
    // Encrypt message
    const encryptedMessage = await proto.encrypt({
      plaintext: Utils.toArray(message, 'utf8'),
      protocolID: [0, 'routing'],
      keyID: '1',
      counterparty: recipient
    })
    
    // Send via specified service
    await service.sendMessage({
      recipient,
      body: Utils.toBase64(encryptedMessage)
    })
  }
  
  async receiveFromAllServices(): Promise<any[]> {
    const allMessages = []
    
    for (const [serviceName, service] of this.services) {
      try {
        const messages = await service.getMessages()
        allMessages.push(...messages.map(msg => ({ ...msg, service: serviceName })))
      } catch (error) {
        console.error(`Failed to receive from ${serviceName}:`, error)
      }
    }
    
    return allMessages
  }
}
```

## Secure Communication Patterns

### End-to-End Encrypted Conversations

```typescript
class SecureConversation {
  private wallet: WalletClient
  private proto: ProtoWallet
  private messageBox: MessageBoxClient
  private conversationId: string
  private participants: string[]
  
  constructor(participants: string[], messageBoxEndpoint: string) {
    this.wallet = new WalletClient('auto', 'localhost')
    this.proto = new ProtoWallet()
    this.messageBox = new MessageBoxClient({ endpoint: messageBoxEndpoint })
    this.participants = participants
    this.conversationId = this.generateConversationId()
  }
  
  async sendToConversation(message: string): Promise<void> {
    const conversationMessage = {
      conversationId: this.conversationId,
      content: message,
      timestamp: Date.now(),
      sender: await this.getMyPublicKey()
    }
    
    // Send to all participants
    for (const participant of this.participants) {
      if (participant !== await this.getMyPublicKey()) {
        await this.sendEncryptedMessage(participant, conversationMessage)
      }
    }
  }
  
  async getConversationMessages(): Promise<any[]> {
    const allMessages = await this.receiveMessages()
    return allMessages.filter(msg => msg.conversationId === this.conversationId)
      .sort((a, b) => a.timestamp - b.timestamp)
  }
  
  private async sendEncryptedMessage(recipient: string, messageData: any): Promise<void> {
    const encryptedMessage = await this.proto.encrypt({
      plaintext: Utils.toArray(JSON.stringify(messageData), 'utf8'),
      protocolID: [0, 'conversation'],
      keyID: '1',
      counterparty: recipient
    })
    
    await this.messageBox.sendMessage({
      recipient,
      body: Utils.toBase64(encryptedMessage)
    })
  }
  
  private async receiveMessages(): Promise<any[]> {
    const messages = await this.messageBox.getMessages()
    const decryptedMessages = []
    
    for (const message of messages) {
      try {
        const decryptedData = await this.proto.decrypt({
          ciphertext: Utils.fromBase64(message.body),
          protocolID: [0, 'conversation'],
          keyID: '1',
          counterparty: message.sender
        })
        
        const messageContent = JSON.parse(Utils.toString(decryptedData, 'utf8'))
        decryptedMessages.push(messageContent)
      } catch (error) {
        console.error('Failed to decrypt conversation message:', error)
      }
    }
    
    return decryptedMessages
  }
  
  private async getMyPublicKey(): Promise<string> {
    return await this.wallet.getPublicKey({
      protocolID: [0, 'conversation'],
      keyID: '1',
      counterparty: 'self'
    })
  }
  
  private generateConversationId(): string {
    const participantHash = this.participants.sort().join('|')
    return `conv_${Date.now()}_${Buffer.from(participantHash).toString('hex').substring(0, 8)}`
  }
}
```

### File Sharing via Message Box

```typescript
async function shareFileViaMessageBox(
  recipientPublicKey: string,
  fileData: Uint8Array,
  fileName: string
): Promise<void> {
  const wallet = new WalletClient('auto', 'localhost')
  const proto = new ProtoWallet()
  const mbc = new MessageBoxClient()
  
  // Create file sharing message
  const fileMessage = {
    type: 'file_share',
    fileName,
    fileSize: fileData.length,
    fileData: Utils.toBase64(fileData),
    timestamp: Date.now()
  }
  
  // Encrypt the file message
  const encryptedMessage = await proto.encrypt({
    plaintext: Utils.toArray(JSON.stringify(fileMessage), 'utf8'),
    protocolID: [0, 'fileShare'],
    keyID: '1',
    counterparty: recipientPublicKey
  })
  
  // Send file via message box
  await mbc.sendMessage({
    recipient: recipientPublicKey,
    body: Utils.toBase64(encryptedMessage),
    messageType: 'file_share'
  })
}

async function receiveSharedFile(senderPublicKey: string): Promise<{
  fileName: string
  fileData: Uint8Array
} | null> {
  const proto = new ProtoWallet()
  const mbc = new MessageBoxClient()
  
  const messages = await mbc.getMessages({
    sender: senderPublicKey,
    messageType: 'file_share'
  })
  
  if (messages.length === 0) return null
  
  try {
    const decryptedData = await proto.decrypt({
      ciphertext: Utils.fromBase64(messages[0].body),
      protocolID: [0, 'fileShare'],
      keyID: '1',
      counterparty: senderPublicKey
    })
    
    const fileMessage = JSON.parse(Utils.toString(decryptedData, 'utf8'))
    
    return {
      fileName: fileMessage.fileName,
      fileData: Utils.fromBase64(fileMessage.fileData)
    }
  } catch (error) {
    console.error('Failed to decrypt shared file:', error)
    return null
  }
}
```

## Error Handling and Best Practices

### Robust Message Handling

```typescript
class RobustMessageHandler {
  private wallet: WalletClient
  private proto: ProtoWallet
  private messageBox: MessageBoxClient
  private messageQueue: any[] = []
  
  constructor(messageBoxEndpoint: string) {
    this.wallet = new WalletClient('auto', 'localhost')
    this.proto = new ProtoWallet()
    this.messageBox = new MessageBoxClient({ endpoint: messageBoxEndpoint })
  }
  
  async processMessagesWithErrorHandling(): Promise<void> {
    try {
      const messages = await this.messageBox.getMessages()
      
      for (const message of messages) {
        try {
          await this.processMessage(message)
        } catch (error) {
          console.error(`Failed to process message from ${message.sender}:`, error)
          // Add to retry queue
          this.messageQueue.push({ ...message, retryCount: 0 })
        }
      }
      
      // Process retry queue
      await this.processRetryQueue()
    } catch (error) {
      console.error('Failed to retrieve messages:', error)
    }
  }
  
  private async processMessage(message: any): Promise<void> {
    const decryptedData = await this.proto.decrypt({
      ciphertext: Utils.fromBase64(message.body),
      protocolID: [0, 'messaging'],
      keyID: '1',
      counterparty: message.sender
    })
    
    const messageContent = JSON.parse(Utils.toString(decryptedData, 'utf8'))
    
    // Process based on message type
    switch (messageContent.type) {
      case 'text':
        await this.handleTextMessage(messageContent)
        break
      case 'file_share':
        await this.handleFileShare(messageContent)
        break
      case 'acknowledgment':
        await this.handleAcknowledgment(messageContent)
        break
      default:
        console.warn('Unknown message type:', messageContent.type)
    }
  }
  
  private async processRetryQueue(): Promise<void> {
    const maxRetries = 3
    const retryableMessages = this.messageQueue.filter(msg => msg.retryCount < maxRetries)
    
    for (const message of retryableMessages) {
      try {
        await this.processMessage(message)
        // Remove from queue on success
        this.messageQueue = this.messageQueue.filter(msg => msg !== message)
      } catch (error) {
        message.retryCount++
        if (message.retryCount >= maxRetries) {
          console.error(`Giving up on message after ${maxRetries} retries`)
          this.messageQueue = this.messageQueue.filter(msg => msg !== message)
        }
      }
    }
  }
  
  private async handleTextMessage(message: any): Promise<void> {
    console.log('Received text message:', message.content)
  }
  
  private async handleFileShare(message: any): Promise<void> {
    console.log('Received file share:', message.fileName)
  }
  
  private async handleAcknowledgment(message: any): Promise<void> {
    console.log('Received acknowledgment for message:', message.originalMessageId)
  }
}
```

## Best Practices

1. **Always Encrypt Sensitive Data**: Use the SDK's encryption features for all message content
2. **Implement Acknowledgments**: Use acknowledgment patterns for critical messages
3. **Handle Errors Gracefully**: Implement retry logic and proper error handling
4. **Use Message Types**: Categorize messages for easier processing and routing
5. **Validate Message Integrity**: Verify decrypted message structure before processing
6. **Implement Rate Limiting**: Avoid overwhelming message box services
7. **Clean Up Old Messages**: Implement message cleanup strategies
8. **Use Secure Endpoints**: Always use HTTPS endpoints for message box services

## Related Guides

- [Creating Encrypted Messages](./encrypted-messages.md)
- [Authenticated API Communication](./authenticated-api-communication.md)
- [ECDH Key Exchange](../tutorials/ecdh-key-exchange.md)
