# Sub-Flow: Bearer Token Acquisition for ACE Service

## Overview

This diagram details the OAuth bearer token acquisition and caching mechanism used for authenticating with the ACE Biometric Events Service.

## Diagram

```mermaid
sequenceDiagram
    participant BL as Business Logic<br/>(BiometricsBL)
    participant Bearer as Bearer Token<br/>Service
    participant Cache as Token Cache<br/>(In-Memory)
    participant ACE_Auth as ACE OAuth<br/>Token Endpoint
    participant AppIns as Application<br/>Insights

    BL->>+Bearer: GetBearerToken()
    Bearer->>AppIns: Log Token Request
    
    Bearer->>+Cache: Check Cached Token<br/>Key: "ACE_SERVICE_TOKEN"
    
    alt Token Found in Cache
        Cache-->>Bearer: Cached Token Object<br/>{AccessToken, ExpiresAt}
        
        Bearer->>Bearer: Check Token Expiration<br/>Is ExpiresAt > DateTime.UtcNow + 5 minutes?
        
        alt Token Still Valid (Not Expired)
            Bearer->>AppIns: Log "Using Cached Token"<br/>{ExpiresIn: (ExpiresAt - Now)}
            Cache-->>-Bearer: Valid Token
            Bearer-->>-BL: Return AccessToken
            Note over BL,Bearer: Cache Hit - Fast Path
        else Token Expired or Expiring Soon
            Bearer->>AppIns: Log "Token Expired or Expiring"
            Note over Bearer: Proceed to Refresh Token
        end
    else Token Not in Cache
        Bearer->>AppIns: Log "No Cached Token Found"
        Cache-->>-Bearer: null
        Note over Bearer: Proceed to Acquire New Token
    end
    
    Note over Bearer,ACE_Auth: === Token Acquisition Flow ===
    
    Bearer->>Bearer: Load OAuth Configuration<br/>- ClientId from config<br/>- ClientSecret from config<br/>- TokenURL from config
    
    Bearer->>Bearer: Build Token Request<br/>grant_type: "client_credentials"<br/>client_id: {ClientId}<br/>client_secret: {ClientSecret}<br/>scope: "biometric.read biometric.write"
    
    Bearer->>AppIns: Log "Requesting New Token"<br/>{ClientId, Endpoint}
    
    Bearer->>+ACE_Auth: POST /oauth/token<br/>Content-Type: application/x-www-form-urlencoded<br/>Body: grant_type=client_credentials&<br/>      client_id={ClientId}&<br/>      client_secret={ClientSecret}&<br/>      scope=biometric.read+biometric.write
    
    alt OAuth Server Success
        ACE_Auth->>ACE_Auth: Validate Client Credentials
        ACE_Auth->>ACE_Auth: Generate Access Token
        
        ACE_Auth-->>-Bearer: 200 OK<br/>{<br/>  "access_token": "eyJhbGci...",<br/>  "token_type": "Bearer",<br/>  "expires_in": 3600,<br/>  "scope": "biometric.read biometric.write"<br/>}
        
        Bearer->>Bearer: Parse Token Response<br/>Extract AccessToken<br/>Calculate ExpiresAt = Now + ExpiresIn
        
        Bearer->>+Cache: Store Token in Cache<br/>Key: "ACE_SERVICE_TOKEN"<br/>Value: {AccessToken, ExpiresAt}<br/>Expiration: 1 hour
        Cache-->>-Bearer: Token Cached
        
        Bearer->>AppIns: Log "Token Acquired Successfully"<br/>{ExpiresIn: 3600, CachedUntil}
        
        Bearer-->>BL: Return AccessToken
        
    else OAuth Server Error - Invalid Credentials
        ACE_Auth-->>Bearer: 401 Unauthorized<br/>{<br/>  "error": "invalid_client",<br/>  "error_description": "Invalid client credentials"<br/>}
        
        Bearer->>AppIns: Log Error "Invalid OAuth Credentials"<br/>{Error, Description}
        
        Bearer->>Bearer: Throw AuthenticationException<br/>("Failed to acquire bearer token: Invalid credentials")
        
        Bearer-->>BL: Exception: AuthenticationException
        
    else OAuth Server Error - Invalid Grant Type
        ACE_Auth-->>Bearer: 400 Bad Request<br/>{<br/>  "error": "unsupported_grant_type",<br/>  "error_description": "Grant type not supported"<br/>}
        
        Bearer->>AppIns: Log Error "Invalid Grant Type"
        
        Bearer->>Bearer: Throw AuthenticationException<br/>("Failed to acquire bearer token: Invalid grant type")
        
        Bearer-->>BL: Exception: AuthenticationException
        
    else OAuth Server Error - Server Error
        ACE_Auth-->>Bearer: 500 Internal Server Error<br/>{<br/>  "error": "server_error",<br/>  "error_description": "Temporary server error"<br/>}
        
        Bearer->>AppIns: Log Error "OAuth Server Error"<br/>{StatusCode: 500}
        
        Bearer->>Bearer: Retry Once After 1 Second
        
        alt Retry Successful
            Bearer->>ACE_Auth: POST /oauth/token (Retry)
            ACE_Auth-->>Bearer: 200 OK + Token
            Bearer->>Cache: Store Token
            Bearer->>AppIns: Log "Token Acquired on Retry"
            Bearer-->>BL: Return AccessToken
        else Retry Failed
            Bearer->>AppIns: Log Error "OAuth Retry Failed"
            Bearer->>Bearer: Throw AuthenticationException<br/>("Failed to acquire bearer token after retry")
            Bearer-->>BL: Exception: AuthenticationException
        end
        
    else Network Error / Timeout
        Note over ACE_Auth: Network Timeout or Connection Error
        
        Bearer->>AppIns: Log Error "Network Error"<br/>{Exception: HttpRequestException}
        
        Bearer->>Bearer: Throw AuthenticationException<br/>("Failed to acquire bearer token: Network error")
        
        Bearer-->>BL: Exception: AuthenticationException
    end
```

