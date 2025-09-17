# API Specifications

This document provides comprehensive specifications for all iServe Protocol APIs, including REST endpoints, WebSocket connections, authentication, and integration patterns.

## Table of Contents
1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Core API](#core-api)
4. [Governance API](#governance-api)
5. [Security API](#security-api)
6. [WebSocket API](#websocket-api)
7. [Error Handling](#error-handling)
8. [Rate Limiting](#rate-limiting)
9. [API Versioning](#api-versioning)

## Overview

The iServe Protocol provides multiple API services built with Go 1.20+ and PostgreSQL:

- **Core API** (`iserve-api`): Credential management, verification, and user operations
- **Governance API** (`iserve-governance-`): Proposal management, voting, and governance operations
- **Security API**: Authentication, validation, monitoring, and access control

### Base URLs
- **Production**: `https://api.iserveprotocol.com`
- **Staging**: `https://api-staging.iserveprotocol.com`
- **Local Development**: `http://localhost:8080`

### API Versioning
All APIs use URL-based versioning with the format `/api/v1/`

## Authentication

### Overview
The iServe Protocol uses JWT-based authentication with Ethereum wallet integration for Web3 compatibility.

### Authentication Flow

#### 1. Wallet Authentication
```http
POST /api/v1/auth/wallet
Content-Type: application/json

{
  "address": "0x742d35Cc...",
  "signature": "0x1234...",
  "message": "Sign this message to authenticate: {timestamp}"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_at": "2024-01-01T12:00:00Z",
    "user": {
      "address": "0x742d35Cc...",
      "role": "user",
      "permissions": ["read", "vote"]
    }
  }
}
```

#### 2. JWT Token Usage
Include the JWT token in the `Authorization` header:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Roles and Permissions

| Role | Permissions | Description |
|------|-------------|-------------|
| `user` | `read`, `vote`, `create_proposal` | Basic token holder |
| `delegate` | `user` + `delegate_vote` | Governance delegate |
| `admin` | All permissions | System administrator |
| `auditor` | `read`, `audit` | Security auditor |

### Session Management
Sessions are automatically managed with configurable timeouts and refresh mechanisms.

## Core API

The Core API handles credential management, verification, and user operations.

### Credentials

#### List Credentials
```http
GET /api/v1/credentials
Authorization: Bearer {token}
```

**Query Parameters:**
- `page` (int): Page number (default: 1)
- `limit` (int): Items per page (default: 20, max: 100)
- `status` (string): Filter by status (`active`, `revoked`, `expired`)
- `issuer` (string): Filter by issuer address
- `holder` (string): Filter by holder address

**Response:**
```json
{
  "success": true,
  "data": {
    "credentials": [
      {
        "id": "cred_123",
        "token_id": "456",
        "issuer": "0x742d35Cc...",
        "holder": "0x123abc...",
        "credential_type": "education",
        "title": "Computer Science Degree",
        "description": "Bachelor of Science in Computer Science",
        "metadata": {
          "institution": "MIT",
          "graduation_date": "2023-05-15",
          "gpa": "3.8"
        },
        "status": "active",
        "created_at": "2023-05-15T10:00:00Z",
        "expires_at": "2026-05-15T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "pages": 8
    }
  }
}
```

#### Get Credential
```http
GET /api/v1/credentials/{id}
Authorization: Bearer {token}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "cred_123",
    "token_id": "456",
    "issuer": "0x742d35Cc...",
    "holder": "0x123abc...",
    "credential_type": "education",
    "title": "Computer Science Degree",
    "description": "Bachelor of Science in Computer Science",
    "metadata": {
      "institution": "MIT",
      "graduation_date": "2023-05-15",
      "gpa": "3.8"
    },
    "status": "active",
    "blockchain_data": {
      "contract_address": "0xCredentialContract...",
      "transaction_hash": "0x123transaction...",
      "block_number": 18500000
    },
    "verification_history": [
      {
        "verifier": "0xVerifier...",
        "timestamp": "2023-06-01T14:30:00Z",
        "result": "valid"
      }
    ],
    "created_at": "2023-05-15T10:00:00Z",
    "updated_at": "2023-06-01T14:30:00Z",
    "expires_at": "2026-05-15T10:00:00Z"
  }
}
```

#### Issue Credential
```http
POST /api/v1/credentials
Authorization: Bearer {token}
Content-Type: application/json

{
  "holder": "0x123abc...",
  "credential_type": "education",
  "title": "Computer Science Degree",
  "description": "Bachelor of Science in Computer Science",
  "metadata": {
    "institution": "MIT",
    "graduation_date": "2023-05-15",
    "gpa": "3.8"
  },
  "expires_at": "2026-05-15T10:00:00Z"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "cred_123",
    "token_id": "456",
    "transaction_hash": "0x123transaction...",
    "status": "pending"
  }
}
```

#### Revoke Credential
```http
DELETE /api/v1/credentials/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "reason": "Credential no longer valid"
}
```

### Verification

#### Verify Credential
```http
POST /api/v1/verify/{credential_id}
Authorization: Bearer {token}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "credential_id": "cred_123",
    "status": "valid",
    "verification_result": {
      "signature_valid": true,
      "not_expired": true,
      "not_revoked": true,
      "issuer_trusted": true
    },
    "verified_at": "2023-07-01T09:15:00Z",
    "verifier": "0xVerifier..."
  }
}
```

#### Verification History
```http
GET /api/v1/verify/history/{credential_id}
Authorization: Bearer {token}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "credential_id": "cred_123",
    "verifications": [
      {
        "id": "ver_789",
        "verifier": "0xVerifier...",
        "result": "valid",
        "timestamp": "2023-07-01T09:15:00Z",
        "details": {
          "signature_valid": true,
          "not_expired": true,
          "not_revoked": true,
          "issuer_trusted": true
        }
      }
    ],
    "total_verifications": 15,
    "last_verified": "2023-07-01T09:15:00Z"
  }
}
```

### Users

#### Register User
```http
POST /api/v1/users/register
Content-Type: application/json

{
  "address": "0x123abc...",
  "username": "alice_doe",
  "email": "alice@example.com",
  "profile": {
    "name": "Alice Doe",
    "bio": "Blockchain developer",
    "website": "https://alice.dev"
  }
}
```

#### Get User Profile
```http
GET /api/v1/users/{address}
Authorization: Bearer {token}
```

#### Update User Profile
```http
PUT /api/v1/users/{address}
Authorization: Bearer {token}
Content-Type: application/json

{
  "username": "alice_smith",
  "profile": {
    "name": "Alice Smith",
    "bio": "Senior blockchain developer"
  }
}
```

## Governance API

The Governance API handles proposal management, voting, and delegation.

### Proposals

#### List Proposals
```http
GET /api/v1/governance/proposals
Authorization: Bearer {token}
```

**Query Parameters:**
- `status` (string): `pending`, `active`, `succeeded`, `defeated`, `canceled`
- `proposer` (string): Filter by proposer address
- `page`, `limit`: Pagination

**Response:**
```json
{
  "success": true,
  "data": {
    "proposals": [
      {
        "id": "prop_123",
        "proposal_id": "1",
        "title": "Increase Quorum Threshold",
        "description": "Proposal to increase governance quorum from 4% to 6%",
        "proposer": "0xProposer...",
        "status": "active",
        "voting": {
          "start_block": 18500000,
          "end_block": 18550000,
          "for_votes": "15000000000000000000000000",
          "against_votes": "5000000000000000000000000",
          "abstain_votes": "2000000000000000000000000"
        },
        "actions": [
          {
            "target": "0xGovernanceParams...",
            "value": "0",
            "signature": "updateQuorum(uint256)",
            "calldata": "0x..."
          }
        ],
        "created_at": "2023-07-01T10:00:00Z",
        "voting_starts_at": "2023-07-02T10:00:00Z",
        "voting_ends_at": "2023-07-09T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 45,
      "pages": 3
    }
  }
}
```

#### Get Proposal
```http
GET /api/v1/governance/proposals/{id}
Authorization: Bearer {token}
```

#### Create Proposal
```http
POST /api/v1/governance/proposals
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Increase Quorum Threshold",
  "description": "Proposal to increase governance quorum from 4% to 6% to ensure better participation",
  "actions": [
    {
      "target": "0xGovernanceParams...",
      "value": "0",
      "signature": "updateQuorum(uint256)",
      "calldata": "0x..."
    }
  ]
}
```

### Voting

#### Cast Vote
```http
POST /api/v1/governance/proposals/{id}/vote
Authorization: Bearer {token}
Content-Type: application/json

{
  "support": 1,
  "reason": "I support this proposal because..."
}
```

**Support Values:**
- `0`: Against
- `1`: For  
- `2`: Abstain

#### Get Voting Power
```http
GET /api/v1/governance/voting-power/{address}
Authorization: Bearer {token}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "address": "0x123abc...",
    "voting_power": "1000000000000000000000",
    "delegated_power": "500000000000000000000",
    "total_power": "1500000000000000000000",
    "delegate": "0xDelegate...",
    "delegators_count": 12
  }
}
```

### Delegation

#### Delegate Voting Power
```http
POST /api/v1/governance/delegate
Authorization: Bearer {token}
Content-Type: application/json

{
  "delegate": "0xDelegate..."
}
```

#### Get Delegation Info
```http
GET /api/v1/governance/delegation/{address}
Authorization: Bearer {token}
```

### Token Information

#### Token Supply
```http
GET /api/v1/tokens/supply
```

**Response:**
```json
{
  "success": true,
  "data": {
    "total_supply": "1000000000000000000000000000",
    "circulating_supply": "400000000000000000000000000",
    "locked_supply": "600000000000000000000000000"
  }
}
```

#### Token Holders
```http
GET /api/v1/tokens/holders
Authorization: Bearer {token}
```

## Security API

The Security API provides authentication, validation, monitoring, and access control.

### Session Management

#### Get Session Status
```http
GET /api/v1/security/session/status
Authorization: Bearer {token}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "session_id": "sess_123",
    "user_address": "0x123abc...",
    "expires_at": "2023-07-01T18:00:00Z",
    "last_activity": "2023-07-01T12:30:00Z",
    "is_valid": true
  }
}
```

#### List Active Sessions
```http
GET /api/v1/admin/security/session/sessions
Authorization: Bearer {admin_token}
```

### Validation

#### Get Validation Schemas
```http
GET /api/v1/admin/security/validation/schemas
Authorization: Bearer {admin_token}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "schemas": [
      {
        "name": "proposal",
        "version": "1.0.0",
        "description": "Schema for governance proposals",
        "schema": {
          "type": "object",
          "required": ["title", "description", "actions"],
          "properties": {
            "title": {
              "type": "string",
              "minLength": 10,
              "maxLength": 200
            }
          }
        }
      }
    ]
  }
}
```

### Monitoring

#### List Security Events
```http
GET /api/v1/admin/monitoring/events
Authorization: Bearer {admin_token}
```

#### Create Alert Rule
```http
POST /api/v1/admin/monitoring/rules
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "name": "Failed Login Attempts",
  "description": "Alert when too many failed login attempts",
  "conditions": {
    "event_type": "authentication_failed",
    "threshold": 5,
    "time_window": "5m"
  },
  "actions": ["email", "webhook"]
}
```

## WebSocket API

Real-time updates for governance events, credential changes, and system notifications.

### Connection
```javascript
const ws = new WebSocket('wss://api.iserveprotocol.com/api/v1/ws');
```

### Authentication
After connection, send authentication message:
```json
{
  "type": "auth",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### Subscriptions

#### Subscribe to Proposal Updates
```json
{
  "type": "subscribe",
  "data": {
    "channel": "proposals",
    "filters": {
      "status": ["active", "succeeded"]
    }
  }
}
```

#### Subscribe to Credential Events
```json
{
  "type": "subscribe",
  "data": {
    "channel": "credentials",
    "filters": {
      "holder": "0x123abc..."
    }
  }
}
```

### Event Messages

#### Proposal Event
```json
{
  "type": "proposal_updated",
  "data": {
    "proposal_id": "1",
    "status": "succeeded",
    "voting": {
      "for_votes": "25000000000000000000000000",
      "against_votes": "8000000000000000000000000"
    },
    "timestamp": "2023-07-09T10:00:00Z"
  }
}
```

#### Credential Event
```json
{
  "type": "credential_issued",
  "data": {
    "credential_id": "cred_456",
    "holder": "0x123abc...",
    "issuer": "0x742d35Cc...",
    "title": "Professional Certificate",
    "timestamp": "2023-07-01T15:30:00Z"
  }
}
```

## Error Handling

### Standard Error Response Format
```json
{
  "success": false,
  "error": {
    "code": "INVALID_SIGNATURE",
    "message": "The provided signature is invalid",
    "details": {
      "field": "signature",
      "expected_signer": "0x123abc..."
    },
    "request_id": "req_789xyz"
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Invalid or missing authentication token |
| `FORBIDDEN` | 403 | Insufficient permissions for the operation |
| `NOT_FOUND` | 404 | Requested resource does not exist |
| `INVALID_INPUT` | 400 | Request validation failed |
| `INVALID_SIGNATURE` | 400 | Ethereum signature validation failed |
| `INSUFFICIENT_BALANCE` | 400 | Insufficient token balance for operation |
| `PROPOSAL_NOT_ACTIVE` | 400 | Proposal is not in voting state |
| `ALREADY_VOTED` | 400 | User has already voted on this proposal |
| `RATE_LIMITED` | 429 | Too many requests in time window |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

## Rate Limiting

### Default Limits
- **Public endpoints**: 100 requests/minute
- **Authenticated endpoints**: 1000 requests/minute  
- **Admin endpoints**: 500 requests/minute
- **WebSocket connections**: 10 connections per IP

### Rate Limit Headers
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1672531200
```

### Rate Limit Exceeded Response
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded",
    "details": {
      "limit": 1000,
      "reset_at": "2023-01-01T12:00:00Z"
    }
  }
}
```

## API Versioning

### Current Versions
- **v1**: Current stable version
- **v2**: Future version (planned)

### Version Compatibility
- API versions are maintained for 12 months after deprecation
- Breaking changes require new major version
- Non-breaking changes are added to current version

### Deprecation Notice
Deprecated endpoints include deprecation headers:
```http
X-API-Deprecated: true
X-API-Sunset: 2024-01-01T00:00:00Z
```

---

*This API specification is maintained by the iServe Protocol development team and updated with each release.*