# Sequence Diagram Quick Reference

## 📊 Diagram Structure

```
End-to-End-Sequence-Diagram.md (PARENT)
│
├─ Phase 1: Boarding Configuration
│
├─ Phase 2: Device Connection
│  └─ [Subflow-01] Device Authentication
│     ├─ JWT Signature Validation
│     ├─ Issuer/Audience Checks
│     ├─ Expiration Verification
│     ├─ AppName/ClientID Authorization
│     └─ Database Registration
│
├─ Phase 3: Hardware Command
│
├─ Phase 4: Scan Processing
│  ├─ [Subflow-02] Bearer Token Acquisition (if ACE flow)
│  │  ├─ Check Token Cache
│  │  ├─ OAuth Client Credentials Flow
│  │  └─ Token Caching & Retry
│  │
│  ├─ [Subflow-03] ACE Scan Processing (modern path)
│  │  ├─ POST to ACE API
│  │  ├─ Match/NoMatch Handling
│  │  ├─ Error Scenarios (4xx, 5xx)
│  │  └─ Retry Logic
│  │
│  └─ [Subflow-04] Service Bus Scan Processing (legacy path)
│     ├─ Publish to Topic
│     ├─ Vendor Processing (external)
│     └─ [Subflow-05] Message Matching Loop
│        ├─ 8-second polling (8 iterations)
│        ├─ Batch receive (100 messages)
│        ├─ CorrelationId matching (Level 1)
│        ├─ PassengerUID matching (Level 2)
│        └─ Complete/Abandon messages
│
├─ Phase 5: Device Status
│
├─ Phase 6: Telemetry
│
├─ Phase 7: Admin
│
├─ Phase 8: Health Check
│
└─ Phase 9: Disconnect
   └─ [Subflow-06] Device Disconnect
      ├─ Graceful Disconnect
      ├─ Network Error
      ├─ Auth Failure
      ├─ Idle Timeout
      └─ Server Shutdown
```

## 🎯 Use Cases: Which Diagram to Read?

| I want to... | Read this diagram |
|-------------|-------------------|
| Understand the overall Ace integration flow | [End-to-End Sequence](End-to-End-Sequence-Diagram.md) |
| Debug WebSocket authentication issues | [Subflow-01: Device Authentication](Subflow-01-Device-Authentication.md) |
| Investigate ACE API errors | [Subflow-03: ACE Scan Processing](Subflow-03-Scan-Processing-ACE.md) |
| Troubleshoot OAuth token problems | [Subflow-02: Bearer Token Acquisition](Subflow-02-Bearer-Token-Acquisition.md) |
| Fix Service Bus message matching issues | [Subflow-05: Message Matching Loop](Subflow-05-Service-Bus-Message-Matching.md) |
| Understand why scans timeout | [Subflow-04: Service Bus Processing](Subflow-04-Scan-Processing-ServiceBus.md) + [Subflow-05](Subflow-05-Service-Bus-Message-Matching.md) |
| Debug device disconnect issues | [Subflow-06: Device Disconnect](Subflow-06-Device-Disconnect.md) |
| Understand admin API flows | [API Flow Sequence](04-Code-Sequence-API-Flow.md) |
| See the original combined flow | [WebSocket Scan Flow](04-Code-Sequence-WebSocket-Scan.md) (legacy) |

## 📈 Complexity Levels

| Diagram | Lines of Code* | Conditional Branches | Complexity |
|---------|---------------|---------------------|------------|
| **Parent (End-to-End)** | ~150 | 2 (ACE vs ServiceBus) | ⭐⭐ Medium |
| **Subflow-01: Auth** | ~180 | 6 (validation checks) | ⭐⭐⭐ High |
| **Subflow-02: Token** | ~200 | 5 (cache + errors) | ⭐⭐⭐ High |
| **Subflow-03: ACE** | ~250 | 7 (response types) | ⭐⭐⭐⭐ Very High |
| **Subflow-04: Service Bus** | ~220 | 2 (success/timeout) | ⭐⭐⭐ High |
| **Subflow-05: Matching** | ~300 | 8+ (nested loops) | ⭐⭐⭐⭐⭐ Extreme |
| **Subflow-06: Disconnect** | ~180 | 5 (disconnect types) | ⭐⭐⭐ High |

*Approximate mermaid diagram lines

## 🔄 Flow Decision Tree

