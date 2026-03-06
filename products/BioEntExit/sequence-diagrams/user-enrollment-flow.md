# BioEntExit — User Enrollment Flow

This sequence diagram describes the process by which an administrator enrolls a new user into the BioEntExit system, capturing their biometric data and configuring their access rights.

## Actors & Components

| Participant | Description |
|-------------|-------------|
| Administrator | Authorised staff member performing the enrollment |
| Admin Portal | Web-based management interface |
| Enrollment Service | Backend service that orchestrates enrollment and template creation |
| Auth Service | Backend service responsible for biometric template storage and matching |
| Access Policy Service | Service that stores and evaluates user access rules |
| Audit Log Service | Service that persists all administrative actions |
| Notification Service | Service that sends confirmations and alerts |

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber

    actor Admin as Administrator
    participant Portal as Admin Portal
    participant EnrollSvc as Enrollment Service
    participant AuthSvc as Auth Service
    participant PolicySvc as Access Policy Service
    participant AuditLog as Audit Log Service
    participant NotifySvc as Notification Service

    Admin->>Portal: Log in and open Enrollment screen
    Portal->>Admin: Display enrollment form

    Admin->>Portal: Enter user details (name, ID, role)
    Admin->>Portal: Initiate biometric capture
    Portal->>EnrollSvc: Start enrollment session (user details)
    EnrollSvc-->>Portal: Return session ID + capture instructions

    loop Biometric sample capture (repeat until quality threshold met)
        Portal->>Admin: Prompt next biometric scan
        Admin->>Portal: Submit biometric sample
        Portal->>EnrollSvc: Forward sample + session ID
        EnrollSvc->>EnrollSvc: Assess sample quality
        alt Sample quality sufficient
            EnrollSvc-->>Portal: Sample accepted
        else Sample quality insufficient
            EnrollSvc-->>Portal: Sample rejected — re-capture required
        end
    end

    EnrollSvc->>AuthSvc: Create biometric template from accepted samples
    AuthSvc->>AuthSvc: Store template linked to user ID
    AuthSvc-->>EnrollSvc: Template stored successfully

    EnrollSvc->>PolicySvc: Assign access rights (user ID, checkpoints, time windows)
    PolicySvc->>PolicySvc: Persist access policy for user
    PolicySvc-->>EnrollSvc: Access policy saved

    EnrollSvc->>AuditLog: Log enrollment event (admin, user ID, timestamp)
    AuditLog-->>EnrollSvc: Acknowledgement

    EnrollSvc->>NotifySvc: Send enrollment confirmation to user and admin
    NotifySvc-->>EnrollSvc: Notification dispatched

    EnrollSvc-->>Portal: Enrollment complete
    Portal-->>Admin: Display success confirmation
```

## Notes

- A minimum number of biometric samples (configurable) must meet the quality threshold before a template is created.
- The enrolling administrator's actions are recorded in the **Audit Log** for accountability.
- Access rights can be updated post-enrollment through the Admin Portal without repeating biometric capture.
