# Pi Hermes Obsidian Memory: Persistent & Portable Memory Architecture

## Executive Summary

The **`pi-hermes-obsidian-memory`** package provides persistent, cross-machine portable memory, full-text session search, passive background learning, and secret scanning for the [Pi Agent Harness](https://github.com/earendil-works/pi-coding-agent). Derived from Nous Research's Hermes agent memory model and Chandra Teja's `pi-hermes-memory`, this self-created package bridges local LLM session lifecycles with cross-device Obsidian Vault synchronization.

This document details the architectural strategy, dual-tier storage engine, vault auto-detection pipeline, security sanitization filters, cross-process lock coordinator, and passive learning loops implemented in `pi-hermes-obsidian-memory`.

---

## 1. Architectural Overview & System Topology

```mermaid
flowchart TD
    subgraph Pi Agent Runtime
        Session[Pi Interactive Session] --> PromptPolicy[Memory Policy Injection]
        Session --> Tools[Memory & Session Tools]
        Session --> PassiveLoop[Passive Learning & Correction Detector]
    end

    subgraph Security & Sanitization Layer
        Tools -->|Memory Write| Scanner[Content & Secret Scanner]
        PassiveLoop -->|Auto Save| Scanner
        Scanner -->|Injection / Secret Blocked?| Reject([Reject Write])
        Scanner -->|Safe Content| LockCoord[Atomic Lock Coordinator]
    end

    subgraph Concurrency & Mutex Layer
        LockCoord -->|Acquire Lease| SQLiteLock[(locks.sqlite)]
        LockCoord -->|Incarnation Probe| ProcCheck[OS Process Check / Linux /proc / Win PS]
    end

    subgraph Dual-Tier Obsidian Storage Store
        LockCoord -->|Commit Write| MarkdownStore[Plaintext Markdown Files]
        LockCoord -->|Sync FTS5| SQLiteDB[(sessions.db WAL Mode)]

        MarkdownStore --> MemFile[MEMORY.md - Max 5k Chars]
        MarkdownStore --> UserFile[USER.md - Profile - Max 5k Chars]
        MarkdownStore --> StandFile[STANDING.md - Pinned Instructions]
        MarkdownStore --> FailFile[failures.md - Categorized Lessons]
        MarkdownStore --> SkillsFolder[skills/ - SKILL.md Files]

        SQLiteDB --> FTSIndex[FTS5 Search Tables]
    end

    subgraph Cross-Machine Sync
        MarkdownStore <--> VaultSync[Obsidian Sync / Vault Sync]
        VaultSync <--> RemoteVault[Remote Workstations & Laptops]
    end
```

---

## 2. Vault Auto-Detection & Cross-Machine Portability Architecture (`paths.ts`)

Standard AI memory solutions store data in hidden local application directories (e.g. `~/.pi/agent/`), isolating facts to a single machine. `pi-hermes-obsidian-memory` routes storage into the user's primary Obsidian Vault (`<vault>/pi/`), enabling zero-infrastructure sync across laptops, desktops, and mobile devices via Obsidian Sync, Syncthing, or Git.

### 2.1 Resolution Priority Pipeline
1. **Configured Override**: If `"memoryDir"` is specified in `hermes-memory-config.json`, it takes immediate precedence.
2. **Open Obsidian Vault**: Parses Obsidian's system config file (`obsidian.json`) to find an actively open vault.
3. **Any Configured Obsidian Vault**: If no vault is currently open, uses the first valid vault declared in `obsidian.json`.
4. **Local Fallback**: If Obsidian is not installed or no vault exists, falls back to `~/.pi/agent/pi-hermes-obsidian-memory/`.

### 2.2 Cross-Platform Config Paths (`resolveObsidianConfigPath`)
- **Windows**: `%APPDATA%/obsidian/obsidian.json`
- **macOS**: `~/Library/Application Support/obsidian/obsidian.json`
- **Linux**: `$XDG_CONFIG_HOME/obsidian/obsidian.json` (defaults to `~/.config/obsidian/obsidian.json`)

---

## 3. Dual-Tier Storage Engine: Plaintext Markdown + SQLite FTS5

Memory is structured into a hybrid dual-tier representation: human-editable Markdown files backed by an indexed SQLite search engine.

```
<obsidian-vault>/pi/
├── MEMORY.md              # Global facts, project conventions, tool quirks (max 5,000 chars)
├── USER.md                # User profile, preferences, communication habits (max 5,000 chars)
├── STANDING.md            # Pinned standing instructions injected into every session (max 2k chars)
├── failures.md            # Categorized failure memories (failure, correction, insight, convention)
├── sessions.db            # SQLite database (FTS5 session index + memory sync)
└── skills/                # Procedural how-to guides stored as native SKILL.md files
    └── <slug>/SKILL.md
```

### 3.1 Tier 1: Human-Readable Markdown Storage
- **Section Signs (`§`)**: Memory entries are delimited by `§` and timestamped.
- **Project Isolation**: Project-specific facts are maintained in dedicated sub-directories (`projects-memory/<project>/MEMORY.md`), keeping repository rules separated from global facts.
- **Standing Instructions (`STANDING.md`)**: Pinned user rules injected into **every** session prompt unconditionally. background review and auto-consolidation are explicitly forbidden from mutating `STANDING.md`.

### 3.2 Tier 2: High-Performance SQLite Search Index (`sqlite-memory-store.ts` & `db.ts`)
- **FTS5 Full-Text Search**: Ingests past session transcripts and memory entries into SQLite FTS5 virtual tables (`messages_fts`, `memories_fts`).
- **WAL Journaling**: Runs `PRAGMA journal_mode = WAL` and `PRAGMA busy_timeout = 5000` for low-latency concurrent reads and writes.
- **On-Demand Search Policy (`policy-only` mode)**: To prevent context exhaustion, full memory files are **not** dumped into initial prompts. Instead, a `<memory-policy>` instruction is injected, prompting the agent to invoke `memory_search` or `session_search` on demand.

---

## 4. Security, Sanitization & Context Fencing Pipeline (`content-scanner.ts`)

Memory stores represent an attack vector for indirect prompt injection and secret exfiltration. `pi-hermes-obsidian-memory` implements a multi-layer security gateway through `scanContent()`.

```
Incoming Memory Write ---> [Invisible Unicode Scan] 
                      ---> [Threat Pattern Scanner] 
                      ---> [Secret & Credential Scanner] 
                      ---> [Context Fencing Wrapper] ---> Commit to Disk
```

### 4.1 Threat Pattern Detection
Blocks attempts to override agent behavior or exfiltrate environment state:
- Prompt Injection: `ignore previous instructions`, `system prompt override`
- Persona Hijacking: `you are now`, `act as if you have no restrictions`
- Exfiltration Payloads: `curl ... ${KEY}`, `cat .env`, `cat ~/.ssh/authorized_keys`

### 4.2 Secret & Credential Filtering
Prevents accidental storage of credentials or tokens:
- **API Keys**: Anthropic (`sk-ant-api`), OpenRouter (`sk-or-v1-`), OpenAI (`sk-`), AWS (`AKIA...`), Stripe (`sk_live_`)
- **Tokens & Auth**: GitHub (`ghp_`, `ghu_`), Slack (`xoxb-`, `xapp-`), Notion (`ntn_`), Bearer tokens, GCP Service Accounts, Azure SAS tokens, Twilio SID/Tokens
- **SSH & Passwords**: `-----BEGIN PRIVATE KEY-----`, inline `password=`, `token=`, `secret=` assignments

### 4.3 Invisible Unicode & Context Fencing
- **Unicode Filter**: Rejects zero-width characters (`\u200b`, `\u200c`, `\ufeff`, etc.) used to mask malicious prompt injections.
- **Context Fencing**: Injected memory content is enclosed in `<memory-context>` XML tags with guard notes specifying that memory content is historical reference data and **must not** be executed as instructions.

---

## 5. Multi-Process Concurrency & Mutex Locks (`atomic-lock-coordinator.ts`)

When multiple `pi` CLI sessions or background child processes run concurrently, uncoordinated writes can corrupt Markdown files or SQLite databases. `AtomicLockCoordinator` provides cross-process synchronization.

```
Process A Request ---> [SQLite locks Table] ---> Compare-and-Swap (CAS)
                            │
               ┌────────────┴────────────┐
               ▼                         ▼
      [Lock Free / Expired]      [Lock Held by Process B]
         Acquire Lease              Check Process B Liveness
               │                         │
     Heartbeat Renewal           Alive & Same Incarnation?
     (CONSOLIDATION_HEARTBEAT)      │                 │
                                  [Yes]             [No]
                                Wait/Poll       Reclaim Lease
```

### 5.1 Process Incarnation Probing
Process ID (PID) reuse can cause lock leaks. To verify whether a lock holder is genuinely alive:
- **Linux**: Probes start time field in `/proc/<pid>/stat`.
- **macOS**: Queries process start time via `ps -o lstart= -p <pid>`.
- **Windows**: Executes PowerShell `(Get-Process -Id <pid>).StartTime.Ticks`.

### 5.2 Atomic CAS & Heartbeat Timers
- **Lease Acquisition**: Executes `BEGIN IMMEDIATE` transactions in `locks.sqlite` to write atomic lock tokens.
- **Heartbeat Renewal**: Active lock holders execute periodic `renew()` beats (`CONSOLIDATION_LOCK_HEARTBEAT_MS = 10,000ms`), renewing their timestamp so long-running operations are never misidentified as wedged processes.
- **Dead-Lock Garbage Collection**: Stale locks owned by dead PIDs are automatically garbage-collected after a 60-second grace period.

---

## 6. Passive Learning & Auto-Consolidation Engines

### 6.1 Background Review (`background-review.ts`)
Executes an automated background review loop every 10 conversation turns or 15 tool calls (`nudgeInterval` / `nudgeToolCalls`). The engine inspects recent dialogue, extracts key environment facts, conventions, and user preferences, and appends them to memory without user intervention.

### 6.2 Real-Time Correction Detector (`correction-detector.ts`)
Instead of waiting for nudge intervals, `isCorrection()` uses a two-pass filter to catch user corrections in real-time:
1. **Negative Patterns**: Suppresses false positives (e.g., "no problem", "not a big deal").
2. **Strong Patterns**: Always triggers (e.g., "wrong", "don't do that", "actually", "I told you").
3. **Weak Patterns + Directive Clauses**: Triggers if a weak indicator (e.g., "no") is followed by a directive verb clause (e.g., "use pnpm instead").

When a correction is detected, the engine instantly updates memory and appends a categorized lesson to `failures.md`.

```
User Message ---> [Negative Pattern Match?] ---> (No Action)
                         │
                         ▼ (No)
                  [Strong Pattern Match?] ---> (Trigger Immediate Save)
                         │
                         ▼ (No)
                  [Weak Pattern Match?] ---> [Contains Directive Clause?]
                                                      │
                                            ┌─────────┴─────────┐
                                            ▼ (Yes)             ▼ (No)
                                     (Trigger Save)        (No Action)
```

### 6.3 Auto-Consolidation (`auto-consolidate.ts`)
When `MEMORY.md`, `USER.md`, or project memory reaches capacity (5,000 characters), `triggerConsolidation()` triggers an LLM-driven merger:
- Consolidates redundant entries, merges overlapping facts, and removes stale observations.
- Prefers direct in-process LLM completion (`direct` transport), falling back to isolated child CLI prompts (`pi -p`) if direct completion is unavailable.

---

## 7. Excalidraw System Visual & Rendered Topology Diagram

The system architecture and sequence flow have been rendered and published to Excalidraw canvas and high-resolution PNG:
- **Rendered PNG Diagram**: [`pi-persistent-portable-memory-strategy.png`](file:///M:/Github/architecture/pi/pi-persistent-portable-memory-strategy.png)
- **Excalidraw Canvas File**: [`pi-persistent-portable-memory-strategy.excalidraw`](file:///M:/Github/architecture/pi/pi-persistent-portable-memory-strategy.excalidraw)

![Pi Hermes Obsidian Memory Architecture Visual](file:///M:/Github/architecture/pi/pi-persistent-portable-memory-strategy.png)

### Canvas Layout & Component Blueprint:
- **Blue Node (Session & Tools)**: Pi Session Interface (`memory_search`, `memory_add`, `session_search`, `skill_manage`)
- **Red Gateway (Security & Sanitization)**: Threat pattern matcher, API key / secret scanner, unicode filter, `<memory-context>` fencing
- **Amber Mutex (Atomic Lock Coordinator)**: Process incarnation probing, SQLite CAS lease locks (`locks.sqlite`), heartbeat timers
- **Purple Engine (Passive Learning & Consolidation)**: Background review (10 turns/15 calls), 2-pass correction detector, LLM auto-consolidator
- **Emerald Container (Obsidian Vault Store)**: `<vault>/pi/` Markdown storage (`MEMORY.md`, `USER.md`, `STANDING.md`, `failures.md`, `skills/`) + SQLite FTS5 database (`sessions.db` WAL mode)


---

## 8. Verification & Production Audit

`pi-hermes-obsidian-memory` has been audited and verified for production use:
- **SQL Injection Safety**: Parameterized SQL queries and quoted identifiers throughout SQLite store.
- **Atomic Disk Safety**: Same-directory temp file renames with external conflict verification.
- **Dependency Surface**: Clean production dependencies (`better-sqlite3`, `chalk`, `typebox`).
- **Zero Local Server Footprint**: Direct filesystem integration with Obsidian Vaults, eliminating REST/HTTP daemon attack surfaces.
