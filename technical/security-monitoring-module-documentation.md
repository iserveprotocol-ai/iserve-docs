# Security Monitoring Module
## iServe Protocol Governance System

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

The Security Monitoring module provides real-time monitoring, alerting, and dashboard capabilities for security events within the iServe Protocol Governance system. It enables administrators to detect, respond to, and analyze security incidents across the platform.

This module is designed to integrate with other security components in the iServe Protocol Governance system, including IP Security, Multi-Signature Support, and Session Management, to provide a comprehensive security monitoring solution.

---

## Features

### Real-time Security Event Monitoring
- Collection and processing of security events from various system components
- Standardized event format with rich metadata
- Configurable event retention and cleanup

### Configurable Alert Rules
- Rule-based alert generation from security events
- Support for complex conditions and thresholds
- Event correlation and grouping
- Cooldown periods to prevent alert storms

### Multiple Notification Channels
- Email notifications with customizable templates
- Webhook notifications for integration with external systems
- Slack notifications with formatted messages
- Severity-based notification routing

### Auto-resolution Rules
- Automatic resolution of alerts based on configurable conditions
- Time-based resolution for transient issues
- Condition-based resolution for specific scenarios
- Audit trails for auto-resolved alerts

### Security Dashboard
- Comprehensive view of security status
- Alert and event statistics
- Top sources, IPs, and users
- Event timeline visualization
- Real-time updates

### Audit Logging
- Detailed logging of all security-related activities
- Integration with existing audit systems
- Tamper-evident logging

---

## Architecture

The Security Monitoring module follows a layered architecture designed for flexibility, scalability, and ease of integration:

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
│  ┌─────────────┐  ┌─────────────┐                               │
│  │Auto Resolver│  │Config Manager│                              │
│  └─────────────┘  └─────────────┘                               │
├─────────────────────────────────────────────────────────────────┤
│                    Repository Layer                             │
│  ┌─────────────────────────┐  ┌─────────────────────────────┐  │
│  │  Repository Interface   │  │  Memory Repository Impl     │  │
│  └─────────────────────────┘  └─────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                      Data Models                                │
│  ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌────────────────────┐  │
│  │ Alert   │ │ Event   │ │ Rule      │ │ Notification       │  │
│  └─────────┘ └─────────┘ └───────────┘ └────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Event Collection**: Security events are collected from various system components through the `AddSecurityEvent` method.
2. **Event Processing**: The Rule Evaluator processes events against configured alert rules.
3. **Alert Generation**: When rule conditions are met, alerts are generated.
4. **Notification**: The Notification Manager sends alerts through configured channels.
5. **Auto-Resolution**: The Auto Resolver automatically resolves alerts based on configured rules.
6. **Dashboard**: The dashboard provides a real-time view of security status.

---

## Core Components

### Service

The `Service` is the central component that coordinates all monitoring activities:

- Manages configuration
- Processes security events
- Generates alerts
- Coordinates with other components

```go
// Service provides security monitoring functionality
type Service struct {
    config              *Config
    repository          Repository
    auditAdapter        AuditAdapter
    notificationManager *NotificationManager
    ruleEvaluator       *RuleEvaluator
    autoResolver        *AutoResolver
    // ... other fields
}
```

### Repository

The `Repository` interface defines methods for storing and retrieving security monitoring data:

```go
// Repository defines the interface for storing and retrieving security monitoring data
type Repository interface {
    // Alert operations
    CreateAlert(ctx context.Context, alert *Alert) error
    GetAlert(ctx context.Context, id string) (*Alert, error)
    UpdateAlert(ctx context.Context, alert *Alert) error
    DeleteAlert(ctx context.Context, id string) error
    ListAlerts(ctx context.Context, filter *AlertFilter) (*AlertResult, error)
    AddAlertAction(ctx context.Context, alertID string, action *AlertAction) error
    
    // Event operations
    CreateEvent(ctx context.Context, event *SecurityEvent) error
    GetEvent(ctx context.Context, id string) (*SecurityEvent, error)
    ListEvents(ctx context.Context, filter *EventFilter) (*EventResult, error)
    DeleteEventsOlderThan(ctx context.Context, date time.Time) (int, error)
    
    // ... other methods
}
```

