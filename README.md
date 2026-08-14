# architecture

> A living repository of higher-level design, architectural strategy, and conceptual documentation.
> No source code lives here — only the thinking behind it.

---

## What this is

A personal knowledge base for architectural decisions, system designs, and conceptual frameworks. Documents here are written to capture **intent and reasoning**, not implementation. They are meant to be referenced, revised, and built upon over time.

This repository is deliberately unbounded — documents are added as systems evolve, designs are updated as understanding sharpens, and nothing is considered finalized.

---

## Structure

Documents are organized by subject domain. Each domain folder may contain written docs, diagrams (Excalidraw), and rendered exports.

```
architecture/
└── pi/
    ├── harness engineering/
    │   ├── operators-guide.md                       # Operator workflow shifts post-harness
    │   ├── pi-harness-hardening-plan.md             # Hardening strategy: fitness checks, token control, bounds
    │   └── why-configure-your-harness.md            # Rationale for harness configuration
    ├── persistent portable memory strategy/
    │   ├── pi-persistent-portable-memory-strategy.md          # Dual-tier memory architecture, vault sync
    │   ├── pi-persistent-portable-memory-strategy.excalidraw  # System topology diagram (source)
    │   └── pi-persistent-portable-memory-strategy.png         # Rendered diagram
    └── subagent delegation implementation/
        ├── pi-delegation-strategy.md                # Delegation pipeline, routing, subprocess isolation
        ├── pi-delegation-strategy.excalidraw        # Architecture diagram (source)
        └── pi-delegation-strategy.png               # Rendered diagram
```

---

## Conventions

- **Markdown** for all written documents.
- **Excalidraw** for diagrams, with a rendered PNG alongside each source file.
- Folder names reflect the subject, not the document type.
- Documents are written for a technical audience and assume familiarity with the system being described.

---

## Status

Living document set. Content is added and revised continuously. No version guarantees — check commit history for change context.
