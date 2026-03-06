# Sub-Flow: Bearer Token Acquisition for ACE Service

## Overview

This diagram details the OAuth bearer token acquisition and caching mechanism used for authenticating with the ACE Biometric Events Service.

## Diagram

```mermaid
sequenceDiagram
    participant BL as BiometricsBL (Business Logic)
    participant Bearer as Bearer Token Service
    participant Cache as In-Memory Token Cache
    participant ACE as ACE OAuth Token Endpoint
    participant AI as Application Insights

    BL->>+Bearer: GetBearerToken()
    Bearer->>AI: Log: TokenRequested

    Bearer->>+Cache: TryGet("ACE_SERVICE_TOKEN")

    alt Cache hit
        Cache-->>Bearer: CachedToken { token, expiresAt }
        Bearer->>Bearer: expiresAt > (UtcNow + 5m)?

        alt Token valid
            Bearer->>AI: Log: TokenCacheHit (ttl remaining)
            Bearer-->>-BL: access_token
        else Expiring/expired
            Bearer->>AI: Log: TokenCacheStale
        end

    else Cache miss
        Cache-->>Bearer: null
        Bearer->>AI: Log: TokenCacheMiss
    end

    Note over Bearer,ACE: Token acquisition (OAuth2 client_credentials)\nConfig: ClientId, ClientSecret (Key Vault), TokenUrl, Scope\nPOST x-www-form-urlencoded: grant_type=client_credentials; scope=biometric.read biometric.write

    Bearer->>AI: Log: TokenAcquireStart
    Bearer->>+ACE: POST /oauth/token

    alt 200 OK
        ACE-->>-Bearer: TokenResponse (access_token, expires_in)
        Bearer->>Bearer: expiresAt = UtcNow + expires_in
        Bearer->>+Cache: Set("ACE_SERVICE_TOKEN", token, ttl=1h)
        Cache-->>-Bearer: cached
        Bearer->>AI: Log: TokenAcquired
        Bearer-->>BL: access_token

    else 401 invalid_client
        ACE-->>Bearer: 401 Unauthorized
        Bearer->>AI: Log: TokenAcquireFailed (invalid_client)
        Bearer-->>BL: throw AuthenticationException

    else 400 unsupported_grant_type
        ACE-->>Bearer: 400 Bad Request
        Bearer->>AI: Log: TokenAcquireFailed (unsupported_grant_type)
        Bearer-->>BL: throw AuthenticationException

    else 5xx server error
        ACE-->>Bearer: 5xx
        Bearer->>AI: Log: TokenAcquireFailed (server_error)
        Bearer->>Bearer: Retry once after 1s

        alt Retry succeeds
            Bearer->>ACE: POST /oauth/token (retry)
            ACE-->>Bearer: 200 OK (token)
            Bearer->>Cache: Set("ACE_SERVICE_TOKEN", token, ttl=1h)
            Bearer->>AI: Log: TokenAcquired (retry)
            Bearer-->>BL: access_token
        else Retry fails
            Bearer->>AI: Log: TokenAcquireFailed (retry_failed)
            Bearer-->>BL: throw AuthenticationException
        end

    else Network timeout / connection error
        Bearer->>AI: Log: TokenAcquireFailed (network_error)
        Bearer-->>BL: throw AuthenticationException
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
