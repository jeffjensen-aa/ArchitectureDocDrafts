# BioEntExit — Failed Authentication Flow

This sequence diagram describes the system's response when a biometric authentication attempt fails at a checkpoint, covering retry logic, lockout behaviour, and operator notification.

## Actors & Components

| Participant | Description |
|-------------|-------------|
| User | The individual attempting to authenticate |
| Entry Terminal | The biometric scanner at the checkpoint |
| Access Controller | Embedded controller managing door/turnstile hardware |
| Auth Service | Backend service that matches biometric samples against enrolled templates |
| Access Policy Service | Service that evaluates access rules and manages lockout state |
| Audit Log Service | Service that persists access events |
| Notification Service | Service that sends real-time alerts to operators |

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

    User->>Terminal: Present biometric
    Terminal->>Terminal: Capture biometric sample
    Terminal->>AuthSvc: Submit sample + terminal ID
    AuthSvc->>AuthSvc: Attempt match against enrolled templates
    AuthSvc-->>Terminal: No match found

    Terminal->>PolicySvc: Increment failed-attempt counter (terminal ID, IP)
    PolicySvc->>PolicySvc: Evaluate lockout policy

    alt Failed attempts below lockout threshold
        PolicySvc-->>Terminal: Retry permitted (attempts remaining: N)
        Terminal-->>User: Display "Not Recognised — Please try again" + alert tone
        Terminal->>AuditLog: Log failed attempt (checkpoint, timestamp, attempt count)
        AuditLog-->>Terminal: Acknowledgement
        Note over User,Terminal: User may retry up to the configured threshold

    else Failed attempts at or above lockout threshold
        PolicySvc-->>Terminal: Terminal LOCKED (lockout duration: T minutes)
        Terminal->>Controller: Keep door locked, activate alert indicator
        Controller-->>Terminal: Acknowledgement
        Terminal-->>User: Display "Access Locked — Contact Security"
        Terminal->>AuditLog: Log lockout event (checkpoint, timestamp, total failed attempts)
        AuditLog-->>Terminal: Acknowledgement
        Terminal->>NotifySvc: Send security alert to operator (checkpoint, timestamp)
        NotifySvc-->>Terminal: Notification dispatched

        Note over Terminal: Terminal remains locked for duration T,<br/>then automatically resets to idle
    end
```

## Notes

- The failed-attempt threshold and lockout duration are configurable per deployment.
- Lockout is tracked **per terminal** to prevent spreading attempts across devices.
- All failed attempts are written to the **Audit Log** regardless of whether a lockout is triggered.
- Operators can manually clear a lockout early via the Admin Portal.
