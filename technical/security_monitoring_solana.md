# Security Monitoring Module
## iServe Protocol Governance System (Solana)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Features](#features)
3. [Architecture](#architecture)
4. [Core Components](#core-components)
5. [Installation and Setup](#installation-and-setup)
6. [Configuration](#configuration)
7. [API Reference](#api-reference)
8. [Integration Guide](#integration-guide)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Examples](#examples)

---

## Introduction

The Security Monitoring module provides real-time monitoring, alerting, and dashboard capabilities for security events within the iServe Protocol Governance system built on Solana. It enables administrators to detect, respond to, and analyze security incidents across the platform, with specific support for Solana's unique architecture and transaction patterns.

This module is designed to integrate with other security components in the iServe Protocol Governance system, including Solana Program Security, SPL Token Security, Multi-Signature Support, and Session Management, to provide a comprehensive security monitoring solution.

---

## Features

### Real-time Solana Security Event Monitoring
- Collection and processing of security events from Solana programs and transactions
- Standardized event format with Solana-specific metadata (slots, signatures, program IDs)
- Configurable event retention and cleanup optimized for Solana's high throughput

### Solana Program-Aware Alert Rules
- Rule-based alert generation specific to Solana program interactions
- Support for complex conditions including account validation, signature verification, and slot-based timing
- Event correlation and grouping for related Solana transactions
- Cooldown periods to prevent alert storms during network congestion

### Multiple Notification Channels
- Email notifications with Solana transaction details
- Webhook notifications for integration with external SIEM systems
- Slack notifications with formatted Solana explorer links
- Severity-based notification routing

### Auto-resolution Rules
- Automatic resolution of alerts based on Solana network conditions
- Time-based resolution for transient network issues
- Condition-based resolution for specific Solana scenarios
- Audit trails for auto-resolved alerts

### Solana-Native Security Dashboard
- Comprehensive view of Solana program security status
- Alert and event statistics with slot-based timelines
- Top programs, signers, and accounts analysis
- Solana network health integration
- Real-time updates with WebSocket support

### Audit Logging
- Detailed logging of all Solana security-related activities
- Integration with existing audit systems
- Tamper-evident logging with signature verification

---

## Architecture

The Security Monitoring module follows a layered architecture designed for Solana's high-throughput environment:

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Layer                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────┐  │
│  │ Alert API   │  │ Event API   │  │  Rules API  │  │ Other  │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Service Layer                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Service   │  │Rule Evaluator│  │  Notification Manager  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │Auto Resolver│  │Config Manager│  │  Solana RPC Monitor     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                    Repository Layer                             │
│  ┌─────────────────────────┐  ┌─────────────────────────────┐  │
│  │  Repository Interface   │  │  Memory Repository Impl     │  │
│  └─────────────────────────┘  └─────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Data Models                                │
│  ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌────────────────────┐  │
│  │ Alert   │ │ Event   │ │ Rule      │ │ Solana Metadata    │  │
│  └─────────┘ └─────────┘ └───────────┘ └────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Event Collection**: Security events are collected from Solana RPC endpoints and program logs
2. **Event Processing**: The Rule Evaluator processes events against Solana-specific alert rules
3. **Alert Generation**: When rule conditions are met, alerts are generated with Solana context
4. **Notification**: The Notification Manager sends alerts through configured channels
5. **Auto-Resolution**: The Auto Resolver automatically resolves alerts based on Solana network state
6. **Dashboard**: The dashboard provides a real-time view of Solana security status

---

## Core Components

### Solana RPC Monitor

The `SolanaRPCMonitor` component interfaces with Solana RPC endpoints to collect security-relevant data:

- Monitors transaction patterns and anomalies
- Tracks program interaction frequencies
- Detects unusual account activity
- Monitors network health and performance

```javascript
// SolanaRPCMonitor class
class SolanaRPCMonitor {
    constructor(connection, config) {
        this.connection = connection;
        this.config = config;
        this.subscriptions = new Map();
        this.eventBuffer = [];
    }
    
    async startMonitoring() {
        // Monitor account changes for security-relevant accounts
        await this.monitorSecurityAccounts();
        
        // Monitor program logs for security events
        await this.monitorProgramLogs();
        
        // Monitor transaction patterns
        await this.monitorTransactionPatterns();
        
        // Monitor network health
        await this.monitorNetworkHealth();
    }
    
    async monitorSecurityAccounts() {
        for (const account of this.config.monitoredAccounts) {
            const subscription = this.connection.onAccountChange(
                new PublicKey(account),
                (accountInfo, context) => {
                    this.handleAccountChange(account, accountInfo, context);
                },
                'confirmed'
            );
            
            this.subscriptions.set(account, subscription);
        }
    }
    
    async monitorProgramLogs() {
        for (const programId of this.config.monitoredPrograms) {
            const subscription = this.connection.onLogs(
                new PublicKey(programId),
                (logs, context) => {
                    this.handleProgramLogs(programId, logs, context);
                },
                'confirmed'
            );
            
            this.subscriptions.set(`logs_${programId}`, subscription);
        }
    }
}
```

### Service

The `Service` component coordinates all monitoring activities with Solana-specific enhancements:

```javascript
// Enhanced Service for Solana
class SolanaSecurityService extends SecurityService {
    constructor(config, repository, auditAdapter) {
        super(config, repository, auditAdapter);
        this.solanaMonitor = new SolanaRPCMonitor(config.solanaConnection, config);
        this.programAnalyzer = new ProgramSecurityAnalyzer();
        this.networkHealthChecker = new NetworkHealthChecker();
    }
    
    async addSolanaSecurityEvent(event) {
        // Enhance event with Solana-specific metadata
        const enhancedEvent = await this.enhanceEventWithSolanaData(event);
        
        // Validate Solana-specific fields
        this.validateSolanaEvent(enhancedEvent);
        
        // Process the event
        await this.addSecurityEvent(enhancedEvent);
        
        // Trigger Solana-specific analysis
        await this.triggerSolanaAnalysis(enhancedEvent);
    }
    
    async enhanceEventWithSolanaData(event) {
        if (event.transactionSignature) {
            const txInfo = await this.solanaMonitor.connection.getTransaction(
                event.transactionSignature,
                { commitment: 'confirmed' }
            );
            
            if (txInfo) {
                event.solanaMetadata = {
                    slot: txInfo.slot,
                    blockTime: txInfo.blockTime,
                    fee: txInfo.meta.fee,
                    computeUnitsConsumed: txInfo.meta.computeUnitsConsumed,
                    accounts: txInfo.transaction.message.accountKeys.map(key => key.toString()),
                    programIds: this.extractProgramIds(txInfo),
                    logs: txInfo.meta.logMessages
                };
            }
        }
        
        return event;
    }
}
```

### Rule Evaluator

The `RuleEvaluator` processes Solana-specific security rules:

```javascript
// Solana-aware Rule Evaluator
class SolanaRuleEvaluator extends RuleEvaluator {
    constructor(repository) {
        super(repository);
        this.solanaValidators = new Map();
        this.initializeSolanaValidators();
    }
    
    initializeSolanaValidators() {
        this.solanaValidators.set('program_interaction', this.validateProgramInteraction.bind(this));
        this.solanaValidators.set('account_change', this.validateAccountChange.bind(this));
        this.solanaValidators.set('token_transfer', this.validateTokenTransfer.bind(this));
        this.solanaValidators.set('governance_action', this.validateGovernanceAction.bind(this));
    }
    
    async evaluateRule(rule, events) {
        // Apply Solana-specific filtering
        const solanaFilteredEvents = this.applySolanaFilters(rule, events);
        
        // Apply standard rule evaluation
        const result = await super.evaluateRule(rule, solanaFilteredEvents);
        
        // Enhance with Solana context
        if (result.shouldAlert) {
            result.solanaContext = await this.buildSolanaContext(rule, solanaFilteredEvents);
        }
        
        return result;
    }
    
    applySolanaFilters(rule, events) {
        return events.filter(event => {
            // Filter by program ID if specified
            if (rule.programId && event.solanaMetadata?.programIds) {
                if (!event.solanaMetadata.programIds.includes(rule.programId)) {
                    return false;
                }
            }
            
            // Filter by slot range if specified
            if (rule.slotRange && event.solanaMetadata?.slot) {
                const { min, max } = rule.slotRange;
                if (event.solanaMetadata.slot < min || event.solanaMetadata.slot > max) {
                    return false;
                }
            }
            
            // Apply custom Solana validators
            if (rule.solanaValidator) {
                const validator = this.solanaValidators.get(rule.solanaValidator);
                if (validator && !validator(event, rule)) {
                    return false;
                }
            }
            
            return true;
        });
    }
}
```

---

## Installation and Setup

### Prerequisites

- Node.js 16+ with Solana Web3.js support
- Access to Solana RPC endpoints (mainnet-beta, devnet, or local)
- Storage backend (PostgreSQL recommended for high throughput)
- SMTP server for email notifications (optional)
- Slack webhook URL for Slack notifications (optional)

### Installation

1. Add the Security Monitoring module to your Solana project:

```bash
npm install @iserve/security-monitoring @solana/web3.js @solana/spl-token
```

2. Import the module in your code:

```javascript
import { SolanaSecurityMonitoring } from '@iserve/security-monitoring';
import { Connection, PublicKey } from '@solana/web3.js';
```

### Basic Setup

```javascript
import { Connection } from '@solana/web3.js';
import { SolanaSecurityMonitoring } from '@iserve/security-monitoring';

async function main() {
    // Create Solana connection
    const connection = new Connection('https://api.mainnet-beta.solana.com', 'confirmed');
    
    // Create monitoring configuration
    const config = {
        solanaConnection: connection,
        cluster: 'mainnet-beta',
        monitoredPrograms: [
            'iServeCredProg1111111111111111111111111', // Your credential program
            'iServeGovProg111111111111111111111111111'  // Your governance program
        ],
        monitoredAccounts: [
            // Add critical accounts to monitor
        ],
        alerting: {
            enabled: true,
            channels: ['email', 'slack']
        }
    };
    
    // Create integration with Solana-specific configuration
    const integration = await SolanaSecurityMonitoring.create(config);
    
    // Start the monitoring service
    await integration.start();
    
    console.log('Solana security monitoring started');
    
    // Use the monitoring service
    const service = integration.getService();
    
    // Add a Solana security event
    const event = {
        type: 'program_interaction',
        source: 'credential_program',
        severity: 'info',
        description: 'Credential minted successfully',
        transactionSignature: 'YourTransactionSignatureHere',
        relatedUserAddress: 'UserWalletAddressHere',
        relatedProgramId: 'iServeCredProg1111111111111111111111111',
        metadata: {
            instruction: 'mint_credential',
            accounts: ['account1', 'account2'],
            slot: 123456789
        }
    };
    
    await service.addSolanaSecurityEvent(event);
}

main().catch(console.error);
```

---

## Configuration

The Security Monitoring module supports extensive Solana-specific configuration:

### Solana Configuration Options

```javascript
const solanaConfig = {
    // Solana connection settings
    connection: {
        endpoint: 'https://api.mainnet-beta.solana.com',
        commitment: 'confirmed',
        wsEndpoint: 'wss://api.mainnet-beta.solana.com',
        confirmTransactionInitialTimeout: 60000
    },
    
    // Programs to monitor
    monitoredPrograms: [
        'iServeCredProg1111111111111111111111111',
        'iServeGovProg111111111111111111111111111',
        'TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA' // SPL Token program
    ],
    
    // Specific accounts to monitor
    monitoredAccounts: [
        'YourGovernanceTokenMint111111111111111',
        'YourTreasuryAccount111111111111111111'
    ],
    
    // Network monitoring settings
    networkMonitoring: {
        enabled: true,
        slotCheckInterval: 5000,        // Check every 5 seconds
        healthCheckInterval: 30000,     // Health check every 30 seconds
        maxSlotLag: 150,               // Alert if >150 slots behind
        minTps: 1000                   // Alert if TPS < 1000
    },
    
    // Transaction monitoring
    transactionMonitoring: {
        enabled: true,
        maxComputeUnits: 200000,       // Alert on high compute usage
        maxFee: 5000,                  // Alert on high fees (in lamports)
        trackFailedTx: true,           // Monitor failed transactions
        trackLargeTransfers: true      // Monitor large token transfers
    },
    
    // Program-specific monitoring
    programMonitoring: {
        enabled: true,
        maxInstructionsPerTx: 10,      // Alert on complex transactions
        trackCPIDepth: true,           // Monitor cross-program invocations
        trackAccountChanges: true      // Monitor account modifications
    }
};
```

### Alert Rules for Solana

```javascript
const solanaAlertRules = [
    {
        id: 'high-compute-usage',
        name: 'High Compute Usage',
        description: 'Alerts on transactions with excessive compute usage',
        enabled: true,
        eventType: 'program_interaction',
        alertType: 'performance',
        alertSeverity: 'medium',
        conditions: {
            solanaMetadata: {
                computeUnitsConsumed: { '>': 150000 }
            }
        },
        windowMinutes: 5,
        threshold: 10,
        cooldownMinutes: 15
    },
    {
        id: 'failed-governance-transaction',
        name: 'Failed Governance Transaction',
        description: 'Alerts on failed governance transactions',
        enabled: true,
        eventType: 'governance_action',
        alertType: 'governance_failure',
        alertSeverity: 'high',
        conditions: {
            status: 'failed',
            programId: 'iServeGovProg111111111111111111111111111'
        },
        windowMinutes: 60,
        threshold: 1,
        cooldownMinutes: 30
    },
    {
        id: 'unusual-token-transfer',
        name: 'Unusual Token Transfer',
        description: 'Alerts on large or unusual token transfers',
        enabled: true,
        eventType: 'token_transfer',
        alertType: 'suspicious_activity',
        alertSeverity: 'high',
        conditions: {
            amount: { '>': 1000000 }, // Alert on transfers > 1M tokens
            solanaMetadata: {
                programIds: ['TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA']
            }
        },
        windowMinutes: 10,
        threshold: 1,
        cooldownMinutes: 60
    }
];
```

---

## API Reference

### Solana-Specific Service Methods

#### addSolanaSecurityEvent

Adds a Solana security event with enhanced metadata.

```javascript
async function addSolanaSecurityEvent(event) {
    // Enhanced event structure for Solana
    const solanaEvent = {
        type: 'program_interaction',
        source: 'credential_program',
        severity: 'info',
        description: 'Program instruction executed',
        transactionSignature: '5j7s...',  // Solana transaction signature
        relatedUserAddress: '9WzD...',    // Wallet address
        relatedProgramId: 'iServe...',    // Program ID
        solanaMetadata: {
            slot: 123456789,
            blockTime: 1640995200,
            fee: 5000,
            computeUnitsConsumed: 45000,
            instruction: 'mint_credential',
            accounts: ['account1', 'account2'],
            logs: ['Program log: Instruction: MintCredential']
        }
    };
    
    await service.addSolanaSecurityEvent(solanaEvent);
}
```

#### monitorSolanaProgram

Monitors a specific Solana program for security events.

```javascript
async function monitorSolanaProgram(programId, options = {}) {
    const monitoring = await service.monitorSolanaProgram(programId, {
        trackInstructions: true,
        trackAccountChanges: true,
        trackCPICalls: true,
        alertOnFailures: true,
        ...options
    });
    
    return monitoring;
}
```

#### getSolanaNetworkHealth

Returns current Solana network health metrics.

```javascript
async function getSolanaNetworkHealth() {
    const health = await service.getSolanaNetworkHealth();
    
    return {
        currentSlot: health.currentSlot,
        slotLag: health.slotLag,
        tps: health.tps,
        averageBlockTime: health.averageBlockTime,
        epochInfo: health.epochInfo,
        validators: health.validators
    };
}
```

### HTTP API Endpoints

#### Solana-Specific Endpoints

- `GET /solana/network-health`: Get Solana network health status
- `GET /solana/programs/{programId}/activity`: Get program activity metrics
- `GET /solana/transactions/{signature}/analysis`: Get transaction security analysis
- `POST /solana/monitor-program`: Start monitoring a new program
- `DELETE /solana/monitor-program/{programId}`: Stop monitoring a program

---

## Integration Guide

### Integration with Solana Programs

You can integrate the Security Monitoring module directly with your Anchor programs:

```rust
// In your Anchor program
use anchor_lang::prelude::*;

#[program]
pub mod iserve_credential {
    use super::*;
    
    pub fn mint_credential(
        ctx: Context<MintCredential>,
        metadata_uri: String,
    ) -> Result<()> {
        // Your credential minting logic here
        
        // Emit security event for monitoring
        emit!(SecurityEvent {
            event_type: "credential_minted".to_string(),
            user: ctx.accounts.holder.key(),
            metadata: serde_json::json!({
                "instruction": "mint_credential",
                "metadata_uri": metadata_uri,
                "issuer": ctx.accounts.issuer.key().to_string()
            }).to_string(),
        });
        
        Ok(())
    }
}

#[event]
pub struct SecurityEvent {
    pub event_type: String,
    pub user: Pubkey,
    pub metadata: String,
}
```

### Integration with Frontend Applications

```javascript
// Frontend integration with React and Solana wallet adapter
import { useConnection, useWallet } from '@solana/wallet-adapter-react';
import { SolanaSecurityMonitoring } from '@iserve/security-monitoring';

function SecurityDashboard() {
    const { connection } = useConnection();
    const { publicKey } = useWallet();
    const [securityService, setSecurityService] = useState(null);
    
    useEffect(() => {
        const initSecurity = async () => {
            const service = await SolanaSecurityMonitoring.create({
                solanaConnection: connection,
                userWallet: publicKey,
                monitoredPrograms: ['your-program-id']
            });
            
            await service.start();
            setSecurityService(service);
        };
        
        if (connection && publicKey) {
            initSecurity();
        }
    }, [connection, publicKey]);
    
    const handleTransactionSent = async (signature) => {
        if (securityService) {
            await securityService.addSolanaSecurityEvent({
                type: 'user_transaction',
                source: 'frontend',
                severity: 'info',
                description: 'User transaction submitted',
                transactionSignature: signature,
                relatedUserAddress: publicKey.toString()
            });
        }
    };
    
    return (
        <div>
            {/* Your dashboard UI */}
        </div>
    );
}
```

### Integration with Solana Validators

```javascript
// Validator integration for comprehensive monitoring
class ValidatorSecurityIntegration {
    constructor(validatorIdentity, connection) {
        this.validatorIdentity = validatorIdentity;
        this.connection = connection;
        this.securityService = null;
    }
    
    async initialize() {
        this.securityService = await SolanaSecurityMonitoring.create({
            solanaConnection: this.connection,
            validatorMode: true,
            validatorIdentity: this.validatorIdentity,
            monitoringLevel: 'comprehensive'
        });
        
        await this.securityService.start();
    }
    
    async reportValidatorEvent(event) {
        await this.securityService.addSolanaSecurityEvent({
            type: 'validator_event',
            source: 'validator',
            severity: event.severity,
            description: event.description,
            validatorIdentity: this.validatorIdentity.toString(),
            metadata: {
                slot: event.slot,
                epoch: event.epoch,
                leaderSchedule: event.leaderSchedule,
                voteAccount: event.voteAccount
            }
        });
    }
}
```

---

## Best Practices

### Solana-Specific Event Collection

1. **Optimize for High Throughput**
   - Use WebSocket subscriptions for real-time monitoring
   - Implement event batching for high-frequency programs
   - Use appropriate commitment levels for different event types

```javascript
// Example: Optimized event collection for high throughput
class OptimizedSolanaEventCollector {
    constructor(connection, batchSize = 100) {
        this.connection = connection;
        this.batchSize = batchSize;
        this.eventBatch = [];
        this.batchTimeout = null;
    }
    
    collectEvent(event) {
        this.eventBatch.push(event);
        
        if (this.eventBatch.length >= this.batchSize) {
            this.processBatch();
        } else if (!this.batchTimeout) {
            this.batchTimeout = setTimeout(() => {
                this.processBatch();
            }, 1000); // Process batch every second
        }
    }
    
    async processBatch() {
        if (this.eventBatch.length === 0) return;
        
        const batch = [...this.eventBatch];
        this.eventBatch = [];
        
        if (this.batchTimeout) {
            clearTimeout(this.batchTimeout);
            this.batchTimeout = null;
        }
        
        try {
            await this.securityService.processSolanaEventBatch(batch);
        } catch (error) {
            console.error('Failed to process event batch:', error);
            // Implement retry logic or dead letter queue
        }
    }
}
```

2. **Monitor Critical Solana Operations**
   - Track governance proposal submissions and votes
   - Monitor large token transfers and unusual patterns
   - Watch for program upgrade attempts
   - Detect unusual compute unit consumption

### Alert Rule Configuration

1. **Solana Network-Aware Rules**
   - Account for network congestion in alert thresholds
   - Use slot-based timing for time-sensitive rules
   - Consider epoch boundaries for governance events

```javascript
// Example: Network-aware alert rule
const networkAwareRule = {
    id: 'network-congestion-adjusted',
    name: 'Adjusted for Network Congestion',
    description: 'Dynamically adjusts thresholds based on network state',
    enabled: true,
    eventType: 'program_interaction',
    alertType: 'performance',
    alertSeverity: 'medium',
    dynamicThreshold: true,
    thresholdCalculator: async (networkHealth) => {
        const baseFee = 5000;
        const congestionMultiplier = networkHealth.slotLag > 100 ? 3 : 1;
        return baseFee * congestionMultiplier;
    },
    conditions: {
        solanaMetadata: {
            fee: { '>': 'DYNAMIC' } // Will use calculated threshold
        }
    }
};
```

2. **Program-Specific Monitoring**
   - Create rules specific to your program's instruction patterns
   - Monitor for unexpected cross-program invocations
   - Track account size changes and rent implications

---

## Troubleshooting

### Common Solana-Specific Issues

#### RPC Rate Limiting

**Symptoms:**
- Events not being collected
- WebSocket disconnections
- API rate limit errors

**Solutions:**
- Use multiple RPC endpoints with load balancing
- Implement exponential backoff for rate-limited requests
- Consider using dedicated RPC services for high-volume monitoring

```javascript
// Example: RPC rate limiting mitigation
class RateLimitedSolanaConnection {
    constructor(endpoints) {
        this.endpoints = endpoints;
        this.currentEndpointIndex = 0;
        this.connection = new Connection(endpoints[0]);
        this.requestQueue = [];
        this.isProcessing = false;
    }
    
    async makeRequest(method, ...args) {
        return new Promise((resolve, reject) => {
            this.requestQueue.push({ method, args, resolve, reject });
            this.processQueue();
        });
    }
    
    async processQueue() {
        if (this.isProcessing) return;
        this.isProcessing = true;
        
        while (this.requestQueue.length > 0) {
            const { method, args, resolve, reject } = this.requestQueue.shift();
            
            try {
                const result = await this.connection[method](...args);
                resolve(result);
                await this.sleep(100); // Rate limiting delay
            } catch (error) {
                if (error.message.includes('rate limit')) {
                    this.rotateEndpoint();
                    this.requestQueue.unshift({ method, args, resolve, reject });
                    await this.sleep(1000);
                } else {
                    reject(error);
                }
            }
        }
        
        this.isProcessing = false;
    }
}
```

#### High Memory Usage

**Symptoms:**
- Memory leaks in long-running monitoring
- Out of memory errors
- Slow performance

**Solutions:**
- Implement proper event cleanup and retention policies
- Use streaming processing for large event volumes
- Monitor and limit WebSocket subscription count

#### Network Latency Issues

**Symptoms:**
- Delayed event processing
- Missed real-time events
- Inconsistent monitoring coverage

**Solutions:**
- Use geographically distributed RPC endpoints
- Implement local caching for frequently accessed data
- Set appropriate timeout values for network operations

---

## Examples

### Comprehensive Solana Program Monitoring

```javascript
// Complete example: Monitoring a Solana-based credential program
import { Connection, PublicKey } from '@solana/web3.js';
import { SolanaSecurityMonitoring } from '@iserve/security-monitoring';

class CredentialProgramMonitor {
    constructor(programId, connection) {
        this.programId = new PublicKey(programId);
        this.connection = connection;
        this.securityService = null;
    }
    
    async initialize() {
        const config = {
            solanaConnection: this.connection,
            cluster: 'mainnet-beta',
            monitoredPrograms: [this.programId.toString()],
            alertRules: [
                {
                    id: 'unauthorized-credential-mint',
                    name: 'Unauthorized Credential Minting Attempt',
                    description: 'Detects attempts to mint credentials by unauthorized issuers',
                    enabled: true,
                    eventType: 'program_interaction',
                    alertType: 'security_violation',
                    alertSeverity: 'critical',
                    conditions: {
                        instruction: 'mint_credential',
                        isAuthorizedIssuer: false
                    },
                    threshold: 1,
                    cooldownMinutes: 5
                },
                {
                    id: 'high-volume-credential-minting',
                    name: 'High Volume Credential Minting',
                    description: 'Detects unusually high credential minting activity',
                    enabled: true,
                    eventType: 'program_interaction',
                    alertType: 'anomaly',
                    alertSeverity: 'medium',
                    conditions: {
                        instruction: 'mint_credential'
                    },
                    windowMinutes: 60,
                    threshold: 100,
                    cooldownMinutes: 30
                }
            ],
            notificationChannels: [
                {
                    name: 'Security Team Slack',
                    type: 'slack',
                    enabled: true,
                    configuration: {
                        webhookUrl: process.env.SLACK_WEBHOOK_URL,
                        channel: '#security-alerts'
                    },
                    minSeverity: 'medium'
                }
            ]
        };
        
        this.securityService = await SolanaSecurityMonitoring.create(config);
        await this.securityService.start();
        
        // Set up program-specific monitoring
        await this.setupProgramMonitoring();
    }
    
    async setupProgramMonitoring() {
        // Monitor program logs
        this.connection.onLogs(
            this.programId,
            (logs, context) => {
                this.handleProgramLogs(logs, context);
            },
            'confirmed'
        );
        
        // Monitor specific accounts related to the program
        await this.monitorProgramAccounts();
    }
    
    async handleProgramLogs(logs, context) {
        for (const log of logs.logs) {
            if (log.includes('mint_credential')) {
                await this.processCredentialMintEvent(logs, context);
            } else if (log.includes('ERROR') || log.includes('UNAUTHORIZED')) {
                await this.processErrorEvent(logs, context);
            }
        }
    }
    
    async processCredentialMintEvent(logs, context) {
        const event = {
            type: 'program_interaction',
            source: 'credential_program',
            severity: 'info',
            description: 'Credential minting detected',
            transactionSignature: logs.signature,
            relatedProgramId: this.programId.toString(),
            solanaMetadata: {
                slot: context.slot,
                logs: logs.logs,
                instruction: 'mint_credential'
            }
        };
        
        // Enhance with transaction details
        try {
            const txInfo = await this.connection.getTransaction(logs.signature);
            if (txInfo) {
                event.solanaMetadata.accounts = txInfo.transaction.message.accountKeys.map(k => k.toString());
                event.solanaMetadata.fee = txInfo.meta.fee;
                event.solanaMetadata.computeUnitsConsumed = txInfo.meta.computeUnitsConsumed;
            }
        } catch (error) {
            console.warn('Failed to get transaction details:', error);
        }
        
        await this.securityService.addSolanaSecurityEvent(event);
    }
    
    async processErrorEvent(logs, context) {
        const event = {
            type: 'program_error',
            source: 'credential_program',
            severity: 'high',
            description: 'Program error detected',
            transactionSignature: logs.signature,
            relatedProgramId: this.programId.toString(),
            solanaMetadata: {
                slot: context.slot,
                logs: logs.logs,
                errorType: 'program_execution_error'
            }
        };
        
        await this.securityService.addSolanaSecurityEvent(event);
    }
    
    async monitorProgramAccounts() {
        // Monitor governance token mint account
        const govTokenMint = new PublicKey('YourGovernanceTokenMint');
        this.connection.onAccountChange(
            govTokenMint,
            (accountInfo, context) => {
                this.handleTokenMintChange(accountInfo, context);
            },
            'confirmed'
        );
        
        // Monitor treasury account
        const treasuryAccount = new PublicKey('YourTreasuryAccount');
        this.connection.onAccountChange(
            treasuryAccount,
            (accountInfo, context) => {
                this.handleTreasuryChange(accountInfo, context);
            },
            'confirmed'
        );
    }
    
    async handleTokenMintChange(accountInfo, context) {
        const event = {
            type: 'account_change',
            source: 'token_program',
            severity: 'medium',
            description: 'Governance token mint account changed',
            solanaMetadata: {
                slot: context.slot,
                account: 'governance_token_mint',
                lamports: accountInfo.lamports,
                owner: accountInfo.owner.toString()
            }
        };
        
        await this.securityService.addSolanaSecurityEvent(event);
    }
    
    async getSecurityDashboard() {
        const dashboard = await this.securityService.getDashboard();
        
        // Add Solana-specific metrics
        const solanaMetrics = {
            networkHealth: await this.securityService.getSolanaNetworkHealth(),
            programActivity: await this.securityService.getProgramActivity(this.programId.toString()),
            recentTransactions: await this.getRecentProgramTransactions()
        };
        
        return {
            ...dashboard,
            solanaMetrics
        };
    }
    
    async getRecentProgramTransactions() {
        try {
            const signatures = await this.connection.getSignaturesForAddress(
                this.programId,
                { limit: 50 },
                'confirmed'
            );
            
            return signatures.map(sig => ({
                signature: sig.signature,
                slot: sig.slot,
                blockTime: sig.blockTime,
                err: sig.err
            }));
        } catch (error) {
            console.error('Failed to get recent transactions:', error);
            return [];
        }
    }
}

// Usage
async function main() {
    const connection = new Connection('https://api.mainnet-beta.solana.com');
    const monitor = new CredentialProgramMonitor('YourProgramIdHere', connection);
    
    await monitor.initialize();
    
    console.log('Credential program monitoring started');
    
    // Get dashboard data periodically
    setInterval(async () => {
        const dashboard = await monitor.getSecurityDashboard();
        console.log('Security Dashboard:', dashboard);
    }, 60000); // Every minute
}

main().catch(console.error);
```

This comprehensive example demonstrates how to monitor a Solana credential program with the Security Monitoring module, including real-time event collection, alert generation, and dashboard reporting with Solana-specific enhancements.