### Rule Evaluator

The `RuleEvaluator` evaluates security events against alert rules:

- Processes batches of security events
- Matches events against rule conditions
- Generates alerts when thresholds are met
- Implements cooldown periods to prevent alert storms

### Notification Manager

The `NotificationManager` handles sending notifications for alerts:

- Sends notifications through configured channels
- Formats messages based on alert data
- Respects severity thresholds for each channel
- Handles notification failures gracefully

### Auto Resolver

The `AutoResolver` automatically resolves alerts based on rules:

- Periodically checks for alerts that match auto-resolve rules
- Applies resolution based on configured conditions
- Updates alert status and adds resolution notes
- Maintains audit trail for auto-resolved alerts

### Config Manager

The `ConfigManager` handles loading and saving configuration:

- Loads configuration from files or environment variables
- Validates configuration for security and consistency
- Provides default configuration values
- Saves configuration changes

---

## Installation and Setup

### Prerequisites

- Go 1.19 or later
- Access to storage backend (memory repository included for development)
- SMTP server for email notifications (optional)
- Slack webhook URL for Slack notifications (optional)

### Installation

1. Add the Security Monitoring module to your project:

```bash
go get github.com/iserveprotocol/governance/internal/security/monitoring
```

2. Import the module in your code:

```go
import "github.com/iserveprotocol/governance/internal/security/monitoring"
```

### Basic Setup

```go
package main

import (
    "context"
    "log"
    
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func main() {
    // Create audit adapter
    auditAdapter := monitoring.NewDefaultAuditAdapter()
    
    // Create integration with default configuration
    integration, err := monitoring.CreateDefaultIntegration("/path/to/config")
    if err != nil {
        log.Fatalf("Failed to create security monitoring: %v", err)
    }
    
    // Start the monitoring service
    err = integration.Start()
    if err != nil {
        log.Fatalf("Failed to start security monitoring: %v", err)
    }
    defer integration.Stop()
    
    // Use the monitoring service
    service := integration.GetService()
    
    // Add a security event
    event := &monitoring.SecurityEvent{
        Type:        "user_login",
        Source:      "auth_service",
        Severity:    monitoring.SeverityInfo,
        Description: "User logged in successfully",
        RelatedUserAddress: "0x1234...",
        RelatedIP:   "192.168.1.1",
    }
    
    service.AddSecurityEvent(context.Background(), event)
    
    // ... rest of your application
}
```

---

## Configuration

The Security Monitoring module is configured through a `Config` struct that can be loaded from a JSON file or environment variables.

### Configuration Options

```go
type Config struct {
    // Enabled indicates whether security monitoring is enabled
    Enabled bool `json:"enabled"`
    
    // AlertingEnabled indicates whether alerting is enabled
    AlertingEnabled bool `json:"alertingEnabled"`
    
    // RealTimeMonitoringEnabled indicates whether real-time monitoring is enabled
    RealTimeMonitoringEnabled bool `json:"realTimeMonitoringEnabled"`
    
    // EventRetentionDays is the number of days to retain security events
    EventRetentionDays int `json:"eventRetentionDays"`
    
    // AlertRetentionDays is the number of days to retain alerts
    AlertRetentionDays int `json:"alertRetentionDays"`
    
    // MaxEventsPerMinute is the maximum number of events to process per minute
    MaxEventsPerMinute int `json:"maxEventsPerMinute"`
    
    // AlertRules are the rules for generating alerts
    AlertRules []*AlertRule `json:"alertRules"`
    
    // NotificationChannels are the channels for sending notifications
    NotificationChannels []*NotificationChannel `json:"notificationChannels"`
    
    // AutoResolveRules are the rules for automatically resolving alerts
    AutoResolveRules []*AutoResolveRule `json:"autoResolveRules"`
}
```

### Default Configuration

The module provides a default configuration with sensible defaults:

```go
func DefaultConfig() *Config {
    now := time.Now()
    
    return &Config{
        Enabled:                 true,
        AlertingEnabled:         true,
        RealTimeMonitoringEnabled: true,
        EventRetentionDays:      90,
        AlertRetentionDays:      365,
        MaxEventsPerMinute:      1000,
        AlertRules: []*AlertRule{
            // Default rules for rate limiting, IP blocking, auth failures, etc.
            // ... (see code for details)
        },
        NotificationChannels: []*NotificationChannel{
            // Default email and webhook channels
            // ... (see code for details)
        },
        AutoResolveRules: []*AutoResolveRule{
            // Default auto-resolve rules
            // ... (see code for details)
        },
    }
}
```

