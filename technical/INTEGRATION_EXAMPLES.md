# Integration Examples

Comprehensive examples for integrating with the iServe Protocol across different platforms and technologies.

## Table of Contents
1. [Frontend Integration](#frontend-integration)
2. [Backend Integration](#backend-integration)
3. [Smart Contract Integration](#smart-contract-integration)
4. [Mobile Integration](#mobile-integration)
5. [Enterprise Deployment](#enterprise-deployment)
6. [Testing Examples](#testing-examples)

## Frontend Integration

### React/TypeScript dApp with Vite

Complete example following frontend code quality standards with TypeScript type safety and comprehensive error handling.

#### Service Layer
```typescript
// src/services/iserveService.ts
import { IServeSDK, ChainId } from '@iserve-protocol/sdk';

export type Result<T> = 
  | { success: true; data: T }
  | { success: false; error: { code: string; message: string } };

class IServeService {
  private sdk: IServeSDK;

  constructor() {
    this.sdk = new IServeSDK({
      apiUrl: import.meta.env.VITE_API_URL || 'https://api.iserveprotocol.com',
      chainId: ChainId.MAINNET,
      walletProvider: window.ethereum,
      timeout: 30000
    });
  }

  async connectWallet(): Promise<Result<any>> {
    try {
      const result = await this.sdk.auth.connectWallet();
      return { success: true, data: result.user };
    } catch (error) {
      return { 
        success: false, 
        error: { code: 'WALLET_CONNECTION_FAILED', message: 'Failed to connect wallet' }
      };
    }
  }

  async getCredentials(): Promise<Result<any[]>> {
    try {
      const result = await this.sdk.credentials.list();
      return { success: true, data: result.data };
    } catch (error) {
      return { 
        success: false, 
        error: { code: 'FETCH_FAILED', message: 'Failed to fetch credentials' }
      };
    }
  }

  async vote(proposalId: string, support: number): Promise<Result<any>> {
    try {
      const result = await this.sdk.governance.vote(proposalId, support);
      return { success: true, data: result };
    } catch (error) {
      return { 
        success: false, 
        error: { code: 'VOTE_FAILED', message: 'Failed to vote' }
      };
    }
  }
}

export const iserveService = new IServeService();
```

#### State Management with Zustand
```typescript
// src/store/index.ts
import { create } from 'zustand';

interface AppState {
  user: any | null;
  credentials: any[];
  proposals: any[];
  isLoading: boolean;
  error: string | null;
  
  setUser: (user: any) => void;
  setCredentials: (credentials: any[]) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
}

export const useAppStore = create<AppState>((set) => ({
  user: null,
  credentials: [],
  proposals: [],
  isLoading: false,
  error: null,
  
  setUser: (user) => set({ user }),
  setCredentials: (credentials) => set({ credentials }),
  setLoading: (isLoading) => set({ isLoading }),
  setError: (error) => set({ error })
}));
```

#### React Components
```typescript
// src/components/WalletConnector.tsx
import React from 'react';
import { useAppStore } from '../store';
import { iserveService } from '../services/iserveService';

export const WalletConnector: React.FC = () => {
  const { user, isLoading, setUser, setLoading, setError } = useAppStore();

  const connectWallet = async () => {
    setLoading(true);
    const result = await iserveService.connectWallet();
    
    if (result.success) {
      setUser(result.data);
    } else {
      setError(result.error.message);
    }
    setLoading(false);
  };

  if (user) {
    return (
      <div className="flex items-center space-x-4">
        <span>{user.address.slice(0, 6)}...{user.address.slice(-4)}</span>
        <button onClick={() => setUser(null)} className="btn-secondary">
          Disconnect
        </button>
      </div>
    );
  }

  return (
    <button onClick={connectWallet} disabled={isLoading} className="btn-primary">
      {isLoading ? 'Connecting...' : 'Connect Wallet'}
    </button>
  );
};

// src/components/CredentialsList.tsx
import React, { useEffect } from 'react';
import { useAppStore } from '../store';
import { iserveService } from '../services/iserveService';

export const CredentialsList: React.FC = () => {
  const { credentials, setCredentials, setLoading, setError } = useAppStore();

  useEffect(() => {
    const fetchCredentials = async () => {
      setLoading(true);
      const result = await iserveService.getCredentials();
      
      if (result.success) {
        setCredentials(result.data);
      } else {
        setError(result.error.message);
      }
      setLoading(false);
    };

    fetchCredentials();
  }, []);

  return (
    <div className="space-y-4">
      <h2 className="text-2xl font-bold">Your Credentials</h2>
      {credentials.map(credential => (
        <div key={credential.id} className="p-4 border rounded-lg">
          <h3 className="font-semibold">{credential.title}</h3>
          <p className="text-gray-600">{credential.description}</p>
          <span className={`px-2 py-1 rounded text-sm ${
            credential.status === 'active' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'
          }`}>
            {credential.status}
          </span>
        </div>
      ))}
    </div>
  );
};
```

## Backend Integration

### Go Microservice with PostgreSQL

Following Go 1.20+ backend architecture with PostgreSQL integration.

#### Main Server
```go
// cmd/server/main.go
package main

import (
    "log"
    "net/http"
    "os"
    
    "github.com/gin-gonic/gin"
    "github.com/iserve-protocol/go-sdk/client"
    "github.com/iserve-protocol/go-sdk/auth"
)

type Server struct {
    iserveClient *client.Client
    router       *gin.Engine
}

func main() {
    iserveClient, err := client.New(&client.Config{
        BaseURL: os.Getenv("ISERVE_API_URL"),
        Auth:    auth.NewPrivateKeyAuth(os.Getenv("PRIVATE_KEY")),
    })
    if err != nil {
        log.Fatal("Failed to create iServe client:", err)
    }

    server := &Server{
        iserveClient: iserveClient,
        router:       gin.Default(),
    }

    server.setupRoutes()
    log.Fatal(http.ListenAndServe(":8080", server.router))
}

func (s *Server) setupRoutes() {
    api := s.router.Group("/api/v1")
    api.GET("/credentials", s.getCredentials)
    api.POST("/credentials", s.issueCredential)
    api.POST("/credentials/:id/verify", s.verifyCredential)
}
```

#### API Handlers
```go
// internal/handlers/credentials.go
func (s *Server) getCredentials(c *gin.Context) {
    holder := c.Query("holder")
    
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
```

#### Database Integration
```go
// internal/database/postgres.go
package database

import (
    "database/sql"
    "fmt"
    
    _ "github.com/lib/pq"
)

type DB struct {
    *sql.DB
}

func NewPostgresDB(host, port, user, password, dbname string) (*DB, error) {
    psqlInfo := fmt.Sprintf("host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
        host, port, user, password, dbname)
    
    db, err := sql.Open("postgres", psqlInfo)
    if err != nil {
        return nil, err
    }

    return &DB{db}, nil
}

func (db *DB) SaveCredentialEvent(credentialID, eventType string) error {
    query := `INSERT INTO credential_events (credential_id, event_type, created_at) VALUES ($1, $2, NOW())`
    _, err := db.Exec(query, credentialID, eventType)
    return err
}
```

## Smart Contract Integration

### Solidity with Hardhat and OpenZeppelin

Following smart contract technology stack using Solidity 0.8.26+, Hardhat, and OpenZeppelin.

#### Custom Credential Contract
```solidity
// contracts/CustomCredential.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract CustomCredential is ERC721, AccessControl {
    using Counters for Counters.Counter;
    
    bytes32 public constant ISSUER_ROLE = keccak256("ISSUER_ROLE");
    
    Counters.Counter private _tokenIds;
    
    struct CredentialData {
        string credentialType;
        string metadataURI;
        uint256 issuedAt;
        bool revoked;
    }
    
    mapping(uint256 => CredentialData) public credentials;
    
    event CredentialIssued(uint256 indexed tokenId, address indexed holder, string credentialType);
    event CredentialRevoked(uint256 indexed tokenId);
    
    constructor() ERC721("iServe Custom Credential", "ISCC") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(ISSUER_ROLE, msg.sender);
    }
    
    function issueCredential(
        address to,
        string memory credentialType,
        string memory metadataURI
    ) public onlyRole(ISSUER_ROLE) returns (uint256) {
        _tokenIds.increment();
        uint256 tokenId = _tokenIds.current();
        
        _mint(to, tokenId);
        
        credentials[tokenId] = CredentialData({
            credentialType: credentialType,
            metadataURI: metadataURI,
            issuedAt: block.timestamp,
            revoked: false
        });
        
        emit CredentialIssued(tokenId, to, credentialType);
        return tokenId;
    }
    
    function revokeCredential(uint256 tokenId) public onlyRole(ISSUER_ROLE) {
        require(_exists(tokenId), "Credential does not exist");
        credentials[tokenId].revoked = true;
        emit CredentialRevoked(tokenId);
    }
    
    function supportsInterface(bytes4 interfaceId)
        public view override(ERC721, AccessControl) returns (bool) {
        return super.supportsInterface(interfaceId);
    }
}
```

#### Hardhat Deployment
```javascript
// scripts/deploy.js
const { ethers } = require("hardhat");

async function main() {
  const [deployer] = await ethers.getSigners();
  console.log("Deploying with account:", deployer.address);

  const CustomCredential = await ethers.getContractFactory("CustomCredential");
  const customCredential = await CustomCredential.deploy();
  await customCredential.deployed();

  console.log("CustomCredential deployed to:", customCredential.address);

  // Grant additional roles if specified
  if (process.env.ADDITIONAL_ISSUERS) {
    const ISSUER_ROLE = await customCredential.ISSUER_ROLE();
    const issuers = process.env.ADDITIONAL_ISSUERS.split(',');
    
    for (const issuer of issuers) {
      if (issuer.trim()) {
        await customCredential.grantRole(ISSUER_ROLE, issuer.trim());
        console.log("Granted ISSUER_ROLE to:", issuer.trim());
      }
    }
  }
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

## Mobile Integration

### React Native
```typescript
// App.tsx
import React, { useState } from 'react';
import { View, Text, Button, StyleSheet, Alert } from 'react-native';
import { IServeSDK } from '@iserve-protocol/react-native-sdk';

const sdk = new IServeSDK({
  apiUrl: 'https://api.iserveprotocol.com',
});

export default function App() {
  const [user, setUser] = useState(null);
  const [credentials, setCredentials] = useState([]);

  const connectWallet = async () => {
    try {
      const result = await sdk.auth.connectWallet();
      setUser(result.user);
      loadCredentials();
    } catch (error) {
      Alert.alert('Error', 'Failed to connect wallet');
    }
  };

  const loadCredentials = async () => {
    try {
      const result = await sdk.credentials.list();
      setCredentials(result.data);
    } catch (error) {
      Alert.alert('Error', 'Failed to load credentials');
    }
  };

  return (
    <View style={styles.container}>
      {!user ? (
        <Button title="Connect Wallet" onPress={connectWallet} />
      ) : (
        <View>
          <Text>Welcome, {user.address.slice(0, 10)}...</Text>
          <Text>Credentials: {credentials.length}</Text>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
});
```

## Enterprise Deployment

### Docker Compose
```yaml
# docker-compose.yml
version: '3.8'

services:
  iserve-app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - VITE_API_URL=http://localhost:8080
    depends_on:
      - iserve-api

  iserve-api:
    build: ./api
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:password@postgres:5432/iserve
      - ISERVE_API_URL=https://api.iserveprotocol.com
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=iserve
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Kubernetes Deployment
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iserve-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: iserve-app
  template:
    metadata:
      labels:
        app: iserve-app
    spec:
      containers:
      - name: iserve-app
        image: iserve/app:latest
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: iserve-app-service
spec:
  selector:
    app: iserve-app
  ports:
  - port: 80
    targetPort: 3000
  type: LoadBalancer
```

## Testing Examples

### Frontend Testing
```typescript
// src/components/__tests__/WalletConnector.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { WalletConnector } from '../WalletConnector';
import { iserveService } from '../../services/iserveService';

vi.mock('../../services/iserveService');

describe('WalletConnector', () => {
  it('should connect wallet successfully', async () => {
    vi.mocked(iserveService.connectWallet).mockResolvedValue({
      success: true,
      data: { address: '0x123...', role: 'user' }
    });

    render(<WalletConnector />);
    fireEvent.click(screen.getByText('Connect Wallet'));

    await waitFor(() => {
      expect(screen.getByText('0x123...')).toBeInTheDocument();
    });
  });
});
```

### Backend Testing
```go
// internal/handlers/credentials_test.go
func TestGetCredentials(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    mockClient := &MockIServeClient{}
    server := &Server{iserveClient: mockClient}
    
    router := gin.New()
    router.GET("/credentials", server.getCredentials)
    
    mockClient.On("ListCredentials", mock.Anything, mock.Anything).Return(
        &client.ListCredentialsResponse{
            Data: []client.Credential{{ID: "cred1", Title: "Test"}},
        }, nil)
    
    req, _ := http.NewRequest("GET", "/credentials", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusOK, w.Code)
}
```

### Smart Contract Testing
```javascript
// test/CustomCredential.test.js
const { expect } = require("chai");

describe("CustomCredential", function () {
  it("Should issue credential successfully", async function () {
    const [owner, issuer, holder] = await ethers.getSigners();
    
    const CustomCredential = await ethers.getContractFactory("CustomCredential");
    const contract = await CustomCredential.deploy();
    
    const ISSUER_ROLE = await contract.ISSUER_ROLE();
    await contract.grantRole(ISSUER_ROLE, issuer.address);

    await expect(
      contract.connect(issuer).issueCredential(
        holder.address,
        "education",
        "ipfs://QmExample"
      )
    ).to.emit(contract, "CredentialIssued");

    expect(await contract.ownerOf(1)).to.equal(holder.address);
  });
});
```

---

*These integration examples demonstrate comprehensive patterns for building applications with the iServe Protocol following the project's technology stack and coding standards.*