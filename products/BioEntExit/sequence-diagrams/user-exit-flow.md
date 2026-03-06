# BioEntExit — User Exit Flow

This sequence diagram describes the flow when a registered user exits a secured facility through a BioEntExit exit checkpoint.

## Actors & Components

| Participant | Description |
|-------------|-------------|
| User | The individual departing the facility |
| Exit Terminal | The biometric scanner or passive sensor at the exit point |
| Access Controller | Embedded controller that manages the exit door/turnstile hardware |
| Auth Service | Backend service that matches biometric samples against enrolled templates |
| Audit Log Service | Service that persists access events for compliance and reporting |

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber

    actor User
    participant Terminal as Exit Terminal
    participant Controller as Access Controller
    participant AuthSvc as Auth Service
    participant AuditLog as Audit Log Service

    User->>Terminal: Present biometric or badge at exit
    Terminal->>Terminal: Capture biometric sample

    alt Biometric scan required on exit
        Terminal->>AuthSvc: Submit sample + terminal ID
        AuthSvc->>AuthSvc: Match sample against enrolled templates

        alt Biometric match found
            AuthSvc-->>Terminal: Return user identity
            Terminal->>Controller: Release exit door / open turnstile
            Controller-->>Terminal: Acknowledgement
            Terminal-->>User: Display "Goodbye" + exit tone
            Terminal->>AuditLog: Log exit event (user, checkpoint, timestamp)
            AuditLog-->>Terminal: Acknowledgement
        else No biometric match
            AuthSvc-->>Terminal: No match found
            Terminal-->>User: Display "Not Recognised" + alert tone
            Terminal->>AuditLog: Log unmatched exit attempt (checkpoint, timestamp)
            AuditLog-->>Terminal: Acknowledgement
        end

    else Passive exit sensor (no scan required)
        Note over Terminal: Motion / IR sensor detects departure
        Terminal->>AuditLog: Log anonymous exit event (checkpoint, timestamp)
        AuditLog-->>Terminal: Acknowledgement
        Terminal->>Controller: Release exit door / open turnstile
        Controller-->>Terminal: Acknowledgement
        Terminal-->>User: Exit door opens
    end
```

## Notes

- Some installations operate exits in **passive mode** where a motion or infrared sensor detects departure without requiring a biometric scan.
- Exit events are correlated with entry events in the **Audit Log** to calculate occupancy and dwell time.
- Anti-passback rules (preventing re-entry without a corresponding exit) are enforced by the **Access Policy Service** during the next entry attempt.
