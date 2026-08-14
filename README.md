# architecture

> Personal knowledge base for architectural decisions, system designs, and conceptual frameworks.
> No code. No implementation. Intent, reasoning, and structure only.

---

| | |
|---|---|
| **Type** | Living document repository |
| **Audience** | Technical — assumes familiarity with the systems described |
| **Formats** | Markdown (prose) · Excalidraw (diagrams) · PNG (rendered exports) |
| **Stability** | None guaranteed — check commit history for revision context |

---

## What this is

A place to think in writing. Documents here capture *why* things are designed a certain way, not *how* they are built. They are meant to be referenced, revised, and built upon as systems evolve and understanding sharpens.

This repository is intentionally unbounded. Nothing here is finalized — documents are living artifacts that reflect current thinking at the time of last edit.

## Structure

Organized by subject domain. Each domain may contain written docs, Excalidraw diagram sources, and rendered PNG exports alongside them.

```
architecture/
└── pi/
    ├── harness engineering/
    │   ├── operators-guide.md                              # Operator workflow shifts post-harness
    │   ├── pi-harness-hardening-plan.md                   # Hardening strategy: fitness checks, token control, bounds
    │   └── why-configure-your-harness.md                  # Rationale for harness configuration
    ├── persistent portable memory strategy/
    │   ├── pi-persistent-portable-memory-strategy.md      # Dual-tier memory architecture, vault sync
    │   ├── pi-persistent-portable-memory-strategy.excalidraw
    │   └── pi-persistent-portable-memory-strategy.png
    └── subagent delegation implementation/
        ├── pi-delegation-strategy.md                      # Delegation pipeline, routing, subprocess isolation
        ├── pi-delegation-strategy.excalidraw
        └── pi-delegation-strategy.png
```

## Conventions

- Folder names reflect the subject, not the document type.
- Excalidraw diagrams ship with a rendered PNG alongside the source file.
- Documents assume a technical reader familiar with the system being described.
- Prose is written to capture intent and reasoning — not steps or specifications.