### Configuration File

You can create a JSON configuration file:

```json
{
  "enabled": true,
  "alertingEnabled": true,
  "realTimeMonitoringEnabled": true,
  "eventRetentionDays": 90,
  "alertRetentionDays": 365,
  "maxEventsPerMinute": 1000,
  "alertRules": [
    {
      "id": "rate-limit-exceeded",
      "name": "Rate Limit Exceeded",
      "description": "Alerts when rate limits are exceeded multiple times",
      "enabled": true,
      "eventType": "security_rate_limit_exceeded",
      "alertType": "rate_limit",
      "alertSeverity": "medium",
      "alertTitle": "Rate Limit Exceeded",
      "alertDescription": "Rate limits have been exceeded multiple times",
      "groupBy": "relatedIp",
      "windowMinutes": 5,
      "threshold": 5,
      "cooldownMinutes": 15
    }
  ],
  "notificationChannels": [
    {
      "id": "email-admin",
      "name": "Admin Email",
      "type": "email",
      "enabled": true,
      "configuration": {
        "recipients": ["admin@example.com"],
        "subject": "Security Alert: {{alert.title}}"
      },
      "minSeverity": "medium"
    }
  ],
  "autoResolveRules": [
    {
      "id": "auto-resolve-rate-limit",
      "name": "Auto-Resolve Rate Limit Alerts",
      "description": "Automatically resolves rate limit alerts after a period of time",
      "enabled": true,
      "alertType": "rate_limit",
      "resolutionMinutes": 60,
      "resolutionNotes": "Automatically resolved after 60 minutes of no new occurrences"
    }
  ]
}
```

### Environment Variables

You can also configure the module using environment variables:

- `SECURITY_MONITORING_ENABLED`: Enable/disable security monitoring
- `SECURITY_ALERTING_ENABLED`: Enable/disable alerting
- `SECURITY_REALTIME_MONITORING_ENABLED`: Enable/disable real-time monitoring
- `SECURITY_EVENT_RETENTION_DAYS`: Number of days to retain security events
- `SECURITY_ALERT_RETENTION_DAYS`: Number of days to retain alerts
- `SECURITY_MAX_EVENTS_PER_MINUTE`: Maximum number of events to process per minute

---

## API Reference

### Service Methods

#### AddSecurityEvent

Adds a security event to the monitoring system.

```go
func (s *Service) AddSecurityEvent(ctx context.Context, event *SecurityEvent) error
```

**Parameters:**
- `ctx`: Context for the operation
- `event`: The security event to add

**Returns:**
- `error`: Error if the operation fails

#### CreateAlert

Creates a new alert.

```go
func (s *Service) CreateAlert(ctx context.Context, alert *Alert) error
```

**Parameters:**
- `ctx`: Context for the operation
- `alert`: The alert to create

**Returns:**
- `error`: Error if the operation fails

#### UpdateAlert

Updates an existing alert.

```go
func (s *Service) UpdateAlert(ctx context.Context, id string, update *UpdateAlertRequest) (*Alert, error)
```

**Parameters:**
- `ctx`: Context for the operation
- `id`: The ID of the alert to update
- `update`: The update request

**Returns:**
- `*Alert`: The updated alert
- `error`: Error if the operation fails

#### GetAlert

Gets an alert by ID.

```go
func (s *Service) GetAlert(ctx context.Context, id string) (*Alert, error)
```

**Parameters:**
- `ctx`: Context for the operation
- `id`: The ID of the alert to get

**Returns:**
- `*Alert`: The alert
- `error`: Error if the operation fails

#### ListAlerts

Lists alerts based on the provided filter.

```go
func (s *Service) ListAlerts(ctx context.Context, filter *AlertFilter) (*AlertResult, error)
```

**Parameters:**
- `ctx`: Context for the operation
- `filter`: The filter to apply

**Returns:**
- `*AlertResult`: The result containing alerts and pagination info
- `error`: Error if the operation fails

