# StorageUploader Patterns

Learn efficient patterns for uploading and managing files using the BSV SDK's storage capabilities.

## Overview

The StorageUploader provides a streamlined interface for uploading files to decentralized storage services. This guide demonstrates practical patterns for file management, retention strategies, and error handling.

## Basic Upload Patterns

### Simple File Upload

```typescript
import { WalletClient, StorageUploader } from '@bsv/sdk'

async function uploadTextFile(content: string): Promise<any> {
  const wallet = new WalletClient('auto', 'localhost')
  
  const uploader = new StorageUploader({
    storageURL: 'https://nanostore.babbage.systems',
    wallet
  })
  
  // Create file data in the correct format
  const fileData = new TextEncoder().encode(content)
  const file = {
    data: Array.from(fileData),
    type: 'text/plain'
  }
  
  const response = await uploader.publishFile({
    file,
    retentionPeriod: 180 // minutes
  })
  
  return response
}

// Usage
const result = await uploadTextFile('Hello, BSV storage!')
console.log('File uploaded:', result)
```

### Binary File Upload

```typescript
async function uploadBinaryFile(fileBuffer: Uint8Array, mimeType: string): Promise<any> {
  const wallet = new WalletClient('auto', 'localhost')
  
  const uploader = new StorageUploader({
    storageURL: 'https://nanostore.babbage.systems',
    wallet
  })
  
  const response = await uploader.publishFile({
    file: {
      data: fileBuffer,
      type: mimeType
    },
    retentionPeriod: 1440 // 24 hours
  })
  
  return response
}

// Upload an image
const imageBuffer = new Uint8Array(/* your image data */)
await uploadBinaryFile(imageBuffer, 'image/png')
```

## Advanced Upload Patterns

### File Upload with Metadata

```typescript
class MetadataUploader {
  private uploader: StorageUploader
  
  constructor(storageURL: string) {
    const wallet = new WalletClient('auto', 'localhost')
    this.uploader = new StorageUploader({
      storageURL,
      wallet
    })
  }
  
  async uploadWithMetadata(
    fileData: Uint8Array,
    metadata: {
      filename: string
      description?: string
      tags?: string[]
      author?: string
    }
  ): Promise<any> {
    // Create metadata header
    const metadataHeader = {
      version: '1.0',
      timestamp: Date.now(),
      ...metadata
    }
    
    // Combine metadata and file data
    const metadataBytes = Array.from(new TextEncoder().encode(JSON.stringify(metadataHeader)))
    const separator = Array.from(new TextEncoder().encode('\n---FILE_DATA---\n'))
    
    const combinedData = new Uint8Array(
      metadataBytes.length + separator.length + fileData.length
    )
    combinedData.set(metadataBytes, 0)
    combinedData.set(separator, metadataBytes.length)
    combinedData.set(fileData, metadataBytes.length + separator.length)
    
    return await this.uploader.publishFile({
      file: {
        data: combinedData,
        type: 'application/octet-stream'
      },
      retentionPeriod: 2880 // 48 hours
    })
  }
}

// Usage
const metadataUploader = new MetadataUploader('https://nanostore.babbage.systems')
await metadataUploader.uploadWithMetadata(fileData, {
  filename: 'document.pdf',
  description: 'Important contract document',
  tags: ['legal', 'contract', 'business'],
  author: 'John Doe'
})
```

### Batch File Upload

```typescript
class BatchUploader {
  private uploader: StorageUploader
  private maxConcurrent: number
  
  constructor(storageURL: string, maxConcurrent: number = 5) {
    const wallet = new WalletClient('auto', 'localhost')
    this.uploader = new StorageUploader({
      storageURL,
      wallet
    })
    this.maxConcurrent = maxConcurrent
  }
  
  async uploadBatch(files: Array<{
    data: Uint8Array
    type: string
    filename?: string
    retentionPeriod?: number
  }>): Promise<any[]> {
    const results = []
    
    // Process files in batches to avoid overwhelming the service
    for (let i = 0; i < files.length; i += this.maxConcurrent) {
      const batch = files.slice(i, i + this.maxConcurrent)
      
      const batchPromises = batch.map(async (file, index) => {
        try {
          const response = await this.uploader.publishFile({
            file: {
              data: file.data,
              type: file.type
            },
            retentionPeriod: file.retentionPeriod || 180
          })
          
          return {
            success: true,
            filename: file.filename || `file_${i + index}`,
            response
          }
        } catch (error) {
          return {
            success: false,
            filename: file.filename || `file_${i + index}`,
            error: error.message
          }
        }
      })
      
      const batchResults = await Promise.all(batchPromises)
      results.push(...batchResults)
      
      // Add delay between batches
      if (i + this.maxConcurrent < files.length) {
        await new Promise(resolve => setTimeout(resolve, 1000))
      }
    }
    
    return results
  }
}

// Usage
const batchUploader = new BatchUploader('https://nanostore.babbage.systems')
const files = [
  { data: fileData1, type: 'text/plain', filename: 'file1.txt' },
  { data: fileData2, type: 'image/png', filename: 'image.png' },
  { data: fileData3, type: 'application/pdf', filename: 'document.pdf' }
]

const results = await batchUploader.uploadBatch(files)
console.log('Batch upload results:', results)
```

