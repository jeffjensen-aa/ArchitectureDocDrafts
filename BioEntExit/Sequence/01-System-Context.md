# Level 1: System Context Diagram

## Overview

This diagram shows the GM Biometrics Boarding API system in its environment, including all users and external systems it interacts with.

## Diagram

```mermaid
C4Context
    title System Context - GM Biometrics Boarding API

    Person(passenger, "Passenger", "Airport traveler undergoing biometric boarding process")
    Person(gateAgent, "Gate Agent", "Airline staff monitoring and managing boarding operations")
    Person(admin, "System Administrator", "IT staff configuring airports, terminals, and devices")

    System(boardingAPI, "GM Biometrics Boarding API", "Manages biometric boarding process, device communication, passenger verification, and telemetry collection")

    System_Ext(biometricDevices, "Biometric Hardware", "Physical scanners at gates (CDA, NEC, VeriScan, SITA)")
    System_Ext(azureServiceBus, "Azure Service Bus", "Message broker for asynchronous vendor-specific and broadcast messaging")
    System_Ext(sqlDatabase, "SQL Server Database", "Stores airports, terminals, devices, statistics, and health data")
    System_Ext(aceService, "ACE Biometric Events Service", "External CBP biometric verification and status service")
    System_Ext(loyaltyService, "Loyalty Service", "External passenger loyalty profile service")
    System_Ext(azureKeyVault, "Azure Key Vault", "Secure storage for JWT signing keys and service credentials")
    System_Ext(appInsights, "Application Insights", "Telemetry, monitoring, and diagnostics platform")

    Rel(passenger, biometricDevices, "Scans biometrics at gate")
    Rel(gateAgent, boardingAPI, "Monitors boarding status", "HTTPS/REST")
    Rel(admin, boardingAPI, "Configures system", "HTTPS/REST")
    
    Rel(biometricDevices, boardingAPI, "Sends scan results, status updates", "WebSocket")
    Rel(boardingAPI, biometricDevices, "Sends commands, hardware messages", "WebSocket")
    
    Rel(boardingAPI, azureServiceBus, "Publishes/subscribes to messages", "AMQP over WebSockets")
    Rel(boardingAPI, sqlDatabase, "Reads/writes configuration and statistics", "TDS/SQL")
    Rel(boardingAPI, aceService, "Sends biometric verification requests", "HTTPS/REST")
    Rel(boardingAPI, loyaltyService, "Retrieves passenger loyalty data", "HTTPS/REST")
    Rel(boardingAPI, azureKeyVault, "Retrieves secrets and tokens", "HTTPS/REST")
    Rel(boardingAPI, appInsights, "Sends telemetry and logs", "HTTPS")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="2")
```

## System Elements

### Users

| User | Description | Primary Use Cases |
|------|-------------|-------------------|
| **Passenger** | Airport traveler going through the biometric boarding process | Scans biometrics at gate devices |
| **Gate Agent** | Airline staff managing the boarding process | Monitors boarding operations, views device status, manages exceptions |
| **System Administrator** | IT staff responsible for system configuration | Configures airports, terminals, devices; views statistics and health metrics |

### External Systems

| System | Type | Purpose | Protocol |
|--------|------|---------|----------|
| **Biometric Hardware** | Physical Devices | Capture biometric data (facial recognition, fingerprints) | WebSocket (WSS) |
| **Azure Service Bus** | Message Broker | Asynchronous messaging between components and vendors | AMQP over WebSockets |
| **SQL Server Database** | Relational Database | Persistent storage for configuration, statistics, and operational data | TDS/SQL |
| **ACE Biometric Events Service** | External API | CBP (Customs and Border Protection) biometric verification service | HTTPS/REST |
| **Loyalty Service** | External API | Retrieves passenger loyalty profiles and information | HTTPS/REST |
| **Azure Key Vault** | Secret Management | Secure storage for JWT signing keys, connection strings, API credentials | HTTPS/REST |
| **Application Insights** | Monitoring Platform | Telemetry collection, logging, performance monitoring, diagnostics | HTTPS |

## Key Interactions

### Boarding Process Flow
1. **Passenger** scans biometrics at **Biometric Hardware** device
2. **Biometric Hardware** sends scan data to **GM Biometrics Boarding API** via WebSocket
3. **GM Biometrics Boarding API** processes the scan:
   - Publishes scan message to **Azure Service Bus**
   - Sends verification request to **ACE Biometric Events Service**
   - May retrieve loyalty information from **Loyalty Service**
4. API receives response and sends result back to **Biometric Hardware**
5. **Gate Agent** monitors the process via REST API

### Configuration Management
1. **System Administrator** accesses admin endpoints via REST API
2. API reads/writes configuration data to **SQL Server Database**
3. Changes are logged to **Application Insights**

### Security & Secrets Management
1. API retrieves JWT signing keys from **Azure Key Vault**
2. Service credentials and connection strings stored securely in Key Vault
3. WebSocket connections validated using JWT tokens

## Technology Context

- **Communication Protocols**: HTTPS/REST, WebSocket (WSS), AMQP, TDS/SQL
- **Authentication**: JWT tokens, OAuth 2.0 bearer tokens
- **Data Formats**: JSON (primary), SQL (database)
- **Cloud Platform**: Microsoft Azure
- **.NET Platform**: .NET 8

## Next Steps

- See [Level 2: Container Diagram](02-Container-Diagram.md) for details on internal system structure
- See [Level 3: Component Diagrams](03-Component-API-Controllers.md) for component-level architecture
