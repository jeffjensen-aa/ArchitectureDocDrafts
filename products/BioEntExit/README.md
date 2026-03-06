# BioEntExit

BioEntExit is a biometric-based entry and exit management system. It controls and logs physical access to secured facilities using biometric identifiers (e.g., fingerprint, facial recognition, iris scan).

## Architecture Diagrams

| Type | Description |
|------|-------------|
| [Sequence Diagrams](./sequence-diagrams/README.md) | Interaction flows — entry, exit, enrollment, and error handling |
| [C4 Diagrams](./c4-diagrams/README.md) | System context, container, and component views |
| [UML Diagrams](./uml-diagrams/README.md) | Class and state diagrams |
| [ERD Diagrams](./erd-diagrams/README.md) | Data model and entity relationships |
| [Data Flow Diagrams](./data-flow-diagrams/README.md) | How biometric and access data flows through the system |
| [Network & Infrastructure Diagrams](./network-infrastructure-diagrams/README.md) | Physical and logical infrastructure layout |

## Key Concepts

- **Enrollment** — Registering a user's biometric data and access rights in the system.
- **Authentication** — Verifying a user's identity at a checkpoint using a biometric scan.
- **Entry Event** — A successful authentication that grants access and logs an entry record.
- **Exit Event** — A scan or sensor trigger that logs a user's departure.
- **Access Policy** — Rules that determine which users may access which checkpoints during which time windows.