### HTTP API Endpoints

#### Alerts

- `GET /alerts`: List alerts with filtering and pagination
- `GET /alerts/{id}`: Get a specific alert
- `PUT /alerts/{id}`: Update an alert
- `POST /alerts/{id}/actions`: Add an action to an alert

#### Events

- `GET /events`: List security events with filtering and pagination
- `GET /events/{id}`: Get a specific security event
- `POST /events`: Add a new security event

#### Alert Rules

- `GET /alert-rules`: List all alert rules
- `GET /alert-rules/{id}`: Get a specific alert rule
- `POST /alert-rules`: Create a new alert rule
- `PUT /alert-rules/{id}`: Update an alert rule
- `DELETE /alert-rules/{id}`: Delete an alert rule

#### Notification Channels

- `GET /notification-channels`: List all notification channels
- `GET /notification-channels/{id}`: Get a specific notification channel
- `POST /notification-channels`: Create a new notification channel
- `PUT /notification-channels/{id}`: Update a notification channel
- `DELETE /notification-channels/{id}`: Delete a notification channel

#### Auto-Resolve Rules

- `GET /auto-resolve-rules`: List all auto-resolve rules
- `GET /auto-resolve-rules/{id}`: Get a specific auto-resolve rule
- `POST /auto-resolve-rules`: Create a new auto-resolve rule
- `PUT /auto-resolve-rules/{id}`: Update an auto-resolve rule
- `DELETE /auto-resolve-rules/{id}`: Delete an auto-resolve rule

#### Dashboard

- `GET /dashboard`: Get dashboard data

#### Configuration

- `GET /config`: Get the current configuration
- `PUT /config`: Update the configuration

---

## Integration Guide

### Integration with HTTP Servers

You can integrate the Security Monitoring module with your HTTP server using middleware:

```go
package main

import (
    "net/http"
    
    "github.com/gorilla/mux"
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func main() {
    // Create and start monitoring integration
    integration, _ := monitoring.CreateDefaultIntegration("/path/to/config")
    integration.Start()
    defer integration.Stop()
    
    // Create router
    router := mux.NewRouter()
    
    // Register monitoring routes
    monitoringRouter := router.PathPrefix("/api/security/monitoring").Subrouter()
    integration.RegisterRoutes(monitoringRouter)
    
    // Add security monitoring middleware to your routes
    router.Use(func(next http.Handler) http.Handler {
        return monitoring.WithMiddleware(next, integration)
    })
    
    // Add your routes
    router.HandleFunc("/api/users", handleUsers).Methods("GET")
    
    // Start server
    http.ListenAndServe(":8080", router)
}
```

### Integration with Other Security Modules

#### IP Security Integration

```go
package main

import (
    "context"
    
    "github.com/iserveprotocol/governance/internal/security/ipsecurity"
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func integrateWithIPSecurity(ipService *ipsecurity.Service, monitoringIntegration *monitoring.Integration) {
    // Create a handler for IP blocking events
    ipService.AddEventHandler("ip_blocked", func(ctx context.Context, data map[string]interface{}) {
        // Create a security event for IP blocking
        event := monitoring.CreateIPBlockedEvent(
            ctx,
            data["ip"].(string),
            data["reason"].(string),
        )
        
        // Add the event to the monitoring system
        monitoringIntegration.AddSecurityEvent(ctx, event)
    })
}
```

#### Rate Limiting Integration

```go
package main

import (
    "context"
    
    "github.com/iserveprotocol/governance/internal/security/ratelimit"
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func integrateWithRateLimiting(rateLimitService *ratelimit.Service, monitoringIntegration *monitoring.Integration) {
    // Create a handler for rate limit exceeded events
    rateLimitService.AddEventHandler("rate_limit_exceeded", func(ctx context.Context, data map[string]interface{}) {
        // Create a security event for rate limit exceeded
        event := monitoring.CreateRateLimitExceededEvent(
            ctx,
            data["ip"].(string),
            data["path"].(string),
            int(data["limit"].(float64)),
        )
        
        // Add the event to the monitoring system
        monitoringIntegration.AddSecurityEvent(ctx, event)
    })
}
```

#### Authentication Integration