## Retention Management

### Smart Retention Strategies

```typescript
class RetentionManager {
  private uploader: StorageUploader
  
  constructor(storageURL: string) {
    const wallet = new WalletClient('auto', 'localhost')
    this.uploader = new StorageUploader({
      storageURL,
      wallet
    })
  }
  
  async uploadWithSmartRetention(
    fileData: Uint8Array,
    fileType: string,
    category: 'temporary' | 'standard' | 'long-term' | 'permanent'
  ): Promise<any> {
    const retentionPeriods = {
      temporary: 60,      // 1 hour
      standard: 1440,     // 24 hours
      'long-term': 10080, // 7 days
      permanent: 525600   // 1 year
    }
    
    return await this.uploader.publishFile({
      file: {
        data: fileData,
        type: fileType
      },
      retentionPeriod: retentionPeriods[category]
    })
  }
  
  async uploadWithCustomRetention(
    fileData: Uint8Array,
    fileType: string,
    expirationDate: Date
  ): Promise<any> {
    const now = new Date()
    const minutesUntilExpiration = Math.ceil((expirationDate.getTime() - now.getTime()) / (1000 * 60))
    
    if (minutesUntilExpiration <= 0) {
      throw new Error('Expiration date must be in the future')
    }
    
    return await this.uploader.publishFile({
      file: {
        data: fileData,
        type: fileType
      },
      retentionPeriod: minutesUntilExpiration
    })
  }
}

// Usage
const retentionManager = new RetentionManager('https://nanostore.babbage.systems')

// Upload with category-based retention
await retentionManager.uploadWithSmartRetention(fileData, 'text/plain', 'long-term')

// Upload with specific expiration date
const expirationDate = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days from now
await retentionManager.uploadWithCustomRetention(fileData, 'image/png', expirationDate)
```

### Renewal Strategies

```typescript
class FileRenewalManager {
  private uploader: StorageUploader
  private fileRegistry: Map<string, {
    fileId: string
    originalRetention: number
    uploadDate: Date
    renewalCount: number
  }> = new Map()
  
  constructor(storageURL: string) {
    const wallet = new WalletClient('auto', 'localhost')
    this.uploader = new StorageUploader({
      storageURL,
      wallet
    })
  }
  
  async uploadWithRenewal(
    fileData: Uint8Array,
    fileType: string,
    initialRetention: number,
    autoRenew: boolean = false
  ): Promise<string> {
    const response = await this.uploader.publishFile({
      file: {
        data: fileData,
        type: fileType
      },
      retentionPeriod: initialRetention
    })
    
    const fileId = response.fileId || response.txid || `file_${Date.now()}`
    
    this.fileRegistry.set(fileId, {
      fileId,
      originalRetention: initialRetention,
      uploadDate: new Date(),
      renewalCount: 0
    })
    
    if (autoRenew) {
      this.scheduleRenewal(fileId, initialRetention)
    }
    
    return fileId
  }
  
  private scheduleRenewal(fileId: string, retentionMinutes: number): void {
    const renewalTime = (retentionMinutes * 60 * 1000) * 0.8 // Renew at 80% of retention period
    
    setTimeout(async () => {
      try {
        await this.renewFile(fileId)
      } catch (error) {
        console.error(`Failed to auto-renew file ${fileId}:`, error)
      }
    }, renewalTime)
  }
  
  async renewFile(fileId: string, extensionMinutes?: number): Promise<void> {
    const fileInfo = this.fileRegistry.get(fileId)
    if (!fileInfo) {
      throw new Error(`File ${fileId} not found in registry`)
    }
    
    const extension = extensionMinutes || fileInfo.originalRetention
    
    // Note: Actual renewal implementation depends on storage service API
    // This is a conceptual example
    try {
      await this.uploader.renewFile({
        fileId,
        additionalRetention: extension
      })
      
      fileInfo.renewalCount++
      this.fileRegistry.set(fileId, fileInfo)
      
      console.log(`File ${fileId} renewed for ${extension} minutes (renewal #${fileInfo.renewalCount})`)
    } catch (error) {
      console.error(`Failed to renew file ${fileId}:`, error)
      throw error
    }
  }
}
```

## Storage Service Integration

### Multi-Service Uploader

```typescript
class MultiServiceUploader {
  private uploaders: Map<string, StorageUploader> = new Map()
  private defaultService: string
  
