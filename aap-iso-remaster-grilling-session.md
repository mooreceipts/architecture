# AAP ISO Remaster — Grilling Session Handoff

> **Purpose:** Resume a structured design grilling session from any machine with any model.
> Paste this file into your conversation context, then say: "Resume grilling from this handoff doc. I have the codebase open — verify against code where possible."

---

## What this is

A domain-modeling grilling session on the AAP ISO remaster redesign. The goal is to resolve every design decision and open question before writing playbooks. Source plan doc: `work/aap-iso-remaster-execution-plan` in Obsidian vault.

## Session rules

- Ask questions **one at a time**, wait for answer before next.
- If a fact can be **found in the codebase**, look it up — don't ask.
- **Decisions** are mine — put each to me with your recommended answer.
- Do not implement anything until all questions resolved and I confirm shared understanding.

---

## Decisions already resolved (do not re-ask these)

### 1. ISO distribution mode → Mode A (permanent)
Artifactory pull-through remote repo exists at org but availability/approval unknown. Decision: build Mode A (each slice pulls from S3 via `get_url` with checksum). If Artifactory lands later, it's a vault URL change only — no code change. The vault whitelist `url` field is the abstraction point.

### 2. No squashfs involvement
Host-specific edits touch **outer ISO filesystem only** (kickstart/config files). No squashfs unpack/repack. Scratch per slice ≈ 1.3 GB (source ISO + extracted tree + output ISO). CPU is trivial. EE does NOT need `unsquashfs`/`mksquashfs`.

### 3. Boot type → UEFI
`xorriso -boot_image any replay` preserves UEFI boot records from source ISO. No Secure Boot complications — boot binaries stay untouched, only config files modified.

### 4. Per-host data source → Infoblox WAPI
CSV hostnames come in as extra var → JT-Prime queries Infoblox WAPI REST API via `ansible.builtin.uri` → returns IPs/network info → populates `geniso` inventory group with host vars → build slices read hostvars to template config files into extracted ISO tree.

### 5. Infoblox auth → CyberArk Conjur
Credentials retrieved via alias from a custom/in-house CyberArk Conjur Ansible collection. This collection must be included in the slim EE.

---

## EE dependencies confirmed so far

| Dependency | Type | Reason |
|---|---|---|
| `xorriso` | System package | ISO remaster |
| Python 3 | Base EE | Standard |
| Internal CA bundle | Baked into EE | TLS to internal S3, Infoblox, Controller API |
| CyberArk Conjur collection (custom/in-house) | Ansible collection | Infoblox credential retrieval |

---

## Resume point: Question 8

### What to ask next (in order)

**Q8 — EE dependency audit (START HERE)**
With codebase access, audit the current EE Dockerfiles + `requirements.yml` + `requirements.txt` to find all remaining dependencies. Specifically check for:
- Dell BMC/iDRAC modules — is it `community.general.redfish_*`, a Dell vendor collection, `racadm` CLI, or raw Redfish API via `uri`?
- Any other custom in-house Ansible collections beyond the Conjur one
- Python pip packages
- Other system packages

**Q9 — What specific config files get modified in the ISO?**
Kickstart (`KS.CFG`, `KS_CUST.CFG`)? Boot menu entries? Network config templates? Other? This determines what the "customize" task block in JT-Build actually does.

**Q10 — Infoblox WAPI query details**
What WAPI object type and fields? (e.g., `GET /wapi/v2.12/record:host?name=<hostname>&_return_fields=ipv4addrs,extattrs`) How are returned fields mapped to inventory vars?

**Q11 — Dell BMC deploy mechanism**
How does the current playbook mount the ISO to BMC, set boot order, and force reboot? Redfish API calls via `uri`? `racadm` CLI? A collection module? This determines deploy-phase dependencies.

**Q12 — Deploy monitoring**
Doc flags this as unverified. How does the current flow detect OS load completion? Options: ping loop, BMC lifecycle log polling, SSH availability check, Redfish system state polling. Need to pin this down for reliable completion detection.

**Q13 — NFS output path structure**
Flat directory with `<HOSTNAME>.iso` files? Per-run subdirectory? Per-version? Matters for cleanup strategy and avoiding cross-run collisions.

**Q14 — Controller API auth**
Bearer token for AAP Controller API calls in JT-Prime — where does it come from? Another Conjur retrieval? AAP Credential Type? Built-in `tower_` vars?

**Q15 — Execution mesh sizing**
How many execution nodes? How much ephemeral disk each? Confirms whether 32 slices is realistic or needs right-sizing. At ~1.3 GB/slice, 32 co-located = ~42 GB.

**Q16 — Output ISO compliance gate**
Any signing, verification, or compliance check on the *output* ISO (not source)? Or is the source SHA gate sufficient for compliance?

**Q17 — `vcf_version` input design**
Free text extra var, or constrained survey dropdown matching whitelist keys (`vcf8`/`vcf9`)? Dropdown prevents typos and enforces only trusted versions.

**Q18 — Error handling / retry per phase**
What happens when: a single slice fails in Build? A single host fails in Deploy? Does the whole workflow abort or do healthy slices continue? Any retry logic?

**Q19 — Rollback strategy**
If build succeeds for 30/32 hosts, deploy succeeds for 25/30 — what's the recovery path for the failed hosts? Re-run whole workflow? Re-run deploy only? Manual intervention?

---

## Architecture context (for the resuming model)

Read the full plan at `work/aap-iso-remaster-execution-plan` in the Obsidian vault for complete context. Key architecture points:

- **3 Job Templates** chained in one Workflow Template: Prime (slice=1) → Build (slice=32) → Deploy (slice=32)
- **Prime** gates on SHA, wipes scratch inventory, populates from Infoblox, emits refs via `set_stats`
- **Build** pulls ISO per-slice, extracts, customizes, rebuilds with `xorriso`, pushes output to NFS
- **Deploy** mounts custom ISO to Dell BMC, boots, monitors (largely unchanged from current)
- **Inventory timing trap**: population must complete in Prime (hard predecessor) before Build slices launch, because slicing counts membership at JT launch time
- **Concurrency guard**: disable simultaneous workflow runs to prevent wipe/populate races on scratch inventory
- No `awx.awx` collection — all Controller API interaction via `ansible.builtin.uri` with bearer token