```go
package main

import (
    "context"
    
    "github.com/iserveprotocol/governance/internal/security/session"
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func integrateWithAuthentication(authService *session.Service, monitoringIntegration *monitoring.Integration) {
    // Create a handler for authentication failure events
    authService.AddEventHandler("auth_failure", func(ctx context.Context, data map[string]interface{}) {
        // Create a security event for authentication failure
        event := monitoring.CreateAuthFailureEvent(
            ctx,
            data["ip"].(string),
            data["userAddress"].(string),
            data["reason"].(string),
        )
        
        // Add the event to the monitoring system
        monitoringIntegration.AddSecurityEvent(ctx, event)
    })
}
```

### Creating Custom Security Events

You can create custom security events for your application:

```go
package main

import (
    "context"
    
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func createCustomSecurityEvent(ctx context.Context, monitoringService *monitoring.Service) {
    // Create a custom security event
    event := &monitoring.SecurityEvent{
        Type:        "custom_event_type",
        Source:      "my_application",
        Severity:    monitoring.SeverityMedium,
        Description: "Something interesting happened",
        RelatedUserAddress: "0x1234...",
        RelatedIP:   "192.168.1.1",
        Metadata: map[string]interface{}{
            "custom_field": "custom_value",
            "another_field": 42,
        },
    }
    
    // Add the event to the monitoring system
    monitoringService.AddSecurityEvent(ctx, event)
}
```

---

## Best Practices

### Event Collection

1. **Standardize Event Creation**
   - Use helper functions to create events
   - Include all relevant context
   - Use consistent event types and sources

```go
// Example: Standardized security event creation
func createSecurityEvent(ctx context.Context, monitoringService *monitoring.Service, eventType string, severity monitoring.AlertSeverity, description string, metadata map[string]interface{}) {
    // Get user and session information from context
    userAddress := getUserAddressFromContext(ctx)
    sessionID := getSessionIDFromContext(ctx)
    clientIP := getClientIPFromContext(ctx)
    
    // Create the security event
    event := &monitoring.SecurityEvent{
        Type:              eventType,
        Source:            "auth_service",
        Severity:          severity,
        Description:       description,
        RelatedUserAddress: userAddress,
        RelatedSessionID:  sessionID,
        RelatedIP:         clientIP,
        Timestamp:         time.Now(),
        Metadata:          metadata,
    }
    
    // Add the event to the monitoring system
    monitoringService.AddSecurityEvent(ctx, event)
}
```

2. **Balance Detail and Volume**
   - Include enough detail for analysis
   - Avoid excessive events that could overwhelm the system
   - Use appropriate severity levels

### Alert Rule Configuration

1. **Create Targeted Alert Rules**
   - Define specific conditions for alerts
   - Use appropriate thresholds based on normal activity
   - Implement grouping to correlate related events

```go
// Example: Effective alert rule configuration
loginFailureRule := &monitoring.AlertRule{
    Name:           "Multiple Login Failures",
    Description:    "Detects multiple failed login attempts",
    Enabled:        true,
    EventType:      "security_auth_failure",
    AlertType:      monitoring.TypeAuthFailure,
    AlertSeverity:  monitoring.SeverityMedium,
    AlertTitle:     "Multiple Authentication Failures",
    AlertDescription: "Multiple authentication failures detected for a user",
    GroupBy:        "relatedUserAddress",
    WindowMinutes:  5,
    Threshold:      5,
    CooldownMinutes: 15,
}
```

2. **Tune Alert Rules**
   - Start with conservative thresholds
   - Adjust based on observed patterns
   - Regularly review and update rules

### Notification Configuration

1. **Configure Multiple Channels**
   - Set up redundant notification channels
   - Use different channels for different severity levels
   - Test notification delivery regularly

```go
// Example: Notification channel configuration
channels := []*monitoring.NotificationChannel{
    {
        Name:     "Security Team Email",
        Type:     "email",
        Enabled:  true,
        Configuration: map[string]interface{}{
            "recipients": []string{"security@example.com"},
            "subject":    "Security Alert: {{alert.title}}",
        },
        MinSeverity: monitoring.SeverityMedium,
    },
    {
        Name:     "Critical Alerts Slack",
        Type:     "slack",
        Enabled:  true,
        Configuration: map[string]interface{}{
            "webhookUrl": "https://hooks.slack.com/services/xxx/yyy/zzz",
            "channel":    "#security-alerts",
        },
        MinSeverity: monitoring.SeverityHigh,
    },
}
```

