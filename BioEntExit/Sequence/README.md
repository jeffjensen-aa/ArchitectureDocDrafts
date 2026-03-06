# GM Web Biometrics Boarding API - Architecture Diagrams

This folder contains comprehensive C4 architecture diagrams for the GM Web Biometrics Boarding API solution.

## C4 Model Overview

The C4 model provides a hierarchical way to visualize software architecture at different levels of abstraction:

1. **Level 1 - System Context**: Shows the system and its relationships with users and other systems
2. **Level 2 - Container**: Shows the high-level technology choices and how containers communicate
3. **Level 3 - Component**: Shows the components within each container
4. **Level 4 - Code**: Shows the detailed code structure and interactions

## Quick Navigation

### 🎯 Start Here: End-to-End Flow
**[End-to-End Sequence Diagram](End-to-End-Sequence-Diagram.md)** - **Parent diagram** showing complete Ace-to-API-to-Backend flow with references to detailed sub-flows

### 📋 Detailed Sub-Flows (Referenced by Parent)
1. [Device Authentication](Subflow-01-Device-Authentication.md) - JWT validation, 6-step authentication process
2. [Bearer Token Acquisition](Subflow-02-Bearer-Token-Acquisition.md) - OAuth flow, token caching, retry logic
3. [ACE Scan Processing](Subflow-03-Scan-Processing-ACE.md) - Modern ACE API integration for biometric verification
4. [Service Bus Scan Processing](Subflow-04-Scan-Processing-ServiceBus.md) - Legacy async vendor processing flow
5. [Message Matching Loop](Subflow-05-Service-Bus-Message-Matching.md) - Complex 2-level matching algorithm
6. [Device Disconnect](Subflow-06-Device-Disconnect.md) - Cleanup, reconnection strategies, disconnect types

## Diagram Files

### Level 1: System Context
- [`01-System-Context.md`](01-System-Context.md) - High-level view of the system in its environment

### Level 2: Container
- [`02-Container-Diagram.md`](02-Container-Diagram.md) - Technical building blocks and deployment units

### Level 3: Component
- [`03-Component-API-Controllers.md`](03-Component-API-Controllers.md) - API Controllers layer components
- [`03-Component-Business-Logic.md`](03-Component-Business-Logic.md) - Business Logic and Messaging components
- [`03-Component-Socket-Manager.md`](03-Component-Socket-Manager.md) - WebSocket management components
- [`03-Component-Data-Access.md`](03-Component-Data-Access.md) - Data Access layer and domain models

### Level 4: Code & Sequence Diagrams

#### Main Sequence Flows
- [`End-to-End-Sequence-Diagram.md`](End-to-End-Sequence-Diagram.md) - **⭐ Parent diagram** - Complete Ace integration flow

#### Detailed Sub-Flows (Modular)
- [`Subflow-01-Device-Authentication.md`](Subflow-01-Device-Authentication.md) - Authentication and token validation
- [`Subflow-02-Bearer-Token-Acquisition.md`](Subflow-02-Bearer-Token-Acquisition.md) - OAuth bearer token management
- [`Subflow-03-Scan-Processing-ACE.md`](Subflow-03-Scan-Processing-ACE.md) - ACE service scan processing
- [`Subflow-04-Scan-Processing-ServiceBus.md`](Subflow-04-Scan-Processing-ServiceBus.md) - Service Bus async processing
- [`Subflow-05-Service-Bus-Message-Matching.md`](Subflow-05-Service-Bus-Message-Matching.md) - Message matching algorithm
- [`Subflow-06-Device-Disconnect.md`](Subflow-06-Device-Disconnect.md) - Disconnect and cleanup process

#### Legacy Sequence Diagrams
- [`04-Code-Sequence-API-Flow.md`](04-Code-Sequence-API-Flow.md) - API request/response flow (admin operations)
- [`04-Code-Sequence-WebSocket-Scan.md`](04-Code-Sequence-WebSocket-Scan.md) - Original WebSocket flow (now split into sub-flows)

#### Code Structure Diagrams
- [`04-Code-BiometricsBL.md`](04-Code-BiometricsBL.md) - BiometricsBL class structure
- [`04-Code-ServiceBusSend.md`](04-Code-ServiceBusSend.md) - ServiceBusSend messaging implementation
- [`04-Code-SocketManager.md`](04-Code-SocketManager.md) - SocketManager WebSocket implementation
- [`04-Code-BiometricDBContext.md`](04-Code-BiometricDBContext.md) - Database context and entities

## Document Organization

### Modular Approach
The sequence diagrams follow a **modular parent-child structure**:
- **Parent Diagram**: High-level flow with references to sub-diagrams
- **Sub-Diagrams**: Detailed conditional logic, error handling, and edge cases

**Benefits**:
- ✅ Easier to understand overall flow
- ✅ Deep-dive into specific areas without overwhelm
- ✅ Maintainable - update sub-flows independently
- ✅ Reusable - sub-flows can be referenced from multiple parents

### Reading Order

**For New Team Members**:
1. Start with [System Context](01-System-Context.md) - Understand the big picture
2. Read [Container Diagram](02-Container-Diagram.md) - Understand technical components
3. Review [End-to-End Sequence](End-to-End-Sequence-Diagram.md) - See main flow
4. Dive into sub-flows as needed based on your area of work

**For Developers Working On**:
- **WebSocket/Device Management**: Read [Subflow-01](Subflow-01-Device-Authentication.md) and [Subflow-06](Subflow-06-Device-Disconnect.md)
- **ACE Integration**: Read [Subflow-02](Subflow-02-Bearer-Token-Acquisition.md) and [Subflow-03](Subflow-03-Scan-Processing-ACE.md)
- **Service Bus/Messaging**: Read [Subflow-04](Subflow-04-Scan-Processing-ServiceBus.md) and [Subflow-05](Subflow-05-Service-Bus-Message-Matching.md)
- **Admin/Configuration**: Read [API Flow Sequence](04-Code-Sequence-API-Flow.md)

## Viewing the Diagrams

These diagrams use Mermaid syntax and can be viewed in:
- GitHub (native support)
- Visual Studio Code (with Mermaid extension)
- GitLab
- Any Markdown viewer with Mermaid support

## Key Architecture Patterns

- **Layered Architecture**: Clear separation between API, Business Logic, and Data Access layers
- **Dependency Injection**: Loose coupling throughout the system using .NET's built-in DI container
- **Repository Pattern**: Abstracted data access through Entity Framework Core DbContext
- **Event-Driven Architecture**: Azure Service Bus pub/sub for asynchronous device communication
- **WebSocket Management**: Real-time bidirectional communication with biometric devices
- **API Versioning**: Backward compatibility support

## Technology Stack

- **.NET 8** with C# 12.0
- **ASP.NET Core Web API**
- **Entity Framework Core 8**
- **Azure Service Bus** (AMQP over WebSockets)
- **WebSocket Protocol** (WSS for secure device communication)
- **SQL Server**
- **Azure Key Vault** (Secrets management)
- **Application Insights** (Monitoring and diagnostics)
