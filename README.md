# Architecture Documentation Drafts

This repository contains architecture documentation for multiple products. Diagrams are written in [Mermaid](https://mermaid.js.org/) and rendered inline within Markdown files.

## Products

| Product | Description |
|---------|-------------|
| [BioEntExit](./products/BioEntExit/README.md) | Biometric entry and exit management system |
| [Product2](./products/Product2/README.md) | *(Placeholder — to be named)* |
| [Product3](./products/Product3/README.md) | *(Placeholder — to be named)* |

## Diagram Types Supported

| Type | Description |
|------|-------------|
| [Sequence Diagrams](https://mermaid.js.org/syntax/sequenceDiagram.html) | Interaction flows between components over time |
| [C4 Diagrams](https://c4model.com/) | Context, Container, Component, and Code views |
| [UML Diagrams](https://mermaid.js.org/syntax/classDiagram.html) | Class, state, activity, and other UML views |
| [ERD Diagrams](https://mermaid.js.org/syntax/entityRelationshipDiagram.html) | Entity Relationship Diagrams for data modeling |
| [Data Flow Diagrams](https://mermaid.js.org/syntax/flowchart.html) | How data moves through a system |
| [Network & Infrastructure Diagrams](https://mermaid.js.org/syntax/flowchart.html) | Logical and physical infrastructure layouts |

## Repository Structure

```
products/
└── <ProductName>/
    ├── README.md
    ├── sequence-diagrams/
    ├── c4-diagrams/
    ├── uml-diagrams/
    ├── erd-diagrams/
    ├── data-flow-diagrams/
    └── network-infrastructure-diagrams/
```

## Viewing Diagrams

Mermaid diagrams render natively in:
- **GitHub** — Mermaid fenced code blocks (` ```mermaid `) render automatically in Markdown files.
- **VS Code** — Install the [Markdown Preview Mermaid Support](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) extension.
- **Mermaid Live Editor** — Paste diagram source at [mermaid.live](https://mermaid.live) for quick iteration.