2. **Prevent Alert Fatigue**
   - Use appropriate severity thresholds
   - Implement alert grouping
   - Configure cooldown periods

### Dashboard Usage

1. **Regular Review**
   - Schedule regular dashboard reviews
   - Look for patterns and trends
   - Investigate unusual activity

2. **Custom Dashboards**
   - Create role-specific dashboards
   - Focus on relevant metrics
   - Include actionable information

---

## Troubleshooting

### Common Issues

#### Events Not Generating Alerts

**Symptoms:**
- Security events are being added but no alerts are generated

**Possible Causes:**
- Alert rules not matching events
- Thresholds not being met
- Cooldown period in effect

**Solutions:**
- Check alert rule configuration
- Verify event format matches rule criteria
- Check if thresholds are appropriate
- Review cooldown periods

#### Notifications Not Being Sent

**Symptoms:**
- Alerts are generated but notifications are not received

**Possible Causes:**
- Notification channels misconfigured
- Severity thresholds not met
- External services unavailable

**Solutions:**
- Verify notification channel configuration
- Check alert severity against channel thresholds
- Test external services (SMTP, webhooks)
- Check for errors in logs

#### Performance Issues

**Symptoms:**
- High CPU or memory usage
- Slow response times
- Events not being processed

**Possible Causes:**
- Too many events being processed
- Inefficient alert rules
- Resource constraints

**Solutions:**
- Adjust MaxEventsPerMinute setting
- Optimize alert rules
- Increase resources
- Implement event filtering

### Diagnostic Tools

#### Checking Service Status

```go
// Example: Checking service status
func checkMonitoringStatus(service *monitoring.Service) {
    config := service.GetConfig()
    fmt.Printf("Monitoring enabled: %v\n", config.Enabled)
    fmt.Printf("Alerting enabled: %v\n", config.AlertingEnabled)
    fmt.Printf("Real-time monitoring enabled: %v\n", config.RealTimeMonitoringEnabled)
    fmt.Printf("Event retention days: %d\n", config.EventRetentionDays)
    fmt.Printf("Alert retention days: %d\n", config.AlertRetentionDays)
    fmt.Printf("Max events per minute: %d\n", config.MaxEventsPerMinute)
}
```

#### Testing Event Processing

```go
// Example: Testing event processing
func testEventProcessing(service *monitoring.Service) {
    // Create a test event
    event := &monitoring.SecurityEvent{
        Type:        "test_event",
        Source:      "test",
        Severity:    monitoring.SeverityMedium,
        Description: "Test event",
        RelatedIP:   "192.168.1.1",
    }
    
    // Add the event
    err := service.AddSecurityEvent(context.Background(), event)
    if err != nil {
        fmt.Printf("Error adding event: %v\n", err)
    } else {
        fmt.Println("Event added successfully")
    }
}
```

#### Checking Alert Rules

```go
// Example: Checking alert rules
func listAlertRules(service *monitoring.Service) {
    rules, err := service.ListAlertRules(context.Background())
    if err != nil {
        fmt.Printf("Error listing rules: %v\n", err)
        return
    }
    
    for _, rule := range rules {
        fmt.Printf("Rule: %s (Enabled: %v)\n", rule.Name, rule.Enabled)
        fmt.Printf("  Event Type: %s\n", rule.EventType)
        fmt.Printf("  Threshold: %d in %d minutes\n", rule.Threshold, rule.WindowMinutes)
        fmt.Printf("  Cooldown: %d minutes\n", rule.CooldownMinutes)
    }
}
```

---

## Examples

### Basic Integration Example

