# SDK Documentation

Software Development Kits (SDKs) for integrating with the iServe Protocol across different programming languages and platforms.

## Table of Contents
1. [Overview](#overview)
2. [JavaScript/TypeScript SDK](#javascripttypescript-sdk)
3. [Python SDK](#python-sdk)
4. [Go SDK](#go-sdk)
5. [Mobile SDKs](#mobile-sdks)
6. [Integration Examples](#integration-examples)

## Overview

The iServe Protocol provides official SDKs for easy integration with the API services. All SDKs follow consistent patterns and provide:

- Type-safe API interactions
- Automatic error handling and retry logic
- Built-in authentication management
- WebSocket support for real-time updates
- Comprehensive documentation and examples

## JavaScript/TypeScript SDK

The primary SDK for frontend applications and Node.js backends.

### Installation
```bash
npm install @iserve-protocol/sdk
# or
yarn add @iserve-protocol/sdk
```

### Quick Start
```typescript
import { IServeSDK, ChainId } from '@iserve-protocol/sdk';

// Initialize SDK
const sdk = new IServeSDK({
  apiUrl: 'https://api.iserveprotocol.com',
  chainId: ChainId.MAINNET,
  walletProvider: window.ethereum // for browser environments
});

// Authenticate with wallet
await sdk.auth.connectWallet();

// Get user credentials
const credentials = await sdk.credentials.list();
console.log('User credentials:', credentials);
```

### Core Classes

#### IServeSDK
Main SDK class for initialization and configuration.

```typescript
interface IServeSDKConfig {
  apiUrl: string;
  chainId: ChainId;
  walletProvider?: any;
  privateKey?: string; // for server environments
  timeout?: number;
  retryAttempts?: number;
}

class IServeSDK {
  constructor(config: IServeSDKConfig);
  
  // Service instances
  auth: AuthService;
  credentials: CredentialService;
  governance: GovernanceService;
  users: UserService;
  websocket: WebSocketService;
}
```

#### AuthService
Handles authentication and wallet integration.

```typescript
class AuthService {
  // Wallet authentication
  async connectWallet(): Promise<AuthResult>;
  async disconnectWallet(): Promise<void>;
  async signMessage(message: string): Promise<string>;
  
  // JWT token management
  async refreshToken(): Promise<string>;
  getToken(): string | null;
  isAuthenticated(): boolean;
  
  // User info
  async getCurrentUser(): Promise<User>;
}

// Example usage
const authResult = await sdk.auth.connectWallet();
console.log('Authenticated user:', authResult.user);
```

#### CredentialService
Manages credential operations.

```typescript
class CredentialService {
  // List and filter credentials
  async list(filters?: CredentialFilters): Promise<PaginatedResult<Credential>>;
  async get(id: string): Promise<Credential>;
  
  // Issue new credentials (issuer only)
  async issue(request: IssueCredentialRequest): Promise<Credential>;
  async revoke(id: string, reason?: string): Promise<void>;
  
  // Verification
  async verify(id: string): Promise<VerificationResult>;
  async getVerificationHistory(id: string): Promise<VerificationHistory>;
}

// Example: Issue a credential
const credential = await sdk.credentials.issue({
  holder: '0x123...',
  credentialType: 'education',
  title: 'Computer Science Degree',
  description: 'Bachelor of Science in Computer Science',
  metadata: {
    institution: 'MIT',
    graduationDate: '2023-05-15',
    gpa: '3.8'
  },
  expiresAt: new Date('2026-05-15')
});
```

#### GovernanceService
Handles governance operations.

```typescript
class GovernanceService {
  // Proposals
  async listProposals(filters?: ProposalFilters): Promise<PaginatedResult<Proposal>>;
  async getProposal(id: string): Promise<Proposal>;
  async createProposal(request: CreateProposalRequest): Promise<Proposal>;
  
  // Voting
  async vote(proposalId: string, support: VoteSupport, reason?: string): Promise<VoteResult>;
  async getVotingPower(address?: string): Promise<VotingPowerInfo>;
  
  // Delegation
  async delegate(delegate: string): Promise<DelegationResult>;
  async getDelegationInfo(address?: string): Promise<DelegationInfo>;
}

// Example: Vote on a proposal
const voteResult = await sdk.governance.vote(
  'prop_123',
  VoteSupport.FOR,
  'I support this proposal because it improves governance'
);
```

#### WebSocketService
Real-time updates and subscriptions.

```typescript
class WebSocketService {
  async connect(): Promise<void>;
  async disconnect(): Promise<void>;
  
  // Subscriptions
  async subscribe(channel: string, filters?: any): Promise<Subscription>;
  async unsubscribe(subscriptionId: string): Promise<void>;
  
  // Event listeners
  on(event: string, callback: (data: any) => void): void;
  off(event: string, callback?: (data: any) => void): void;
}

// Example: Subscribe to proposal updates
await sdk.websocket.connect();

const subscription = await sdk.websocket.subscribe('proposals', {
  status: ['active', 'succeeded']
});

sdk.websocket.on('proposal_updated', (data) => {
  console.log('Proposal updated:', data);
});
```

### Error Handling

The SDK provides comprehensive error handling with typed error responses:

```typescript
import { IServeError, ErrorCode } from '@iserve-protocol/sdk';

try {
  const credential = await sdk.credentials.get('invalid_id');
} catch (error) {
  if (error instanceof IServeError) {
    switch (error.code) {
      case ErrorCode.NOT_FOUND:
        console.log('Credential not found');
        break;
      case ErrorCode.UNAUTHORIZED:
        console.log('Please authenticate first');
        break;
      case ErrorCode.RATE_LIMITED:
        console.log('Too many requests, please wait');
        break;
      default:
        console.log('Error:', error.message);
    }
  }
}
```

### React Hooks

For React applications, the SDK provides convenient hooks:

```typescript
import { useCredentials, useProposal, useWallet } from '@iserve-protocol/sdk/react';

function CredentialsList() {
  const { data: credentials, loading, error } = useCredentials();
  const { connect, disconnect, user } = useWallet();
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return (
    <div>
      {!user ? (
        <button onClick={connect}>Connect Wallet</button>
      ) : (
        <div>
          <p>Welcome, {user.address}</p>
          <button onClick={disconnect}>Disconnect</button>
          
          <h3>Your Credentials:</h3>
          {credentials?.map(cred => (
            <div key={cred.id}>
              <h4>{cred.title}</h4>
              <p>{cred.description}</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

## Python SDK

For Python backend applications and data analysis.

### Installation
```bash
pip install iserve-protocol-sdk
```

### Quick Start
```python
from iserve_sdk import IServeSDK
from iserve_sdk.auth import PrivateKeyAuth

# Initialize SDK with private key authentication
sdk = IServeSDK(
    api_url='https://api.iserveprotocol.com',
    auth=PrivateKeyAuth('your_private_key_here')
)

# Get credentials
credentials = await sdk.credentials.list()
print(f"Found {len(credentials.data)} credentials")
```

### Core Classes

```python
class IServeSDK:
    def __init__(self, api_url: str, auth: AuthProvider):
        self.credentials = CredentialService(self)
        self.governance = GovernanceService(self)
        self.users = UserService(self)

class CredentialService:
    async def list(self, filters: Optional[CredentialFilters] = None) -> PaginatedResult[Credential]:
        """List user credentials with optional filtering."""
        
    async def get(self, credential_id: str) -> Credential:
        """Get a specific credential by ID."""
        
    async def issue(self, request: IssueCredentialRequest) -> Credential:
        """Issue a new credential."""
        
    async def verify(self, credential_id: str) -> VerificationResult:
        """Verify a credential's authenticity."""
```

### Example: Bulk Credential Processing

```python
import asyncio
from iserve_sdk import IServeSDK
from iserve_sdk.auth import PrivateKeyAuth

async def process_credentials():
    sdk = IServeSDK(
        api_url='https://api.iserveprotocol.com',
        auth=PrivateKeyAuth(private_key)
    )
    
    # Get all credentials
    page = 1
    all_credentials = []
    
    while True:
        result = await sdk.credentials.list(page=page, limit=100)
        all_credentials.extend(result.data)
        
        if page >= result.pagination.pages:
            break
        page += 1
    
    # Verify all credentials
    verification_results = []
    for credential in all_credentials:
        try:
            result = await sdk.credentials.verify(credential.id)
            verification_results.append({
                'credential_id': credential.id,
                'status': result.status,
                'verified_at': result.verified_at
            })
        except Exception as e:
            print(f"Failed to verify {credential.id}: {e}")
    
    return verification_results

# Run the async function
results = asyncio.run(process_credentials())
print(f"Verified {len(results)} credentials")
```

## Go SDK

For Go backend services and microservices.

### Installation
```bash
go get github.com/iserve-protocol/go-sdk
```

### Quick Start
```go
package main

import (
    "context"
    "fmt"
    "log"
    
    "github.com/iserve-protocol/go-sdk/client"
    "github.com/iserve-protocol/go-sdk/auth"
)

func main() {
    // Create client with private key auth
    client, err := client.New(&client.Config{
        BaseURL: "https://api.iserveprotocol.com",
        Auth:    auth.NewPrivateKeyAuth(privateKey),
    })
    if err != nil {
        log.Fatal(err)
    }
    
    // Get credentials
    ctx := context.Background()
    credentials, err := client.Credentials.List(ctx, &client.ListCredentialsRequest{
        Page:  1,
        Limit: 20,
    })
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("Found %d credentials\n", len(credentials.Data))
}
```

### Service Interfaces

```go
// CredentialService interface
type CredentialService interface {
    List(ctx context.Context, req *ListCredentialsRequest) (*ListCredentialsResponse, error)
    Get(ctx context.Context, id string) (*Credential, error)
    Issue(ctx context.Context, req *IssueCredentialRequest) (*Credential, error)
    Revoke(ctx context.Context, id string, reason string) error
    Verify(ctx context.Context, id string) (*VerificationResult, error)
}

// GovernanceService interface
type GovernanceService interface {
    ListProposals(ctx context.Context, req *ListProposalsRequest) (*ListProposalsResponse, error)
    GetProposal(ctx context.Context, id string) (*Proposal, error)
    CreateProposal(ctx context.Context, req *CreateProposalRequest) (*Proposal, error)
    Vote(ctx context.Context, proposalID string, support VoteSupport, reason string) (*VoteResult, error)
}
```

### Example: Microservice Integration

```go
package main

import (
    "context"
    "encoding/json"
    "net/http"
    
    "github.com/gin-gonic/gin"
    "github.com/iserve-protocol/go-sdk/client"
)

type CredentialHandler struct {
    iserveClient *client.Client
}

func (h *CredentialHandler) GetCredentials(c *gin.Context) {
    userAddress := c.GetString("user_address") // from auth middleware
    
    credentials, err := h.iserveClient.Credentials.List(c, &client.ListCredentialsRequest{
        Holder: userAddress,
        Page:   1,
        Limit:  50,
    })
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, credentials)
}

func (h *CredentialHandler) IssueCredential(c *gin.Context) {
    var req client.IssueCredentialRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    credential, err := h.iserveClient.Credentials.Issue(c, &req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, credential)
}
```

## Mobile SDKs

### React Native SDK

```bash
npm install @iserve-protocol/react-native-sdk
```

```typescript
import { IServeSDK } from '@iserve-protocol/react-native-sdk';
import WalletConnectProvider from '@walletconnect/react-native-dapp';

const sdk = new IServeSDK({
  apiUrl: 'https://api.iserveprotocol.com',
  walletProvider: WalletConnectProvider,
  chainId: ChainId.MAINNET
});

// Use in React Native components
function CredentialsScreen() {
  const [credentials, setCredentials] = useState([]);
  
  useEffect(() => {
    async function loadCredentials() {
      try {
        const result = await sdk.credentials.list();
        setCredentials(result.data);
      } catch (error) {
        console.error('Failed to load credentials:', error);
      }
    }
    
    loadCredentials();
  }, []);
  
  return (
    <View>
      {credentials.map(cred => (
        <CredentialCard key={cred.id} credential={cred} />
      ))}
    </View>
  );
}
```

### iOS SDK (Swift)

```swift
import IServeSDK

let sdk = IServeSDK(
    apiURL: "https://api.iserveprotocol.com",
    chainID: .mainnet
)

// Authenticate with wallet
sdk.auth.connectWallet { result in
    switch result {
    case .success(let user):
        print("Authenticated: \(user.address)")
    case .failure(let error):
        print("Authentication failed: \(error)")
    }
}

// Get credentials
sdk.credentials.list { result in
    switch result {
    case .success(let credentials):
        print("Found \(credentials.data.count) credentials")
    case .failure(let error):
        print("Failed to load credentials: \(error)")
    }
}
```

### Android SDK (Kotlin)

```kotlin
import com.iserveprotocol.sdk.IServeSDK
import com.iserveprotocol.sdk.ChainId

val sdk = IServeSDK.Builder()
    .apiUrl("https://api.iserveprotocol.com")
    .chainId(ChainId.MAINNET)
    .build()

// Authenticate
sdk.auth.connectWallet { result ->
    when (result) {
        is Result.Success -> {
            println("Authenticated: ${result.data.user.address}")
        }
        is Result.Error -> {
            println("Authentication failed: ${result.error}")
        }
    }
}

// Get credentials
sdk.credentials.list { result ->
    when (result) {
        is Result.Success -> {
            println("Found ${result.data.data.size} credentials")
        }
        is Result.Error -> {
            println("Failed to load credentials: ${result.error}")
        }
    }
}
```

## Integration Examples

### Complete dApp Integration

```typescript
// Complete React dApp example
import React, { useState, useEffect } from 'react';
import { IServeSDK, ChainId } from '@iserve-protocol/sdk';

function App() {
  const [sdk] = useState(() => new IServeSDK({
    apiUrl: 'https://api.iserveprotocol.com',
    chainId: ChainId.MAINNET,
    walletProvider: window.ethereum
  }));
  
  const [user, setUser] = useState(null);
  const [credentials, setCredentials] = useState([]);
  const [proposals, setProposals] = useState([]);
  
  useEffect(() => {
    // Check if already authenticated
    if (sdk.auth.isAuthenticated()) {
      loadUserData();
    }
  }, []);
  
  async function connectWallet() {
    try {
      const result = await sdk.auth.connectWallet();
      setUser(result.user);
      await loadUserData();
    } catch (error) {
      console.error('Failed to connect wallet:', error);
    }
  }
  
  async function loadUserData() {
    try {
      const [credentialsResult, proposalsResult] = await Promise.all([
        sdk.credentials.list(),
        sdk.governance.listProposals({ status: 'active' })
      ]);
      
      setCredentials(credentialsResult.data);
      setProposals(proposalsResult.data);
    } catch (error) {
      console.error('Failed to load data:', error);
    }
  }
  
  async function voteOnProposal(proposalId: string, support: VoteSupport) {
    try {
      await sdk.governance.vote(proposalId, support);
      await loadUserData(); // Refresh data
    } catch (error) {
      console.error('Failed to vote:', error);
    }
  }
  
  return (
    <div>
      {!user ? (
        <button onClick={connectWallet}>Connect Wallet</button>
      ) : (
        <div>
          <h2>Welcome, {user.address}</h2>
          
          <section>
            <h3>Your Credentials ({credentials.length})</h3>
            {credentials.map(cred => (
              <div key={cred.id}>
                <h4>{cred.title}</h4>
                <p>{cred.description}</p>
                <span>Status: {cred.status}</span>
              </div>
            ))}
          </section>
          
          <section>
            <h3>Active Proposals ({proposals.length})</h3>
            {proposals.map(proposal => (
              <div key={proposal.id}>
                <h4>{proposal.title}</h4>
                <p>{proposal.description}</p>
                <button onClick={() => voteOnProposal(proposal.id, VoteSupport.FOR)}>
                  Vote For
                </button>
                <button onClick={() => voteOnProposal(proposal.id, VoteSupport.AGAINST)}>
                  Vote Against
                </button>
              </div>
            ))}
          </section>
        </div>
      )}
    </div>
  );
}
```

### Backend Service Integration

```go
// Complete Go microservice example
package main

import (
    "context"
    "log"
    "net/http"
    "time"
    
    "github.com/gin-gonic/gin"
    "github.com/iserve-protocol/go-sdk/client"
    "github.com/iserve-protocol/go-sdk/auth"
)

type Server struct {
    iserveClient *client.Client
    router       *gin.Engine
}

func NewServer(privateKey string) (*Server, error) {
    // Initialize iServe client
    iserveClient, err := client.New(&client.Config{
        BaseURL: "https://api.iserveprotocol.com",
        Auth:    auth.NewPrivateKeyAuth(privateKey),
        Timeout: 30 * time.Second,
    })
    if err != nil {
        return nil, err
    }
    
    s := &Server{
        iserveClient: iserveClient,
        router:       gin.Default(),
    }
    
    s.setupRoutes()
    return s, nil
}

func (s *Server) setupRoutes() {
    api := s.router.Group("/api/v1")
    
    // Credential endpoints
    api.GET("/credentials", s.getCredentials)
    api.POST("/credentials", s.issueCredential)
    api.POST("/credentials/:id/verify", s.verifyCredential)
    
    // Governance endpoints
    api.GET("/proposals", s.getProposals)
    api.POST("/proposals/:id/vote", s.voteOnProposal)
}

func (s *Server) getCredentials(c *gin.Context) {
    holder := c.Query("holder")
    if holder == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "holder parameter required"})
        return
    }
    
    credentials, err := s.iserveClient.Credentials.List(c, &client.ListCredentialsRequest{
        Holder: holder,
        Page:   1,
        Limit:  50,
    })
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, credentials)
}

func (s *Server) issueCredential(c *gin.Context) {
    var req client.IssueCredentialRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    credential, err := s.iserveClient.Credentials.Issue(c, &req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, credential)
}

func main() {
    server, err := NewServer("your_private_key_here")
    if err != nil {
        log.Fatal("Failed to create server:", err)
    }
    
    log.Println("Starting server on :8080")
    log.Fatal(http.ListenAndServe(":8080", server.router))
}
```

---

*SDK documentation is maintained alongside API releases and includes comprehensive examples for all supported platforms.*