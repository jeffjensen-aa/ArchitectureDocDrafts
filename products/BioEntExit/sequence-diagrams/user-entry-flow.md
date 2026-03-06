# BioEntExit — User Entry Flow

This sequence diagram describes the end-to-end flow when a registered user attempts to enter a secured facility through a BioEntExit checkpoint.

## Actors & Components

| Participant | Description |
|-------------|-------------|
| User | The individual seeking entry |
| Entry Terminal | The physical biometric scanning device at the checkpoint |
| Access Controller | Embedded controller that manages the door/turnstile hardware |
| Auth Service | Backend service that matches biometric samples against enrolled templates |
| Access Policy Service | Service that evaluates whether the authenticated user is permitted entry at this checkpoint and time |
| Audit Log Service | Service that persists access events for compliance and reporting |
| Notification Service | Service that sends real-time alerts to facility operators |

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber

    actor User
    participant Terminal as Entry Terminal
    participant Controller as Access Controller
    participant AuthSvc as Auth Service
    participant PolicySvc as Access Policy Service
    participant AuditLog as Audit Log Service
    participant NotifySvc as Notification Service

    User->>Terminal: Present biometric (e.g., fingerprint / face)
    Terminal->>Terminal: Capture biometric sample
    Terminal->>AuthSvc: Submit sample + terminal ID
    AuthSvc->>AuthSvc: Match sample against enrolled templates

    alt Biometric match found
        AuthSvc-->>Terminal: Return user identity + confidence score
        Terminal->>PolicySvc: Request access decision (user ID, checkpoint ID, timestamp)
        PolicySvc->>PolicySvc: Evaluate access policy rules

        alt Access granted
            PolicySvc-->>Terminal: Access GRANTED
            Terminal->>Controller: Unlock door / open turnstile
            Controller-->>Terminal: Acknowledgement
            Terminal-->>User: Display "Access Granted" + audible tone
            Terminal->>AuditLog: Log entry event (user, checkpoint, timestamp, result)
            AuditLog-->>Terminal: Acknowledgement
        else Access denied (policy violation)
            PolicySvc-->>Terminal: Access DENIED (reason: policy)
            Terminal-->>User: Display "Access Denied" + alert tone
            Terminal->>AuditLog: Log denied entry event (user, checkpoint, timestamp, reason)
            AuditLog-->>Terminal: Acknowledgement
            Terminal->>NotifySvc: Send access denial alert to operator
        end

    else No biometric match
        AuthSvc-->>Terminal: No match found
        Terminal-->>User: Display "Not Recognised" + alert tone
        Terminal->>AuditLog: Log failed authentication event (checkpoint, timestamp)
        AuditLog-->>Terminal: Acknowledgement
        Terminal->>NotifySvc: Send unknown-person alert to operator
    end
```

## Notes

- The **Entry Terminal** enforces a configurable timeout; if no valid scan is submitted within the window the terminal resets to idle.
- Confidence score thresholds for biometric acceptance are configurable per deployment environment.
- All events — including failures — are written to the **Audit Log** to support regulatory compliance.