```go
package main

import (
    "context"
    "log"
    "net/http"
    "time"
    
    "github.com/gorilla/mux"
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func main() {
    // Create a new security monitoring integration
    integration, err := monitoring.CreateDefaultIntegration("./config")
    if err != nil {
        log.Fatalf("Failed to create security monitoring: %v", err)
    }

    // Start the monitoring service
    err = integration.Start()
    if err != nil {
        log.Fatalf("Failed to start security monitoring: %v", err)
    }
    defer integration.Stop()

    // Create router
    router := mux.NewRouter()

    // Register monitoring routes
    monitoringRouter := router.PathPrefix("/api/security/monitoring").Subrouter()
    integration.RegisterRoutes(monitoringRouter)

    // Add example routes
    router.HandleFunc("/api/login", handleLogin).Methods("POST")
    router.HandleFunc("/api/users", handleListUsers).Methods("GET")

    // Add security monitoring middleware
    router.Use(func(next http.Handler) http.Handler {
        return monitoring.WithMiddleware(next, integration)
    })

    // Start server
    log.Println("Starting server on :8080")
    log.Fatal(http.ListenAndServe(":8080", router))
}

func handleLogin(w http.ResponseWriter, r *http.Request) {
    // Handle login logic
    // ...
}

func handleListUsers(w http.ResponseWriter, r *http.Request) {
    // Handle user listing logic
    // ...
}
```

### Creating Custom Security Events

```go
package main

import (
    "context"
    "log"
    
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func main() {
    // Create a new security monitoring integration
    integration, err := monitoring.CreateDefaultIntegration("./config")
    if err != nil {
        log.Fatalf("Failed to create security monitoring: %v", err)
    }

    // Start the monitoring service
    err = integration.Start()
    if err != nil {
        log.Fatalf("Failed to start security monitoring: %v", err)
    }
    defer integration.Stop()

    // Get the service
    service := integration.GetService()

    // Create and add custom security events
    createConfigChangeEvent(context.Background(), service)
    createUserRoleChangeEvent(context.Background(), service)
    createSystemErrorEvent(context.Background(), service)

    log.Println("Custom security events created")
}

func createConfigChangeEvent(ctx context.Context, service *monitoring.Service) {
    event := &monitoring.SecurityEvent{
        Type:              "config_change",
        Source:            "settings_service",
        Severity:          monitoring.SeverityHigh,
        Description:       "System configuration changed",
        RelatedUserAddress: "0xadmin1234",
        Timestamp:         time.Now(),
        Metadata: map[string]interface{}{
            "component": "governance_settings",
            "change":    "Changed voting threshold from 51% to 67%",
        },
    }
    
    service.AddSecurityEvent(ctx, event)
}

func createUserRoleChangeEvent(ctx context.Context, service *monitoring.Service) {
    event := &monitoring.SecurityEvent{
        Type:              "role_change",
        Source:            "user_management",
        Severity:          monitoring.SeverityMedium,
        Description:       "User role changed",
        RelatedUserAddress: "0xadmin1234",
        Timestamp:         time.Now(),
        Metadata: map[string]interface{}{
            "targetUser": "0xuser5678",
            "role":       "validator",
            "action":     "assigned",
        },
    }
    
    service.AddSecurityEvent(ctx, event)
}

func createSystemErrorEvent(ctx context.Context, service *monitoring.Service) {
    event := &monitoring.SecurityEvent{
        Type:        "system_error",
        Source:      "contract_service",
        Severity:    monitoring.SeverityHigh,
        Description: "System error occurred",
        Timestamp:   time.Now(),
        Metadata: map[string]interface{}{
            "component": "contract_service",
            "error":     "Failed to deploy contract: out of gas",
        },
    }
    
    service.AddSecurityEvent(ctx, event)
}
```

### Custom Alert Rule Configuration

