# TypeScript Guidelines and Best Practices

This document outlines TypeScript development standards for the iServe Protocol frontend applications, emphasizing type safety, comprehensive error handling, and transaction status management.

## 📋 Table of Contents

1. [TypeScript Configuration](#typescript-configuration)
2. [Type Safety Standards](#type-safety-standards)
3. [Error Handling Patterns](#error-handling-patterns)
4. [Transaction Management](#transaction-management)
5. [Service Architecture](#service-architecture)
6. [Component Guidelines](#component-guidelines)
7. [Testing with TypeScript](#testing-with-typescript)
8. [Performance Optimization](#performance-optimization)

## TypeScript Configuration

### Strict Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["DOM", "DOM.Iterable", "ES6"],
    "allowJs": false,
    "skipLibCheck": true,
    "esModuleInterop": false,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": [
    "src"
  ],
  "exclude": [
    "node_modules",
    "dist"
  ]
}
```

### Path Mapping

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@/components/*": ["src/components/*"],
      "@/services/*": ["src/services/*"],
      "@/types/*": ["src/types/*"],
      "@/utils/*": ["src/utils/*"],
      "@/hooks/*": ["src/hooks/*"]
    }
  }
}
```

## Type Safety Standards

### Core Type Definitions

```typescript
// src/types/credential.ts
export interface CredentialData {
  readonly id: string;
  readonly holder: string;
  readonly issuer: string;
  readonly schema: string;
  readonly issuanceDate: Date;
  readonly expirationDate: Date | null;
  readonly revoked: boolean;
  readonly metadata: CredentialMetadata;
}

export interface CredentialMetadata {
  readonly name: string;
  readonly description: string;
  readonly image?: string;
  readonly attributes: ReadonlyArray<CredentialAttribute>;
}

export interface CredentialAttribute {
  readonly trait_type: string;
  readonly value: string | number | boolean;
  readonly display_type?: 'date' | 'number' | 'boost_percentage' | 'boost_number';
}

// Branded types for better type safety
export type Address = string & { readonly __brand: 'Address' };
export type TokenId = string & { readonly __brand: 'TokenId' };
export type TransactionHash = string & { readonly __brand: 'TransactionHash' };

// Type guards
export const isValidAddress = (address: string): address is Address => {
  return /^0x[a-fA-F0-9]{40}$/.test(address);
};

export const isValidTokenId = (tokenId: string): tokenId is TokenId => {
  return /^\d+$/.test(tokenId);
};
```

### API Response Types

```typescript
// src/types/api.ts
export interface ApiResponse<T> {
  readonly success: boolean;
  readonly data?: T;
  readonly error?: ApiError;
  readonly timestamp: string;
}

export interface ApiError {
  readonly code: string;
  readonly message: string;
  readonly details?: Record<string, unknown>;
}

export interface PaginatedResponse<T> {
  readonly items: ReadonlyArray<T>;
  readonly total: number;
  readonly page: number;
  readonly pageSize: number;
  readonly hasMore: boolean;
}

// Result type for error handling
export type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E };
```

### Web3 Types

```typescript
// src/types/web3.ts
export interface WalletState {
  readonly isConnected: boolean;
  readonly address: Address | null;
  readonly chainId: number | null;
  readonly provider: ethers.providers.Web3Provider | null;
}

export interface TransactionState {
  readonly status: 'idle' | 'pending' | 'confirming' | 'success' | 'error';
  readonly hash: TransactionHash | null;
  readonly error: string | null;
  readonly confirmations: number;
}

export interface ContractInteraction<T = unknown> {
  readonly method: string;
  readonly args: ReadonlyArray<unknown>;
  readonly expectedResult?: T;
}
```

## Error Handling Patterns

### Error Types

```typescript
// src/types/errors.ts
export abstract class AppError extends Error {
  abstract readonly code: string;
  abstract readonly userMessage: string;
  
  constructor(
    message: string,
    public readonly cause?: Error
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class ValidationError extends AppError {
  readonly code = 'VALIDATION_ERROR';
  readonly userMessage = 'Please check your input and try again.';
}

export class NetworkError extends AppError {
  readonly code = 'NETWORK_ERROR';
  readonly userMessage = 'Network connection failed. Please check your internet connection.';
}

export class WalletError extends AppError {
  readonly code = 'WALLET_ERROR';
  readonly userMessage = 'Wallet connection failed. Please check your wallet and try again.';
}

export class TransactionError extends AppError {
  readonly code = 'TRANSACTION_ERROR';
  readonly userMessage = 'Transaction failed. Please try again.';
  
  constructor(
    message: string,
    public readonly transactionHash?: TransactionHash,
    cause?: Error
  ) {
    super(message, cause);
  }
}
```

### Error Boundary Component

```typescript
// src/components/ErrorBoundary.tsx
import React, { ErrorInfo, ReactNode } from 'react';
import { AppError } from '@/types/errors';

interface Props {
  readonly children: ReactNode;
  readonly fallback?: (error: Error) => ReactNode;
}

interface State {
  readonly hasError: boolean;
  readonly error: Error | null;
}

export class ErrorBoundary extends React.Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    console.error('Error caught by boundary:', error, errorInfo);
    
    // Log to error reporting service
    this.logError(error, errorInfo);
  }

  private logError(error: Error, errorInfo: ErrorInfo): void {
    // Implement error logging service integration
    if (error instanceof AppError) {
      console.error(`[${error.code}] ${error.message}`, {
        userMessage: error.userMessage,
        cause: error.cause,
        errorInfo
      });
    }
  }

  render(): ReactNode {
    if (this.state.hasError && this.state.error) {
      if (this.props.fallback) {
        return this.props.fallback(this.state.error);
      }

      const error = this.state.error;
      const userMessage = error instanceof AppError 
        ? error.userMessage 
        : 'An unexpected error occurred.';

      return (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <p>{userMessage}</p>
          <button onClick={() => window.location.reload()}>
            Reload Page
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Error Handling Hook

```typescript
// src/hooks/useErrorHandler.ts
import { useCallback } from 'react';
import { useNotification } from './useNotification';
import { AppError, NetworkError, WalletError, TransactionError } from '@/types/errors';

interface ErrorHandler {
  readonly handleError: (error: unknown) => void;
  readonly handleAsyncError: <T>(promise: Promise<T>) => Promise<T | null>;
}

export const useErrorHandler = (): ErrorHandler => {
  const { showError } = useNotification();

  const handleError = useCallback((error: unknown): void => {
    console.error('Error handled:', error);

    if (error instanceof AppError) {
      showError(error.userMessage);
      return;
    }

    if (error instanceof Error) {
      // Map known error patterns to user-friendly messages
      if (error.message.includes('user rejected')) {
        showError('Transaction was cancelled by user.');
        return;
      }

      if (error.message.includes('insufficient funds')) {
        showError('Insufficient funds for this transaction.');
        return;
      }

      if (error.message.includes('network')) {
        showError('Network error. Please check your connection.');
        return;
      }
    }

    // Fallback for unknown errors
    showError('An unexpected error occurred. Please try again.');
  }, [showError]);

  const handleAsyncError = useCallback(async <T>(
    promise: Promise<T>
  ): Promise<T | null> => {
    try {
      return await promise;
    } catch (error) {
      handleError(error);
      return null;
    }
  }, [handleError]);

  return { handleError, handleAsyncError };
};
```

## Transaction Management

### Transaction Hook

```typescript
// src/hooks/useTransaction.ts
import { useState, useCallback } from 'react';
import { ethers } from 'ethers';
import { TransactionState, TransactionHash } from '@/types/web3';
import { TransactionError } from '@/types/errors';
import { useErrorHandler } from './useErrorHandler';

interface TransactionOptions {
  readonly onSuccess?: (hash: TransactionHash) => void;
  readonly onError?: (error: TransactionError) => void;
  readonly confirmations?: number;
}

interface TransactionHook {
  readonly state: TransactionState;
  readonly execute: <T>(
    contractCall: () => Promise<ethers.ContractTransaction>,
    options?: TransactionOptions
  ) => Promise<T | null>;
  readonly reset: () => void;
}

export const useTransaction = (): TransactionHook => {
  const [state, setState] = useState<TransactionState>({
    status: 'idle',
    hash: null,
    error: null,
    confirmations: 0
  });

  const { handleError } = useErrorHandler();

  const execute = useCallback(async <T>(
    contractCall: () => Promise<ethers.ContractTransaction>,
    options: TransactionOptions = {}
  ): Promise<T | null> => {
    const { onSuccess, onError, confirmations = 1 } = options;

    try {
      setState(prev => ({ ...prev, status: 'pending', error: null }));

      const tx = await contractCall();
      const hash = tx.hash as TransactionHash;

      setState(prev => ({ 
        ...prev, 
        status: 'confirming',
        hash,
        confirmations: 0
      }));

      // Wait for confirmations
      const receipt = await tx.wait(confirmations);

      setState(prev => ({ 
        ...prev, 
        status: 'success',
        confirmations: receipt.confirmations
      }));

      onSuccess?.(hash);
      return receipt as T;

    } catch (error) {
      const txError = new TransactionError(
        error instanceof Error ? error.message : 'Transaction failed',
        state.hash ?? undefined,
        error instanceof Error ? error : undefined
      );

      setState(prev => ({ 
        ...prev, 
        status: 'error',
        error: txError.userMessage
      }));

      onError?.(txError);
      handleError(txError);
      return null;
    }
  }, [handleError, state.hash]);

  const reset = useCallback(() => {
    setState({
      status: 'idle',
      hash: null,
      error: null,
      confirmations: 0
    });
  }, []);

  return { state, execute, reset };
};
```

### Transaction Status Component

```typescript
// src/components/TransactionStatus.tsx
import React from 'react';
import { TransactionState, TransactionHash } from '@/types/web3';

interface Props {
  readonly state: TransactionState;
  readonly onRetry?: () => void;
  readonly className?: string;
}

export const TransactionStatus: React.FC<Props> = ({ 
  state, 
  onRetry, 
  className = '' 
}) => {
  const getStatusIcon = (): string => {
    switch (state.status) {
      case 'pending':
        return '⏳';
      case 'confirming':
        return '🔄';
      case 'success':
        return '✅';
      case 'error':
        return '❌';
      default:
        return '';
    }
  };

  const getStatusMessage = (): string => {
    switch (state.status) {
      case 'pending':
        return 'Confirm transaction in your wallet...';
      case 'confirming':
        return `Confirming transaction... (${state.confirmations} confirmations)`;
      case 'success':
        return 'Transaction successful!';
      case 'error':
        return state.error || 'Transaction failed';
      default:
        return '';
    }
  };

  const getExplorerLink = (hash: TransactionHash): string => {
    // Adjust for different networks
    return `https://etherscan.io/tx/${hash}`;
  };

  if (state.status === 'idle') {
    return null;
  }

  return (
    <div className={`transaction-status ${className}`}>
      <div className="status-content">
        <span className="status-icon">{getStatusIcon()}</span>
        <span className="status-message">{getStatusMessage()}</span>
      </div>

      {state.hash && (
        <a
          href={getExplorerLink(state.hash)}
          target="_blank"
          rel="noopener noreferrer"
          className="explorer-link"
        >
          View on Etherscan ↗
        </a>
      )}

      {state.status === 'error' && onRetry && (
        <button onClick={onRetry} className="retry-button">
          Try Again
        </button>
      )}
    </div>
  );
};
```

## Service Architecture

### Base Service Class

```typescript
// src/services/BaseService.ts
import { ApiResponse, Result } from '@/types/api';
import { NetworkError, ValidationError } from '@/types/errors';

export abstract class BaseService {
  protected readonly baseURL: string;

  constructor(baseURL: string) {
    this.baseURL = baseURL;
  }

  protected async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<Result<T, NetworkError>> {
    try {
      const response = await fetch(`${this.baseURL}${endpoint}`, {
        headers: {
          'Content-Type': 'application/json',
          ...options.headers,
        },
        ...options,
      });

      if (!response.ok) {
        throw new NetworkError(
          `HTTP ${response.status}: ${response.statusText}`
        );
      }

      const data: ApiResponse<T> = await response.json();

      if (!data.success) {
        throw new NetworkError(
          data.error?.message || 'API request failed'
        );
      }

      if (!data.data) {
        throw new NetworkError('No data received from API');
      }

      return { success: true, data: data.data };
    } catch (error) {
      if (error instanceof NetworkError) {
        return { success: false, error };
      }

      return {
        success: false,
        error: new NetworkError(
          error instanceof Error ? error.message : 'Network request failed'
        )
      };
    }
  }

  protected validateInput<T>(
    data: unknown,
    validator: (data: unknown) => data is T
  ): Result<T, ValidationError> {
    if (validator(data)) {
      return { success: true, data };
    }

    return {
      success: false,
      error: new ValidationError('Invalid input data')
    };
  }
}
```

### Credential Service

```typescript
// src/services/CredentialService.ts
import { BaseService } from './BaseService';
import { CredentialData, Address, TokenId } from '@/types/credential';
import { PaginatedResponse, Result } from '@/types/api';
import { ValidationError } from '@/types/errors';
import { isValidAddress, isValidTokenId } from '@/types/credential';

export class CredentialService extends BaseService {
  constructor() {
    super(process.env.REACT_APP_API_URL || 'http://localhost:8080');
  }

  async getCredential(tokenId: TokenId): Promise<Result<CredentialData, Error>> {
    if (!isValidTokenId(tokenId)) {
      return {
        success: false,
        error: new ValidationError('Invalid token ID format')
      };
    }

    return this.request<CredentialData>(`/api/v1/credentials/${tokenId}`);
  }

  async getCredentialsByHolder(
    address: Address,
    page = 1,
    pageSize = 20
  ): Promise<Result<PaginatedResponse<CredentialData>, Error>> {
    if (!isValidAddress(address)) {
      return {
        success: false,
        error: new ValidationError('Invalid address format')
      };
    }

    return this.request<PaginatedResponse<CredentialData>>(
      `/api/v1/credentials/holder/${address}?page=${page}&pageSize=${pageSize}`
    );
  }

  async getCredentialsByIssuer(
    address: Address,
    page = 1,
    pageSize = 20
  ): Promise<Result<PaginatedResponse<CredentialData>, Error>> {
    if (!isValidAddress(address)) {
      return {
        success: false,
        error: new ValidationError('Invalid address format')
      };
    }

    return this.request<PaginatedResponse<CredentialData>>(
      `/api/v1/credentials/issuer/${address}?page=${page}&pageSize=${pageSize}`
    );
  }
}

// Service instance
export const credentialService = new CredentialService();
```

## Component Guidelines

### Component Props Interface

```typescript
// src/components/CredentialCard.tsx
import React from 'react';
import { CredentialData } from '@/types/credential';

interface Props {
  readonly credential: CredentialData;
  readonly showDetails?: boolean;
  readonly onSelect?: (credential: CredentialData) => void;
  readonly className?: string;
}

export const CredentialCard: React.FC<Props> = ({
  credential,
  showDetails = false,
  onSelect,
  className = ''
}) => {
  const handleClick = React.useCallback(() => {
    onSelect?.(credential);
  }, [credential, onSelect]);

  const formatDate = React.useCallback((date: Date): string => {
    return new Intl.DateTimeFormat('en-US', {
      year: 'numeric',
      month: 'short',
      day: 'numeric'
    }).format(date);
  }, []);

  return (
    <div 
      className={`credential-card ${className}`}
      onClick={handleClick}
      role={onSelect ? 'button' : undefined}
      tabIndex={onSelect ? 0 : undefined}
    >
      <div className="credential-header">
        <h3>{credential.metadata.name}</h3>
        {credential.revoked && (
          <span className="revoked-badge">Revoked</span>
        )}
      </div>

      {credential.metadata.image && (
        <img 
          src={credential.metadata.image} 
          alt={credential.metadata.name}
          className="credential-image"
        />
      )}

      <div className="credential-info">
        <p>{credential.metadata.description}</p>
        <div className="credential-dates">
          <span>Issued: {formatDate(credential.issuanceDate)}</span>
          {credential.expirationDate && (
            <span>Expires: {formatDate(credential.expirationDate)}</span>
          )}
        </div>
      </div>

      {showDetails && (
        <div className="credential-details">
          <div className="attributes">
            {credential.metadata.attributes.map((attr, index) => (
              <div key={index} className="attribute">
                <span className="trait-type">{attr.trait_type}:</span>
                <span className="value">{attr.value}</span>
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
};
```

### Reusable Hook Pattern

```typescript
// src/hooks/useCredentials.ts
import { useState, useEffect, useCallback } from 'react';
import { CredentialData, Address } from '@/types/credential';
import { credentialService } from '@/services/CredentialService';
import { useErrorHandler } from './useErrorHandler';

interface UseCredentialsResult {
  readonly credentials: ReadonlyArray<CredentialData>;
  readonly loading: boolean;
  readonly error: string | null;
  readonly refetch: () => Promise<void>;
  readonly hasMore: boolean;
  readonly loadMore: () => Promise<void>;
}

export const useCredentials = (
  holderAddress: Address | null
): UseCredentialsResult => {
  const [credentials, setCredentials] = useState<ReadonlyArray<CredentialData>>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);

  const { handleAsyncError } = useErrorHandler();

  const fetchCredentials = useCallback(async (pageNum: number, reset = false) => {
    if (!holderAddress || loading) return;

    setLoading(true);
    setError(null);

    const result = await handleAsyncError(
      credentialService.getCredentialsByHolder(holderAddress, pageNum)
    );

    if (result?.success) {
      const { items, hasMore: more } = result.data;
      
      setCredentials(prev => 
        reset ? items : [...prev, ...items]
      );
      setHasMore(more);
    } else if (result) {
      setError('Failed to load credentials');
    }

    setLoading(false);
  }, [holderAddress, loading, handleAsyncError]);

  const refetch = useCallback(async () => {
    setPage(1);
    await fetchCredentials(1, true);
  }, [fetchCredentials]);

  const loadMore = useCallback(async () => {
    if (hasMore && !loading) {
      const nextPage = page + 1;
      setPage(nextPage);
      await fetchCredentials(nextPage);
    }
  }, [hasMore, loading, page, fetchCredentials]);

  useEffect(() => {
    if (holderAddress) {
      refetch();
    } else {
      setCredentials([]);
      setError(null);
    }
  }, [holderAddress, refetch]);

  return {
    credentials,
    loading,
    error,
    refetch,
    hasMore,
    loadMore
  };
};
```

## Testing with TypeScript

### Component Testing

```typescript
// src/components/__tests__/CredentialCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { CredentialCard } from '../CredentialCard';
import { CredentialData } from '@/types/credential';

const mockCredential: CredentialData = {
  id: '1',
  holder: '0x1234567890123456789012345678901234567890' as Address,
  issuer: '0x0987654321098765432109876543210987654321' as Address,
  schema: 'university-degree',
  issuanceDate: new Date('2023-01-01'),
  expirationDate: null,
  revoked: false,
  metadata: {
    name: 'Bachelor of Science',
    description: 'Computer Science Degree',
    attributes: [
      { trait_type: 'University', value: 'MIT' },
      { trait_type: 'Major', value: 'Computer Science' }
    ]
  }
};

describe('CredentialCard', () => {
  it('should render credential information correctly', () => {
    render(<CredentialCard credential={mockCredential} />);

    expect(screen.getByText('Bachelor of Science')).toBeInTheDocument();
    expect(screen.getByText('Computer Science Degree')).toBeInTheDocument();
    expect(screen.getByText(/Issued: Jan 1, 2023/)).toBeInTheDocument();
  });

  it('should call onSelect when clicked', () => {
    const onSelect = jest.fn();
    render(<CredentialCard credential={mockCredential} onSelect={onSelect} />);

    fireEvent.click(screen.getByRole('button'));

    expect(onSelect).toHaveBeenCalledWith(mockCredential);
  });

  it('should show revoked badge when credential is revoked', () => {
    const revokedCredential = { ...mockCredential, revoked: true };
    render(<CredentialCard credential={revokedCredential} />);

    expect(screen.getByText('Revoked')).toBeInTheDocument();
  });

  it('should show attributes when showDetails is true', () => {
    render(<CredentialCard credential={mockCredential} showDetails />);

    expect(screen.getByText('University:')).toBeInTheDocument();
    expect(screen.getByText('MIT')).toBeInTheDocument();
    expect(screen.getByText('Major:')).toBeInTheDocument();
    expect(screen.getByText('Computer Science')).toBeInTheDocument();
  });
});
```

### Service Testing

```typescript
// src/services/__tests__/CredentialService.test.ts
import { credentialService } from '../CredentialService';
import { Address, TokenId } from '@/types/credential';

// Mock fetch
global.fetch = jest.fn();

describe('CredentialService', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('getCredential', () => {
    it('should return credential data for valid token ID', async () => {
      const mockCredential = { id: '1', holder: '0x123...' };
      (fetch as jest.Mock).mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          success: true,
          data: mockCredential
        })
      });

      const result = await credentialService.getCredential('1' as TokenId);

      expect(result.success).toBe(true);
      if (result.success) {
        expect(result.data).toEqual(mockCredential);
      }
    });

    it('should return validation error for invalid token ID', async () => {
      const result = await credentialService.getCredential('invalid' as TokenId);

      expect(result.success).toBe(false);
      if (!result.success) {
        expect(result.error.code).toBe('VALIDATION_ERROR');
      }
    });
  });
});
```

## Performance Optimization

### Memoization Patterns

```typescript
// src/hooks/useMemoizedCredentials.ts
import { useMemo } from 'react';
import { CredentialData } from '@/types/credential';

interface UseMemoizedCredentialsProps {
  readonly credentials: ReadonlyArray<CredentialData>;
  readonly searchTerm: string;
  readonly sortBy: 'date' | 'name' | 'issuer';
  readonly showRevoked: boolean;
}

export const useMemoizedCredentials = ({
  credentials,
  searchTerm,
  sortBy,
  showRevoked
}: UseMemoizedCredentialsProps) => {
  return useMemo(() => {
    let filtered = credentials;

    // Filter by search term
    if (searchTerm) {
      filtered = filtered.filter(credential =>
        credential.metadata.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
        credential.metadata.description.toLowerCase().includes(searchTerm.toLowerCase())
      );
    }

    // Filter revoked credentials
    if (!showRevoked) {
      filtered = filtered.filter(credential => !credential.revoked);
    }

    // Sort credentials
    const sorted = [...filtered].sort((a, b) => {
      switch (sortBy) {
        case 'date':
          return b.issuanceDate.getTime() - a.issuanceDate.getTime();
        case 'name':
          return a.metadata.name.localeCompare(b.metadata.name);
        case 'issuer':
          return a.issuer.localeCompare(b.issuer);
        default:
          return 0;
      }
    });

    return sorted;
  }, [credentials, searchTerm, sortBy, showRevoked]);
};
```

### Code Splitting

```typescript
// src/pages/CredentialPage.tsx
import React, { Suspense } from 'react';
import { ErrorBoundary } from '@/components/ErrorBoundary';
import { LoadingSpinner } from '@/components/LoadingSpinner';

// Lazy load heavy components
const CredentialList = React.lazy(() => import('@/components/CredentialList'));
const CredentialDetails = React.lazy(() => import('@/components/CredentialDetails'));

export const CredentialPage: React.FC = () => {
  return (
    <ErrorBoundary>
      <div className="credential-page">
        <Suspense fallback={<LoadingSpinner />}>
          <CredentialList />
        </Suspense>
        
        <Suspense fallback={<LoadingSpinner />}>
          <CredentialDetails />
        </Suspense>
      </div>
    </ErrorBoundary>
  );
};
```

---

*These TypeScript guidelines ensure type safety, comprehensive error handling, and proper transaction status management throughout the iServe Protocol frontend applications.*