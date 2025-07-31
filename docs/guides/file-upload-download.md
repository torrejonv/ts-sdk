# Implementing File Upload/Download Features

Learn how to build robust file upload and download systems using UHRP storage with the BSV TypeScript SDK.

## Problem

You need to implement decentralized file storage and retrieval in your application with integrity verification and proper error handling.

## Solution

Use StorageUploader and StorageDownloader with proper file management, batch operations, and error handling.

### Basic File Upload System

```typescript
import { StorageUploader, WalletClient } from '@bsv/sdk'

class FileUploadService {
  private uploader: StorageUploader
  
  constructor(storageURL: string, wallet?: WalletClient) {
    this.uploader = new StorageUploader({
      storageURL,
      wallet: wallet || new WalletClient('auto', 'localhost')
    })
  }
  
  async uploadFile(
    fileData: File | Uint8Array,
    retentionDays: number = 30
  ): Promise<string> {
    let data: number[]
    let mimeType: string
    
    if (fileData instanceof File) {
      const arrayBuffer = await fileData.arrayBuffer()
      data = Array.from(new Uint8Array(arrayBuffer))
      mimeType = fileData.type
    } else {
      data = Array.from(fileData)
      mimeType = 'application/octet-stream'
    }
    
    const result = await this.uploader.publishFile({
      file: { data, type: mimeType },
      retentionPeriod: retentionDays * 24 * 60
    })
    
    return result.uhrpURL
  }
}
```

### Robust File Download System

```typescript
import { StorageDownloader } from '@bsv/sdk'

class FileDownloadService {
  private downloader: StorageDownloader
  
  constructor(networkPreset: 'mainnet' | 'testnet' = 'mainnet') {
    this.downloader = new StorageDownloader({ networkPreset })
  }
  
  async downloadFile(uhrpUrl: string): Promise<{
    data: Uint8Array
    mimeType: string | null
    verified: boolean
  }> {
    try {
      const result = await this.downloader.download(uhrpUrl)
      
      return {
        data: new Uint8Array(result.data),
        mimeType: result.mimeType,
        verified: true // Hash verification is automatic
      }
    } catch (error) {
      console.error('Download failed:', error)
      throw new Error(`Failed to download file: ${error.message}`)
    }
  }
  
  async downloadWithRetry(
    uhrpUrl: string,
    maxRetries: number = 3
  ): Promise<Uint8Array> {
    let lastError: Error
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const result = await this.downloadFile(uhrpUrl)
        return result.data
      } catch (error) {
        lastError = error
        if (attempt < maxRetries) {
          await new Promise(resolve => setTimeout(resolve, 1000 * attempt))
        }
      }
    }
    
    throw lastError!
  }
}
```

### Batch File Operations

```typescript
import { StorageUploader, StorageDownloader, WalletClient } from '@bsv/sdk'

class BatchFileService {
  private uploader: StorageUploader
  private downloader: StorageDownloader
  
  constructor(storageURL: string, wallet?: WalletClient) {
    this.uploader = new StorageUploader({
      storageURL,
      wallet: wallet || new WalletClient('auto', 'localhost')
    })
    this.downloader = new StorageDownloader()
  }
  
  async batchUpload(files: Array<{
    data: Uint8Array
    name: string
    type: string
  }>): Promise<Array<{
    name: string
    uhrpUrl?: string
    error?: string
  }>> {
    const results = await Promise.allSettled(
      files.map(async (file) => {
        const result = await this.uploader.publishFile({
          file: { data: Array.from(file.data), type: file.type },
          retentionPeriod: 30 * 24 * 60 // 30 days
        })
        return { name: file.name, uhrpUrl: result.uhrpURL }
      })
    )
    
    return results.map((result, index) => {
      if (result.status === 'fulfilled') {
        return result.value
      } else {
        return {
          name: files[index].name,
          error: result.reason.message
        }
      }
    })
  }
  
  async batchDownload(uhrpUrls: string[]): Promise<Array<{
    url: string
    data?: Uint8Array
    error?: string
  }>> {
    const results = await Promise.allSettled(
      uhrpUrls.map(url => this.downloader.download(url))
    )
    
    return results.map((result, index) => {
      const url = uhrpUrls[index]
      
      if (result.status === 'fulfilled') {
        return {
          url,
          data: new Uint8Array(result.value.data)
        }
      } else {
        return {
          url,
          error: result.reason.message
        }
      }
    })
  }
}
```

### File Management with Metadata

```typescript
interface FileRecord {
  uhrpUrl: string
  originalName: string
  mimeType: string
  size: number
  uploadDate: Date
  expiryDate: Date
  tags: string[]
}

class FileManager {
  private files: Map<string, FileRecord> = new Map()
  private uploader: StorageUploader
  private downloader: StorageDownloader
  
  constructor(storageURL: string, wallet?: WalletClient) {
    this.uploader = new StorageUploader({
      storageURL,
      wallet: wallet || new WalletClient('auto', 'localhost')
    })
    this.downloader = new StorageDownloader()
  }
  
  async uploadWithMetadata(
    fileData: Uint8Array,
    fileName: string,
    mimeType: string,
    tags: string[] = [],
    retentionDays: number = 30
  ): Promise<FileRecord> {
    const result = await this.uploader.publishFile({
      file: { data: Array.from(fileData), type: mimeType },
      retentionPeriod: retentionDays * 24 * 60
    })
    
    const record: FileRecord = {
      uhrpUrl: result.uhrpURL,
      originalName: fileName,
      mimeType,
      size: fileData.length,
      uploadDate: new Date(),
      expiryDate: new Date(Date.now() + retentionDays * 24 * 60 * 60 * 1000),
      tags
    }
    
    this.files.set(result.uhrpURL, record)
    return record
  }
  
  async renewFile(uhrpUrl: string, additionalDays: number): Promise<void> {
    await this.uploader.renewFile(uhrpUrl, additionalDays * 24 * 60)
    
    const record = this.files.get(uhrpUrl)
    if (record) {
      record.expiryDate = new Date(
        record.expiryDate.getTime() + additionalDays * 24 * 60 * 60 * 1000
      )
    }
  }
  
  getFilesByTag(tag: string): FileRecord[] {
    return Array.from(this.files.values())
      .filter(file => file.tags.includes(tag))
  }
  
  getExpiringFiles(daysAhead: number = 7): FileRecord[] {
    const cutoff = new Date(Date.now() + daysAhead * 24 * 60 * 60 * 1000)
    return Array.from(this.files.values())
      .filter(file => file.expiryDate <= cutoff)
      .sort((a, b) => a.expiryDate.getTime() - b.expiryDate.getTime())
  }
}
```

## Best Practices

1. **Always validate file integrity** after download using hash verification
2. **Implement proper retry logic** for network failures
3. **Use batch operations** for multiple files to improve performance
4. **Track file metadata** including expiration dates and renewal needs
5. **Implement proper error handling** for storage quota and payment issues

## Security Considerations

- **Encrypt sensitive files** before upload
- **Validate file types** and sizes before processing
- **Use authenticated storage endpoints** for sensitive data
- **Implement access controls** for file downloads

## Related

- [UHRP Storage Tutorial](../tutorials/uhrp-storage.md)
- [Security Best Practices](./security-best-practices.md)