```
Scan Received
│
├─ Feature Flag: ACEServiceEnabled AND CorrelationId ends with "ACE"?
│  │
│  YES → ACE Service Flow
│  │    ├─ [Subflow-02] Get Bearer Token
│  │    └─ [Subflow-03] POST to ACE API
│  │
│  NO → Service Bus Flow
│       ├─ [Subflow-04] Publish to Topic
│       └─ [Subflow-05] Poll Queue with Matching Loop
│
└─ Return Response or Timeout
```

## 📝 Key Concepts by Subflow

### Subflow-01: Device Authentication
- **6-step JWT validation** (signature, issuer, audience, expiry, appname, clientid)
- **Fail-closed security** (any failure = disconnect)
- **Database tracking** (WebSocketDevices table)

### Subflow-02: Bearer Token Acquisition
- **1-hour token caching** (with 5-minute expiration buffer)
- **OAuth 2.0 client credentials** grant type
- **Retry once on 500 errors**

### Subflow-03: ACE Scan Processing
- **Synchronous REST API** to CBP biometric service
- **Match/NoMatch** responses with confidence scores
- **30-second timeout** with retry
- **Feature-flagged** activation

### Subflow-04: Service Bus Scan Processing
- **Asynchronous pub/sub** pattern
- **Vendor-specific routing** via correlation filters
- **Two phases**: Publish (50ms) + Receive (1-8 seconds)

### Subflow-05: Message Matching Loop
- **Two-level matching**: CorrelationId (message property) + PassengerUID (body)
- **8 iterations × 1 second** = 8-second timeout
- **Batch receive**: Up to 100 messages per iteration
- **Complete matched, abandon unmatched** messages

### Subflow-06: Device Disconnect
- **5 disconnect types**: Normal, NetworkError, AuthFailure, IdleTimeout, ServerShutdown
- **Cleanup order**: Database → Dictionary → Log → Close → Dispose
- **Reconnection strategies** vary by disconnect reason

## 🎨 Diagram Color Coding

In the parent diagram, sub-flows are highlighted with colored boxes:
- 🔵 **Blue** (rgb(200, 220, 240)) - Authentication/Security flows
- 🟢 **Green** (rgb(220, 240, 220)) - Token acquisition
- 🔴 **Red** (rgb(240, 220, 220)) - ACE processing
- 🟡 **Yellow** (rgb(240, 240, 220)) - Service Bus processing
- 🟣 **Purple** (rgb(240, 220, 240)) - Cleanup/Disconnect

## 📊 Performance Benchmarks

| Flow | Best Case | Average | Worst Case | Timeout |
|------|-----------|---------|------------|---------|
| **ACE Flow** | 500ms | 1000ms | 2500ms | 30s |
| **Service Bus Flow** | 1500ms | 3000ms | 8000ms | 8s |
| **Token Acquisition (cache hit)** | <1ms | <1ms | <1ms | N/A |
| **Token Acquisition (cache miss)** | 500ms | 800ms | 2000ms | 10s |
| **Message Matching (1st iteration)** | 100ms | 100ms | 1100ms | N/A |
| **Message Matching (full timeout)** | N/A | N/A | 8000ms | 8s |

## 🔗 Cross-References

Each sub-diagram includes:
- **Related Flows**: Links to parent and sibling diagrams
- **Configuration Examples**: JSON snippets for feature flags and settings
- **Code Samples**: C# implementation details
- **Monitoring Queries**: Application Insights KQL queries
- **Error Scenarios**: Comprehensive error handling matrices

## 📚 Additional Resources

- **Architecture Overview**: [System Context](01-System-Context.md)
- **Component Details**: [Container Diagram](02-Container-Diagram.md)
- **API Contracts**: See OpenAPI/Swagger documentation
- **Database Schema**: See database migration files

## 🔧 Maintenance Notes

**When to Update Sub-Diagrams**:
- Authentication logic changes → Update [Subflow-01](Subflow-01-Device-Authentication.md)
- OAuth configuration changes → Update [Subflow-02](Subflow-02-Bearer-Token-Acquisition.md)
- ACE API contract changes → Update [Subflow-03](Subflow-03-Scan-Processing-ACE.md)
- Service Bus topology changes → Update [Subflow-04](Subflow-04-Scan-Processing-ServiceBus.md)
- Message matching algorithm changes → Update [Subflow-05](Subflow-05-Service-Bus-Message-Matching.md)
- Disconnect handling changes → Update [Subflow-06](Subflow-06-Device-Disconnect.md)

**Parent diagram only needs updates for**:
- New phases added to overall flow
- Major architectural changes (new backend service)
- Change in sub-flow sequence/dependencies

---

**Last Updated**: March 6, 2026  
**Diagram Format**: Mermaid (sequence diagrams)  
**Model**: C4 Level 4 (Code)