## Configuration

### OAuth Settings (appsettings.json)
```json
{
  "AceBiometricEventsService": {
    "AuthURL": "https://auth.ace.cbp.gov",
    "TokenEndpoint": "/oauth/token",
    "ClientId": "biometrics-boarding-api-client",
    "ClientSecret": "*** STORED IN KEY VAULT ***",
    "Scope": "biometric.read biometric.write"
  }
}
```

### Client Secret Storage
- **Not in appsettings**: Stored in Azure Key Vault
- **Retrieved at startup**: Loaded into configuration
- **Rotation**: Supports key rotation without code changes

## Token Caching Strategy

### Cache Implementation
```csharp
// In-memory cache using IMemoryCache
IMemoryCache tokenCache;

public async Task<string> GetBearerToken()
{
    // Try to get from cache
    if (tokenCache.TryGetValue("ACE_SERVICE_TOKEN", out CachedToken cached))
    {
        // Check expiration with 5-minute buffer
        if (cached.ExpiresAt > DateTime.UtcNow.AddMinutes(5))
        {
            return cached.AccessToken;
        }
    }
    
    // Acquire new token
    var token = await AcquireNewToken();
    
    // Cache for 1 hour
    tokenCache.Set("ACE_SERVICE_TOKEN", token, 
        TimeSpan.FromHours(1));
    
    return token.AccessToken;
}
```

### Cache Benefits
- **Reduces OAuth calls**: From 100s per minute to ~1 per hour
- **Improved performance**: No network call on cache hit
- **Cost savings**: Fewer external API calls
- **Resilience**: Continue working if auth server has brief outage

### Expiration Buffer
- **Buffer**: 5 minutes before actual expiration
- **Purpose**: Prevent using token that might expire during request
- **Refresh trigger**: Start refresh 5 minutes early

