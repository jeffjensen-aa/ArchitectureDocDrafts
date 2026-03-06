# Sub-Flow: Device WebSocket Authentication

## Overview

This diagram details the device authentication flow when a biometric device establishes a WebSocket connection with the Boarding API.

## Diagram

```mermaid
sequenceDiagram
    participant Device as Biometric<br/>Device
    participant Socket as Socket Manager
    participant KV as Azure<br/>Key Vault
    participant DB as SQL Server<br/>Database
    participant AppIns as Application<br/>Insights

    Device->>+Socket: WebSocket Connect Request<br/>wss://api.example.com/ws
    Socket->>AppIns: Log Connection Attempt
    
    Device->>Socket: Send Auth Message<br/>{Token: JWT, Route: "Register", DeviceName}
    
    Socket->>Socket: Extract JWT from Message
    
    Socket->>+KV: GET JWT Signing Key<br/>(RSA Public Key)
    KV-->>-Socket: RSA Public Key
    
    Socket->>Socket: ValidateToken(JWT)<br/>Step 1: Verify Signature with Public Key
    
    alt Signature Invalid
        Socket->>AppIns: Log "Invalid JWT Signature"
        Socket-->>Device: "Unauthorized - Invalid Signature"
        Socket->>Socket: Close WebSocket Connection
        Socket-->>-Device: Connection Terminated
        Note over Device,Socket: Authentication Failed
    end
    
    Socket->>Socket: ValidateToken(JWT)<br/>Step 2: Check Issuer
    
    alt Issuer Mismatch
        Socket->>AppIns: Log "Invalid Issuer"
        Socket-->>Device: "Unauthorized - Invalid Issuer"
        Socket->>Socket: Close WebSocket Connection
        Socket-->>-Device: Connection Terminated
        Note over Device,Socket: Authentication Failed
    end
    
    Socket->>Socket: ValidateToken(JWT)<br/>Step 3: Check Audience
    
    alt Audience Mismatch
        Socket->>AppIns: Log "Invalid Audience"
        Socket-->>Device: "Unauthorized - Invalid Audience"
        Socket->>Socket: Close WebSocket Connection
        Socket-->>-Device: Connection Terminated
        Note over Device,Socket: Authentication Failed
    end
    
    Socket->>Socket: ValidateToken(JWT)<br/>Step 4: Verify Expiration
    
    alt Token Expired
        Socket->>AppIns: Log "Expired Token"
        Socket-->>Device: "Unauthorized - Token Expired"
        Socket->>Socket: Close WebSocket Connection
        Socket-->>-Device: Connection Terminated
        Note over Device,Socket: Authentication Failed
    end
    
    Socket->>Socket: ValidateToken(JWT)<br/>Step 5: Check App Name Claim
    
    alt App Name Not in Allowed List
        Socket->>AppIns: Log "Invalid App Name"
        Socket-->>Device: "Unauthorized - Invalid App Name"
        Socket->>Socket: Close WebSocket Connection
        Socket-->>-Device: Connection Terminated
        Note over Device,Socket: Authentication Failed
    end
    
    Socket->>Socket: ValidateToken(JWT)<br/>Step 6: Check Client ID Claim
    
    alt Client ID Not in Allowed List
        Socket->>AppIns: Log "Invalid Client ID"
        Socket-->>Device: "Unauthorized - Invalid Client ID"
        Socket->>Socket: Close WebSocket Connection
        Socket-->>-Device: Connection Terminated
        Note over Device,Socket: Authentication Failed
    end
    
    Note over Socket: All Validation Checks Passed
    
    Socket->>+DB: INSERT INTO WebSocketDevices<br/>(DeviceName, ConnectionTime, Status, Region)<br/>VALUES (?, GETDATE(), 'Connected', ?)
    DB-->>-Socket: Device Registered (ID returned)
    
    Socket->>Socket: Add to Active Connections Dictionary<br/>Dictionary[DeviceName] = WebSocket
    
    Socket->>AppIns: Log Device Connected<br/>{DeviceName, ConnectionTime, IP}
    
    Socket-->>Device: WebSocket Acknowledged<br/>"Registration Successful"
    
    Note over Device,Socket: Authentication Successful<br/>WebSocket Connection Active
```

## Validation Steps Detail

### 1. JWT Signature Verification
- **Algorithm**: RSA256
- **Key Source**: Azure Key Vault
- **Purpose**: Ensures token wasn't tampered with
- **Failure Action**: Immediate disconnect

### 2. Issuer Validation
- **Expected**: Configured issuer from appsettings
- **Example**: "https://biometrics.aa.com"
- **Purpose**: Verify token source
- **Failure Action**: Immediate disconnect

### 3. Audience Validation
- **Expected**: API endpoint or service identifier
- **Example**: "biometrics-boarding-api"
- **Purpose**: Ensure token is for this service
- **Failure Action**: Immediate disconnect

### 4. Expiration Check
- **Claim**: `exp` (Unix timestamp)
- **Validation**: `exp > DateTime.UtcNow`
- **Purpose**: Prevent replay attacks
- **Failure Action**: Immediate disconnect

### 5. App Name Claim
- **Claim**: Custom claim "appName"
- **Validation**: Must be in allowed list from configuration
- **Example**: ["BiometricsDevice", "VeriScan", "NECScanner"]
- **Purpose**: Application-level authorization
- **Failure Action**: Immediate disconnect

### 6. Client ID Claim
- **Claim**: Custom claim "clientId" or standard "azp"
- **Validation**: Must be in allowed list from configuration
- **Example**: ["device-client-001", "device-client-002"]
- **Purpose**: Client-level authorization
- **Failure Action**: Immediate disconnect

## Database Registration

### WebSocketDevices Table Insert
```sql
INSERT INTO WebSocketDevices 
(DeviceName, ConnectionTime, Status, Region, IPAddress)
VALUES 
(?, GETDATE(), 'Connected', ?, ?)
```

**Fields**:
- **DeviceName**: Unique device identifier (e.g., "VERISCAN-DFW-A5-001")
- **ConnectionTime**: UTC timestamp
- **Status**: 'Connected'
- **Region**: Derived from device name or claims
- **IPAddress**: Client IP from WebSocket context

## In-Memory State Management

### Active Connections Dictionary
```csharp
Dictionary<string, WebSocket> activeConnections = new();
activeConnections[deviceName] = webSocket;
```

**Purpose**: Fast lookup for sending messages to specific devices

## Error Handling

### Connection Errors
- **Invalid Token**: Log security event, close connection
- **Database Error**: Log error, close connection gracefully
- **Key Vault Error**: Log error, retry once, then fail

### Logging Events
All events logged to Application Insights with:
- **DeviceName**
- **Timestamp**
- **IP Address**
- **Validation Step Failed** (if applicable)
- **Error Message**

## Security Considerations

### Token Validation is Critical
- **All 6 steps must pass** for authentication
- **No partial authentication** allowed
- **Fail-closed** approach: any failure = disconnect

### Audit Trail
- **All connection attempts** logged
- **Failed validations** logged as security events
- **Successful connections** tracked for monitoring

### Rate Limiting
- Implicit through Azure App Service
- Can add custom rate limiting per device

## Related Flows
- **Parent Flow**: [End-to-End Sequence Diagram](End-to-End-Sequence-Diagram.md)
- **Next Flow**: [Scan Processing](Subflow-03-Scan-Processing-ACE.md)
- **Disconnect Flow**: [Device Disconnect](Subflow-06-Device-Disconnect.md)

## Version Information
- **Last Updated**: March 6, 2026
- **.NET Version**: .NET 8
- **Security**: JWT with RSA256