  constructor(services: Array<{
    name: string
    url: string
    isDefault?: boolean
  }>) {
    const wallet = new WalletClient('auto', 'localhost')
    
    for (const service of services) {
      this.uploaders.set(service.name, new StorageUploader({
        storageURL: service.url,
        wallet
      }))
      
      if (service.isDefault) {
        this.defaultService = service.name
      }
    }
    
    if (!this.defaultService && services.length > 0) {
      this.defaultService = services[0].name
    }
  }
  
  async uploadWithRedundancy(
    fileData: Uint8Array,
    fileType: string,
    services: string[] = [],
    retentionPeriod: number = 180
  ): Promise<any[]> {
    const targetServices = services.length > 0 ? services : [this.defaultService]
    const results = []
    
    for (const serviceName of targetServices) {
      const uploader = this.uploaders.get(serviceName)
      if (!uploader) {
        results.push({
          service: serviceName,
          success: false,
          error: `Service not found: ${serviceName}`
        })
        continue
      }
      
      try {
        const response = await uploader.publishFile({
          file: {
            data: fileData,
            type: fileType
          },
          retentionPeriod
        })
        
        results.push({
          service: serviceName,
          success: true,
          response
        })
      } catch (error) {
        results.push({
          service: serviceName,
          success: false,
          error: error.message
        })
      }
    }
    
    return results
  }
  
  async uploadWithFallback(
    fileData: Uint8Array,
    fileType: string,
    retentionPeriod: number = 180
  ): Promise<any> {
    const serviceNames = Array.from(this.uploaders.keys())
    
    for (const serviceName of serviceNames) {
      try {
        const uploader = this.uploaders.get(serviceName)
        const response = await uploader.publishFile({
          file: {
            data: fileData,
            type: fileType
          },
          retentionPeriod
        })
        
        console.log(`Successfully uploaded to ${serviceName}`)
        return { service: serviceName, response }
      } catch (error) {
        console.warn(`Failed to upload to ${serviceName}:`, error.message)
      }
    }
    
    throw new Error('All storage services failed')
  }
}

// Usage
const multiUploader = new MultiServiceUploader([
  { name: 'primary', url: 'https://primary-storage.example.com', isDefault: true },
  { name: 'backup', url: 'https://backup-storage.example.com' },
  { name: 'archive', url: 'https://archive-storage.example.com' }
])

// Upload with redundancy
const redundantResults = await multiUploader.uploadWithRedundancy(
  fileData, 
  'text/plain', 
  ['primary', 'backup']
)

// Upload with automatic fallback
const fallbackResult = await multiUploader.uploadWithFallback(fileData, 'image/png')
```

## Error Handling and Monitoring

### Robust Upload Handler

```typescript
class RobustUploader {
  private uploader: StorageUploader
  private retryAttempts: number
  private retryDelay: number
  
  constructor(
    storageURL: string,
    options: {
      retryAttempts?: number
      retryDelay?: number
    } = {}
  ) {
    const wallet = new WalletClient('auto', 'localhost')
    this.uploader = new StorageUploader({
      storageURL,
      wallet
    })
    this.retryAttempts = options.retryAttempts || 3
    this.retryDelay = options.retryDelay || 1000
  }
  
  async uploadWithRetry(
    fileData: Uint8Array,
    fileType: string,
    retentionPeriod: number = 180
  ): Promise<any> {
    let lastError
    
    for (let attempt = 1; attempt <= this.retryAttempts; attempt++) {
      try {
        const response = await this.uploader.publishFile({
          file: {
            data: fileData,
            type: fileType
          },
          retentionPeriod
        })
        
        if (attempt > 1) {
          console.log(`Upload succeeded on attempt ${attempt}`)
        }
        
        return response
      } catch (error) {
        lastError = error
        console.warn(`Upload attempt ${attempt} failed:`, error.message)
        
        if (attempt < this.retryAttempts) {
          await new Promise(resolve => setTimeout(resolve, this.retryDelay * attempt))
        }
      }
    }
    
    throw new Error(`Upload failed after ${this.retryAttempts} attempts: ${lastError.message}`)
  }
  