```go
package main

import (
    "context"
    "log"
    
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func main() {
    // Create a new security monitoring integration
    integration, err := monitoring.CreateDefaultIntegration("./config")
    if err != nil {
        log.Fatalf("Failed to create security monitoring: %v", err)
    }

    // Start the monitoring service
    err = integration.Start()
    if err != nil {
        log.Fatalf("Failed to start security monitoring: %v", err)
    }
    defer integration.Stop()

    // Get the service
    service := integration.GetService()

    // Create custom alert rules
    createCustomAlertRules(context.Background(), service)

    log.Println("Custom alert rules created")
}

func createCustomAlertRules(ctx context.Context, service *monitoring.Service) {
    // Create a rule for config changes
    configChangeRule := &monitoring.CreateAlertRuleRequest{
        Name:           "Configuration Change",
        Description:    "Alerts when system configuration is changed",
        Enabled:        true,
        EventType:      "config_change",
        AlertType:      monitoring.TypeConfigChange,
        AlertSeverity:  monitoring.SeverityHigh,
        AlertTitle:     "System Configuration Changed",
        AlertDescription: "A critical system configuration has been modified",
        WindowMinutes:  60,
        Threshold:      1,
        CooldownMinutes: 60,
        NotificationChannels: []string{"email-admin", "slack-security"},
    }
    
    _, err := service.CreateAlertRule(ctx, configChangeRule)
    if err != nil {
        log.Printf("Failed to create config change rule: %v", err)
    }
    
    // Create a rule for multiple system errors
    systemErrorRule := &monitoring.CreateAlertRuleRequest{
        Name:           "Multiple System Errors",
        Description:    "Alerts when multiple system errors occur",
        Enabled:        true,
        EventType:      "system_error",
        AlertType:      monitoring.TypeSystemError,
        AlertSeverity:  monitoring.SeverityCritical,
        AlertTitle:     "Multiple System Errors Detected",
        AlertDescription: "Multiple system errors have occurred in a short period",
        GroupBy:        "source",
        WindowMinutes:  15,
        Threshold:      3,
        CooldownMinutes: 30,
        NotificationChannels: []string{"email-admin", "slack-security", "pagerduty"},
    }
    
    _, err = service.CreateAlertRule(ctx, systemErrorRule)
    if err != nil {
        log.Printf("Failed to create system error rule: %v", err)
    }
}
```

### Custom Notification Channel Configuration

```go
package main

import (
    "context"
    "log"
    
    "github.com/iserveprotocol/governance/internal/security/monitoring"
)

func main() {
    // Create a new security monitoring integration
    integration, err := monitoring.CreateDefaultIntegration("./config")
    if err != nil {
        log.Fatalf("Failed to create security monitoring: %v", err)
    }

    // Start the monitoring service
    err = integration.Start()
    if err != nil {
        log.Fatalf("Failed to start security monitoring: %v", err)
    }
    defer integration.Stop()

    // Get the service
    service := integration.GetService()

    // Create custom notification channels
    createCustomNotificationChannels(context.Background(), service)

    log.Println("Custom notification channels created")
}

func createCustomNotificationChannels(ctx context.Context, service *monitoring.Service) {
    // Create a Slack notification channel
    slackChannel := &monitoring.CreateNotificationChannelRequest{
        Name:     "Security Team Slack",
        Type:     "slack",
        Enabled:  true,
        Configuration: map[string]interface{}{
            "webhookUrl": "https://hooks.slack.com/services/xxx/yyy/zzz",
            "channel":    "#security-alerts",
        },
        MinSeverity: monitoring.SeverityMedium,
    }
    
    _, err := service.CreateNotificationChannel(ctx, slackChannel)
    if err != nil {
        log.Printf("Failed to create Slack channel: %v", err)
    }
    
    // Create an email notification channel
    emailChannel := &monitoring.CreateNotificationChannelRequest{
        Name:     "Security Team Email",
        Type:     "email",
        Enabled:  true,
        Configuration: map[string]interface{}{
            "recipients": []string{"security@example.com", "admin@example.com"},
            "subject":    "Security Alert: {{alert.title}}",
        },
        MinSeverity: monitoring.SeverityMedium,
    }
    
    _, err = service.CreateNotificationChannel(ctx, emailChannel)
    if err != nil {
        log.Printf("Failed to create email channel: %v", err)
    }
    
    // Create a webhook notification channel
    webhookChannel := &monitoring.CreateNotificationChannelRequest{
        Name:     "SIEM Integration",
        Type:     "webhook",
        Enabled:  true,
        Configuration: map[string]interface{}{
            "url":    "https://siem.example.com/api/security/events",
            "method": "POST",
            "headers": map[string]string{
                "Authorization": "Bearer token123",
                "Content-Type": "application/json",
            },
        },
        MinSeverity: monitoring.SeverityLow,
    }
    
    _, err = service.CreateNotificationChannel(ctx, webhookChannel)
    if err != nil {
        log.Printf("Failed to create webhook channel: %v", err)
    }
}
```