## OAuth 2.0 Client Credentials Flow

### Grant Type
- **Type**: `client_credentials`
- **Use Case**: Service-to-service authentication
- **No User Context**: Machine-to-machine

### Request Format
```http
POST /oauth/token HTTP/1.1
Host: auth.ace.cbp.gov
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=biometrics-boarding-api-client&
client_secret=***&
scope=biometric.read+biometric.write
```

### Response Format (Success)
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "biometric.read biometric.write"
}
```

### Token Usage
```http
POST /biometric-verification HTTP/1.1
Host: api.ace.cbp.gov
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

## Error Handling

### Error Types and Responses

| Error Type | HTTP Status | Action | Retry? |
|------------|-------------|--------|--------|
| Invalid Credentials | 401 | Log security event, throw exception | No |
| Invalid Grant Type | 400 | Log config error, throw exception | No |
| Expired Client Secret | 401 | Alert operations, throw exception | No |
| Server Error | 500 | Log error, retry once | Yes (1x) |
| Network Timeout | N/A | Log error, retry once | Yes (1x) |
| Rate Limited | 429 | Wait and retry | Yes (with backoff) |

### Retry Logic
```csharp
// Retry configuration
MaxRetries = 1
RetryDelay = 1 second
Timeout = 10 seconds per attempt
```

## Monitoring

### Application Insights Events

#### Token Acquisition Events
- **TokenRequested**: When GetBearerToken() is called
- **TokenCacheHit**: Token found in cache and still valid
- **TokenCacheMiss**: Token not in cache or expired
- **TokenAcquired**: New token successfully acquired
- **TokenAcquisitionFailed**: Failed to acquire token

#### Metrics Tracked
- **Token Acquisition Duration**: Time to get token from OAuth server
- **Cache Hit Rate**: Percentage of cache hits vs misses
- **Token Acquisition Errors**: Count of failures by error type
- **Tokens in Flight**: Concurrent token acquisition requests

### Alerts
- **High Error Rate**: > 5% token acquisition failures
- **Cache Miss Rate**: > 10% cache misses (indicates short TTL)
- **Slow Token Acquisition**: > 5 seconds to acquire token

## Security Considerations

### Client Secret Protection
- **Never Logged**: Client secret never appears in logs
- **Key Vault Storage**: Retrieved securely at startup
- **Rotation Support**: Can rotate without code deployment
- **Environment Separation**: Different secrets per environment

### Token Security
- **HTTPS Only**: Token transmitted over TLS
- **Short-Lived**: 1-hour expiration
- **Scope Limitation**: Only requested scopes granted
- **No Token Logging**: Access token never fully logged (only last 4 chars)

### Audit Trail
- **All Acquisitions Logged**: When and why tokens are acquired
- **Failure Logging**: All failures with context
- **Success Rate Tracking**: Monitor for anomalies

## Performance Characteristics

### Cache Hit Scenario
- **Duration**: < 1 ms
- **Network Calls**: 0
- **Dependencies**: In-memory cache only

### Cache Miss Scenario
- **Duration**: 500-2000 ms (network RTT)
- **Network Calls**: 1 (to OAuth server)
- **Dependencies**: ACE OAuth endpoint

### High-Load Behavior
- **Concurrent Requests**: Multiple threads can safely get token
- **First Request**: Acquires token, subsequent use cache
- **Lock Strategy**: Brief lock during acquisition to prevent duplicate calls

## Related Flows
- **Parent Flow**: [ACE Scan Processing](Subflow-03-Scan-Processing-ACE.md)
- **Used By**: BiometricEventsService for all ACE API calls
- **Configuration**: Azure Key Vault for secret storage

## Version Information
- **Last Updated**: March 6, 2026
- **OAuth Version**: OAuth 2.0
- **Grant Type**: Client Credentials
- **.NET Version**: .NET 8