  async uploadWithValidation(
    fileData: Uint8Array,
    fileType: string,
    retentionPeriod: number = 180,
    validator?: (response: any) => boolean
  ): Promise<any> {
    if (fileData.length === 0) {
      throw new Error('Cannot upload empty file')
    }
    
    const maxSize = 10 * 1024 * 1024 // 10MB limit
    if (fileData.length > maxSize) {
      throw new Error(`File too large: ${fileData.length} bytes (max: ${maxSize})`)
    }
    
    const response = await this.uploadWithRetry(fileData, fileType, retentionPeriod)
    
    if (validator && !validator(response)) {
      throw new Error('Upload validation failed')
    }
    
    return response
  }
}

// Usage
const robustUploader = new RobustUploader('https://nanostore.babbage.systems', {
  retryAttempts: 5,
  retryDelay: 2000
})

// Upload with automatic retry and validation
const response = await robustUploader.uploadWithValidation(
  fileData,
  'application/pdf',
  1440,
  (response) => response && response.txid // Validate response has transaction ID
)
```

## Performance Optimization

### Chunked Upload for Large Files

```typescript
class ChunkedUploader {
  private uploader: StorageUploader
  private chunkSize: number
  
  constructor(storageURL: string, chunkSize: number = 1024 * 1024) { // 1MB chunks
    const wallet = new WalletClient('auto', 'localhost')
    this.uploader = new StorageUploader({
      storageURL,
      wallet
    })
    this.chunkSize = chunkSize
  }
  
  async uploadLargeFile(
    fileData: Uint8Array,
    fileType: string,
    retentionPeriod: number = 180,
    onProgress?: (percentage: number) => void
  ): Promise<any[]> {
    const chunks = this.createChunks(fileData)
    const results = []
    
    for (let i = 0; i < chunks.length; i++) {
      const chunk = chunks[i]
      const chunkMetadata = {
        chunkIndex: i,
        totalChunks: chunks.length,
        originalSize: fileData.length,
        fileType
      }
      
      try {
        const response = await this.uploader.publishFile({
          file: {
            data: this.addChunkMetadata(chunk, chunkMetadata),
            type: 'application/octet-stream'
          },
          retentionPeriod
        })
        
        results.push({
          chunkIndex: i,
          success: true,
          response
        })
        
        if (onProgress) {
          onProgress(((i + 1) / chunks.length) * 100)
        }
      } catch (error) {
        results.push({
          chunkIndex: i,
          success: false,
          error: error.message
        })
      }
    }
    
    return results
  }
  
  private createChunks(data: Uint8Array): Uint8Array[] {
    const chunks = []
    for (let i = 0; i < data.length; i += this.chunkSize) {
      chunks.push(data.slice(i, i + this.chunkSize))
    }
    return chunks
  }
  
  private addChunkMetadata(chunk: Uint8Array, metadata: any): Uint8Array {
    const metadataBytes = Array.from(new TextEncoder().encode(JSON.stringify(metadata)))
    const separator = Array.from(new TextEncoder().encode('\n---CHUNK---\n'))
    
    const result = new Uint8Array(metadataBytes.length + separator.length + chunk.length)
    result.set(metadataBytes, 0)
    result.set(separator, metadataBytes.length)
    result.set(chunk, metadataBytes.length + separator.length)
    
    return result
  }
}

// Usage
const chunkedUploader = new ChunkedUploader('https://nanostore.babbage.systems')
const largeFileResults = await chunkedUploader.uploadLargeFile(
  largeFileData,
  'video/mp4',
  7200, // 5 days retention
  (progress) => console.log(`Upload progress: ${progress.toFixed(1)}%`)
)
```

## Best Practices

1. **Choose Appropriate Retention Periods**: Match retention to your actual needs to optimize costs
2. **Implement Error Handling**: Always handle network failures and service errors gracefully
3. **Use Batch Operations**: Upload multiple files efficiently when possible
4. **Validate File Size**: Check file sizes before uploading to avoid failures
5. **Monitor Upload Progress**: Provide feedback for large file uploads
6. **Plan for Redundancy**: Consider multi-service uploads for critical files
7. **Implement Renewal Logic**: Plan for file retention management in long-term applications
8. **Optimize for Network**: Use appropriate chunk sizes for your network conditions

## Related Guides

- [File Upload/Download Features](./file-upload-download.md)
- [UHRP Storage Tutorial](../tutorials/uhrp-storage.md)
- [Error Handling](../tutorials/error-handling.md)
