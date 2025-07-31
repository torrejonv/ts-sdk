# React Integration Guide

This guide provides comprehensive steps for integrating the BSV TypeScript SDK with React applications, covering everything from basic setup to advanced patterns and production considerations.

## Prerequisites

Before starting, ensure you have:

- **Node.js** (version 16 or higher) installed from [nodejs.org](https://nodejs.org/)
- Basic knowledge of **JavaScript**, **React**, and **TypeScript**
- A code editor like **VS Code**
- **Git** for version control

## Project Setup

### Creating a New React Project

Begin by creating a new React project with TypeScript support:

```bash
# Create new React app with TypeScript template
npx create-react-app my-bsv-app --template typescript

# Navigate to project directory
cd my-bsv-app
```

### Installing the BSV SDK

Install the BSV TypeScript SDK and required dependencies:

```bash
# Install BSV SDK
npm install @bsv/sdk

# Install additional dependencies for React integration
npm install @types/node
```

### Environment Configuration

Create a `.env` file in your project root for configuration:

```env
# .env
REACT_APP_NETWORK=mainnet
REACT_APP_ARC_URL=https://api.taal.com/arc
REACT_APP_ARC_API_KEY=your_api_key_here
```

## Basic Transaction Component

### Creating the BSV Button Component

Create a new file `src/components/BsvButton.tsx`:

```typescript
import React, { useState } from 'react';
import { WalletClient, Script } from '@bsv/sdk';

interface BsvButtonProps {
  onTransactionComplete?: (txId: string) => void;
  onError?: (error: Error) => void;
}

const BsvButton: React.FC<BsvButtonProps> = ({ 
  onTransactionComplete, 
  onError 
}) => {
  const [isLoading, setIsLoading] = useState(false);
  const [lastTxId, setLastTxId] = useState<string | null>(null);

  const handleTransaction = async () => {
    setIsLoading(true);
    
    try {
      // Use WalletClient for React applications to avoid CORS issues
      // This connects to the local wallet and handles broadcasting properly
      const wallet = new WalletClient('auto', 'localhost');
      
      // Create transaction using WalletClient (recommended for React)
      const response = await wallet.createAction({
        description: 'React BSV transaction example',
        outputs: [
          {
            satoshis: 100, // Use 100 satoshis like in the working tutorial
            lockingScript: Script.fromASM(`OP_RETURN ${Buffer.from('Hello from React!').toString('hex')}`).toHex(),
            outputDescription: 'React integration test data'
          }
        ],
        // You can add options here if needed
        // options: { ... }
      });
      
      if (response.txid) {
        setLastTxId(response.txid);
        onTransactionComplete?.(response.txid);
      }
      
    } catch (error) {
      console.error('Transaction failed:', error);
      onError?.(error as Error);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="bsv-button-container">
      <button 
        className="bsv-transaction-button"
        onClick={handleTransaction}
        disabled={isLoading}
      >
        {isLoading ? 'Creating Transaction...' : 'Create BSV Transaction'}
      </button>
      
      {lastTxId && (
        <div className="bsv-status success">
          Transaction created! TXID: {lastTxId.substring(0, 16)}...
        </div>
      )}
    </div>
  );
};

export default BsvButton;
```

### Integrating into Your App

Update `src/App.tsx` to use the BSV component:

```typescript
import React from 'react';
import BsvButton from './components/BsvButton';
import './App.css';

function App() {
  const handleTransactionComplete = (txId: string) => {
    console.log('Transaction completed:', txId);
    alert(`Transaction successful! ID: ${txId}`);
  };

  const handleError = (error: Error) => {
    console.error('Transaction error:', error);
    alert(`Transaction failed: ${error.message}`);
  };

  return (
    <div className="App">
      <header className="App-header">
        <h1>BSV React Integration Demo</h1>
        <BsvButton 
          onTransactionComplete={handleTransactionComplete}
          onError={handleError}
        />
      </header>
    </div>
  );
}

export default App;
```

## Advanced Patterns

### React Context for BSV Operations

Create a context for managing BSV operations across your app:

```typescript
// src/contexts/BsvContext.tsx
import React, { createContext, useContext, useState, ReactNode } from 'react';
import { PrivateKey, ARC } from '@bsv/sdk';

interface BsvContextType {
  privateKey: PrivateKey | null;
  arc: ARC | null;
  isConnected: boolean;
  connect: () => Promise<void>;
  disconnect: () => void;
}

const BsvContext = createContext<BsvContextType | undefined>(undefined);

export const BsvProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [privateKey, setPrivateKey] = useState<PrivateKey | null>(null);
  const [arc, setArc] = useState<ARC | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  const connect = async () => {
    try {
      // In production, implement proper wallet connection
      const key = PrivateKey.fromRandom();
      const arcInstance = new ARC(
        process.env.REACT_APP_ARC_URL || 'https://api.taal.com/arc',
        process.env.REACT_APP_ARC_API_KEY
      );
      
      setPrivateKey(key);
      setArc(arcInstance);
      setIsConnected(true);
    } catch (error) {
      console.error('Failed to connect:', error);
      throw error;
    }
  };

  const disconnect = () => {
    setPrivateKey(null);
    setArc(null);
    setIsConnected(false);
  };

  return (
    <BsvContext.Provider value={{
      privateKey,
      arc,
      isConnected,
      connect,
      disconnect
    }}>
      {children}
    </BsvContext.Provider>
  );
};

export const useBsv = () => {
  const context = useContext(BsvContext);
  if (context === undefined) {
    throw new Error('useBsv must be used within a BsvProvider');
  }
  return context;
};
```

### Custom Hooks for BSV Operations

Create reusable hooks for common BSV operations:

```typescript
// src/hooks/useBsvTransaction.ts
import { useState, useCallback } from 'react';
import { Transaction, P2PKH, Script } from '@bsv/sdk';
import { useBsv } from '../contexts/BsvContext';

export const useBsvTransaction = () => {
  const { privateKey, arc } = useBsv();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const createTransaction = useCallback(async (
    outputs: Array<{ satoshis: number; address?: string; data?: string }>
  ) => {
    if (!privateKey || !arc) {
      throw new Error('BSV not connected');
    }

    setIsLoading(true);
    setError(null);

    try {
      const tx = new Transaction();

      // Add outputs
      outputs.forEach(output => {
        if (output.address) {
          // Send to address
          tx.addOutput({
            satoshis: output.satoshis,
            lockingScript: new P2PKH().lock(output.address)
          });
        } else if (output.data) {
          // Data output
          tx.addOutput({
            satoshis: 0,
            lockingScript: Script.fromASM(`OP_RETURN ${Buffer.from(output.data).toString('hex')}`)
          });
        }
      });

      await tx.fee();
      await tx.sign();
      
      const result = await tx.broadcast(arc);
      const txId = Buffer.from(tx.id()).toString('hex');
      
      return { txId, transaction: tx };
    } catch (err) {
      const error = err as Error;
      setError(error);
      throw error;
    } finally {
      setIsLoading(false);
    }
  }, [privateKey, arc]);

  return {
    createTransaction,
    isLoading,
    error
  };
};
```

## Wallet Integration

### BRC-100 Wallet Integration

For production applications, integrate with a BRC-100 compliant wallet (such as MetaNet wallet):

```typescript
// src/components/WalletConnector.tsx
import React, { useState, useEffect } from 'react';

interface WalletConnectorProps {
  onConnect: (wallet: any) => void;
  onDisconnect: () => void;
}

const WalletConnector: React.FC<WalletConnectorProps> = ({
  onConnect,
  onDisconnect
}) => {
  const [isConnected, setIsConnected] = useState(false);
  const [wallet, setWallet] = useState<any>(null);

  useEffect(() => {
    // Check if MetaNet wallet is available
    const checkWallet = async () => {
      if (typeof window !== 'undefined' && (window as any).metanet) {
        const metanetWallet = (window as any).metanet;
        setWallet(metanetWallet);
      }
    };

    checkWallet();
  }, []);

  const connectWallet = async () => {
    if (!wallet) {
      alert('MetaNet wallet not found. Please install MetaNet wallet extension.');
      return;
    }

    try {
      await wallet.connect();
      setIsConnected(true);
      onConnect(wallet);
    } catch (error) {
      console.error('Failed to connect wallet:', error);
    }
  };

  const disconnectWallet = () => {
    setIsConnected(false);
    setWallet(null);
    onDisconnect();
  };

  return (
    <div className="wallet-connector">
      {!isConnected ? (
        <button onClick={connectWallet} className="connect-button">
          Connect MetaNet Wallet
        </button>
      ) : (
        <div className="wallet-connected">
          <span>✅ Wallet Connected</span>
          <button onClick={disconnectWallet} className="disconnect-button">
            Disconnect
          </button>
        </div>
      )}
    </div>
  );
};

export default WalletConnector;
```

## Error Handling and Loading States

### Error Boundary Component

Create an error boundary for BSV operations:

```typescript
// src/components/BsvErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

class BsvErrorBoundary extends Component<Props, State> {
  public state: State = {
    hasError: false
  };

  public static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  public componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('BSV Error Boundary caught an error:', error, errorInfo);
  }

  public render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h2>Something went wrong with BSV operations</h2>
          <details style={{ whiteSpace: 'pre-wrap' }}>
            <summary>Error Details</summary>
            {this.state.error && this.state.error.toString()}
          </details>
          <button onClick={() => this.setState({ hasError: false, error: undefined })}>
            Try Again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

export default BsvErrorBoundary;
```

## Styling and UI Components

### CSS Styles

Add styles to `src/App.css`:

```css
/* BSV Component Styles */
.bsv-button-container {
  margin: 20px 0;
  text-align: center;
}

.bsv-transaction-button {
  background: linear-gradient(45deg, #f7931a, #ff6b35);
  border: none;
  border-radius: 8px;
  color: white;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  padding: 12px 24px;
  transition: all 0.3s ease;
}

.bsv-transaction-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(247, 147, 26, 0.3);
}

.bsv-transaction-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.transaction-result {
  margin-top: 15px;
  padding: 15px;
  background: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #f7931a;
}

.transaction-result a {
  color: #f7931a;
  text-decoration: none;
  font-weight: bold;
}

.wallet-connector {
  margin: 20px 0;
}

.connect-button {
  background: #28a745;
  border: none;
  border-radius: 6px;
  color: white;
  cursor: pointer;
  font-size: 14px;
  padding: 10px 20px;
}

.wallet-connected {
  display: flex;
  align-items: center;
  gap: 10px;
}

.disconnect-button {
  background: #dc3545;
  border: none;
  border-radius: 4px;
  color: white;
  cursor: pointer;
  font-size: 12px;
  padding: 6px 12px;
}

.error-boundary {
  background: #f8d7da;
  border: 1px solid #f5c6cb;
  border-radius: 8px;
  color: #721c24;
  margin: 20px 0;
  padding: 20px;
}
```

## Testing

### Unit Tests

Create tests for your BSV components:

```typescript
// src/components/__tests__/BsvButton.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import BsvButton from '../BsvButton';

// Mock the BSV SDK
jest.mock('@bsv/sdk', () => ({
  PrivateKey: {
    fromRandom: jest.fn(() => ({
      toPublicKey: jest.fn(() => ({
        toHash: jest.fn(() => 'mock-hash')
      }))
    }))
  },
  Transaction: jest.fn(() => ({
    addOutput: jest.fn(),
    fee: jest.fn(),
    sign: jest.fn(),
    broadcast: jest.fn(),
    id: jest.fn(() => Buffer.from('mock-tx-id'))
  })),
  P2PKH: jest.fn(() => ({
    lock: jest.fn(() => 'mock-locking-script')
  })),
  ARC: jest.fn()
}));

describe('BsvButton', () => {
  test('renders create transaction button', () => {
    render(<BsvButton />);
    expect(screen.getByText('Create BSV Transaction')).toBeInTheDocument();
  });

  test('shows loading state when creating transaction', async () => {
    render(<BsvButton />);
    
    fireEvent.click(screen.getByText('Create BSV Transaction'));
    
    expect(screen.getByText('Creating Transaction...')).toBeInTheDocument();
  });

  test('calls onTransactionComplete when transaction succeeds', async () => {
    const mockOnComplete = jest.fn();
    render(<BsvButton onTransactionComplete={mockOnComplete} />);
    
    fireEvent.click(screen.getByText('Create BSV Transaction'));
    
    await waitFor(() => {
      expect(mockOnComplete).toHaveBeenCalledWith('mock-tx-id');
    });
  });
});
```

### Deployment

```bash
# Build for production
npm run build

# Serve built files
npx serve -s build
```

## Running the Application

Start your development server:

```bash
npm start
```

Your React app with BSV integration will be available at `http://localhost:3000`.

## Troubleshooting

### Debug Mode

Enable debug logging by adding to your `.env`:

```env
REACT_APP_DEBUG=true
```

## Resources

- [BSV TypeScript SDK Documentation](https://docs.bsvblockchain.org/guides/sdks/ts)
- [React Documentation](https://reactjs.org/docs)
- [TypeScript Handbook](https://www.typescriptlang.org/docs)
- [BSV Academy](https://bsvacademy.net)
