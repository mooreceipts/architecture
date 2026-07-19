# AAP ISO Remaster Workflow — Execution Plan & Design Handoff

> **Status:** planning. This document captures decisions, open questions, and a
> step-by-step build order so a future session (or Claude Code in the repo) can
> continue without re-deriving context.

---

## 1. Goal

Redesign an existing AAP workflow that builds **host-specific custom ISOs** (VCF8
or VCF9 base) and deploys them to Dell BMC targets. Up to **32 hosts built
concurrently** via job slicing. Source ISO ≈ **650 MB**.

### Objectives driving the redesign
1. **Eliminate dual Execution Environments.** Today one EE per VCF major version,
   with the ISO + extracted tree **baked into the EE image**. Bloats images,
   doubles maintenance.
2. **Single slim EE**, pull the version-specific ISO at runtime instead of baking it.
3. **Add SHA validation** of the pulled ISO against an internal whitelist, for
   security/compliance — **kept fully internal** (no external SHA fetch; org has
   procedural overhead around outbound requests from AAP).

---

## 2. Current process (as-is)

1. **Two GitLab projects** (one per VCF major version) each build + publish a
   custom EE (Dockerfile) with the ISO baked in: ISO pulled, contents extracted
   to a local dir inside the EE, EE pushed to AAP.
2. **AAP workflow** takes a comma-delimited hostname list as an extra var; runs a
   different EE/job depending on VCF8 vs VCF9 (selection driven by which files are
   baked into the targeted EE).
3. Hostnames split → IPs gathered → host + network info added to a `geniso` group
   in a scratch inventory (**NULL_HOSTS**) used to job-slice (up to 32 concurrent).
4. **Per build slice:** use locally-baked extracted files → apply host-specific
   config edits → `xorriso` rebuild → output `<HOSTNAME>.iso` → push to a single
   **remote NFS** (required because Dell BMCs can only mount images from NFS).
5. **Deploy slice (concurrent):** mount the host's custom ISO to the BMC → set
   boot to virtual CD/DVD → force reboot → monitor OS load to completion.

---

## 3. Target architecture (to-be)

Three-phase workflow, **single slim EE** (tooling only: `xorriso`, python, S3
client — no baked ISO). ISO version becomes a runtime variable.

```
[JT-Prime]   slice=1   → gate + inventory prep + emit refs
     │ on-success (hard predecessor)
     ▼
[JT-Build]   slice=32  → pull/read ISO, customize, xorriso, push to NFS
     │ on-success
     ▼
[JT-Deploy]  slice=32  → mount ISO → BMC vCD → reboot → monitor
```

Three **separate Job Templates** chained in one Workflow Template. Slicing is a
per-JT property (`job_slice_count`), which is why Prime (unsliced) must be its own
JT from the sliced Build/Deploy JTs.

---

## 4. Key decisions made

### 4.1 SHA validation — internal, static, local comparison
- Store a **vault-encrypted whitelist in-project**, keyed by version, holding both
  URL and expected SHA:
  ```yaml
  iso_whitelist:
    vcf8: { url: "...", sha256: "abc..." }
    vcf9: { url: "...", sha256: "def..." }
  ```
- Validate with `get_url`'s `checksum: "sha256:{{ ... }}"` — comparison is **local**
  against a literal string already in context. **No external request.**
- Deliberately **avoid** `checksum: "sha256:https://.../CHECKSUMS"` — that form
  *does* fetch and is what we're avoiding.
- Tradeoff: ISO updates require a vault edit + re-encrypt. This is a feature for
  compliance — git history answers "what SHA did we trust on date X".

### 4.2 ISO distribution — the concurrency/NetApp-load question
- 650 MB × 32 ≈ **21 GB** of reads of **one immutable object**. ONTAP serves this
  from **read cache** after the first hit; real ceiling is **link egress
  bandwidth**, not S3/ONTAP struggling. ~7 s @ 25 GbE / ~17 s @ 10 GbE fully
  parallel, one-time per run.
- **NFS does NOT help the read side** — a remaster touches every byte, so
  loop-mounting the source over NFS still reads 650 MB × 32 from the filer, plus
  more metadata chatter. NFS zero-copy only helps "use in place," which does not
  apply (hosts need writable local copies to extract/modify/rebuild).

**Two shippable modes (no new infra needed):**

| | **Mode A — validate-and-emit** | **Mode B — validate-and-stage** |
|---|---|---|
| Prime node | validates SHA, emits ref via `set_stats` | pulls ONCE + validates, copies blessed ISO to shared NFS, emits path |
| Build slices | each pulls 650 MB from S3 + re-verifies checksum | each copies staged ISO from NFS (`remote_src: true`) |
| S3 origin reads | 32 | 1 |
| Herd lands on | S3 | NFS |
| Win when | S3 egress can absorb ~21 GB/run | NFS is cheaper/less contended than S3 |
| Always provides | central version/SHA compliance gate | central version/SHA compliance gate |

- **Recommended starting point: Mode A.** Simplest (no staging, no NFS
  read+write stacking), 21 GB once/run is likely a non-event on datacenter fabric,
  and Prime still earns its keep as the gate. Move to B only if S3 egress is
  measured to hurt. **Watch Mode B for read+write contention** on the same filer,
  since NFS already carries output ISOs + BMC mounts.
- **Artifactory pull-through remote repo** strictly dominates both (32 origin reads
  → 1, *and* slices pull independently with no staging). **Still the thing worth
  chasing** — but blocked pending confirmation it's available for this use.

### 4.3 Prime node concept
- First workflow node, **unsliced (slice=1)**, so pull/validate/prep happens
  **once**, not 32×.
- "Stage on prime node local disk" **does not work**: sliced jobs run in separate
  ephemeral EEs across the mesh; slice #14 can't see prime's `/tmp`. Staging must be
  a location **all EEs can independently reach** (NFS / S3 prefix).
- Cross-node data passed via **`set_stats`** (metadata only — never ISO bytes).
  Default behavior surfaces the values as **extra vars** on downstream nodes.

### 4.4 Inventory population — API path (no `awx.awx` collection)
- Org has **no `awx.awx` collection**; interact via **Controller API v2 +
  `ansible.builtin.uri`** with a **bearer token** (from Credential/vault — never
  inline creds). `validate_certs: true`; **EE must trust internal CA bundle** or
  `uri` TLS fails.
- Persist to the **real inventory** via API, **not `add_host`** (`add_host` is
  in-memory for one play only; a separate sliced JT won't see it).

### 4.5 The inventory-timing trap (critical)
- **Slicing counts inventory membership at the moment the sliced JT launches**,
  which is *before* any `add_host`/population tasks in that same job run.
- Therefore **population must fully complete in an earlier node** (Prime) that is a
  **hard on-success predecessor** of JT-Build. Otherwise slice count is computed
  against an empty/stale group.

### 4.6 Stale membership + clean slate
- Scratch inventory (NULL_HOSTS) accumulates hosts across runs → stale slices.
- For a scratch inventory, **DELETE hosts outright** (`DELETE /api/v2/hosts/{id}/`)
  each run is cleaner than disassociate — deleting removes group membership
  automatically and avoids orphan/name-collision buildup. Use disassociate
  (`POST {id, disassociate:true}` → 204) only if host objects are referenced
  elsewhere.
- **Pagination trap:** Controller API caps page size (default 25, max 200 via
  `page_size`). Naive single-GET cleanup leaves later pages alive → reintroduces the
  stale-slice bug. Probe `json.count` → compute page count, or follow `json.next`.
- **Ordering within Prime:** (1) SHA-gate first, (2) then wipe, (3) then populate,
  (4) then `set_stats`. Never wipe before the gate — a failed gate would destroy the
  inventory for nothing.

### 4.7 Concurrency guard
- Concurrent workflow runs against the same scratch inventory race catastrophically
  (run B's wipe eats run A's hosts mid-build). **Disable simultaneous runs** on the
  Workflow Template, **or** namespace the group per run (`geniso_{{ workflow_job_id }}`).
  Simplest: disable concurrent workflow runs explicitly.

### 4.8 Execution-node sizing (the real failure mode, not the download)
- 32 slices ≠ 32 machines — AAP packs slices onto available mesh capacity; multiple
  EEs can co-locate on one node.
- Per-slice working set ≈ source ISO (~650 MB) + extracted tree (~1 GB+, more with
  squashfs) + output ISO (~650 MB) ≈ **~2.3 GB scratch**. 32 co-located ≈ **~74 GB
  ephemeral disk**.
- Keep extract/modify/rebuild in **local ephemeral scratch**, never a network mount
  (fast small-file ops; slice isolation).
- Size execution-node ephemeral storage + `container_options` limits to
  ~2.3 GB/slice × max co-located slices. Clean scratch per slice (`block`/`always`).
- **squashfs recompression is the CPU driver** — confirm whether the ISO contains a
  squashfs that gets unpacked/repacked (open question below).
- Prefer **`xorriso`** over `genisoimage` — preserves hybrid BIOS/UEFI, El Torito,
  GPT on remaster.
- Right-size `job_slice_count` to **execution capacity**, not blindly to 32.
  `job_slice_count` is a *max* parallelism, not a requirement.

---

## 5. Ordered execution plan (build order)

### Phase 0 — Confirm load-bearing unknowns (blocks final mode choice)
- [ ] **Artifactory pull-through remote repo available?** If yes, it supersedes
      Modes A/B for ISO distribution.
- [ ] Will ISOs live in **NetApp S3**? Confirm bucket + endpoint.
- [ ] Is there a shared **staging spot** slices can read (needed only for Mode B)?
- [ ] Execution mesh: **node count + per-node disk/CPU** → confirms 32 is realistic.
- [ ] Does the ISO contain a **squashfs** that is unpacked/repacked? (CPU/scratch sizing)
- [ ] Boot type to preserve: **BIOS / UEFI / hybrid**, Secure Boot? (xorriso flags)
- [ ] Source of per-host modification data (inventory vars / survey / CMDB API)?
- [ ] Output ISO destination confirmed = NFS (BMC constraint) — yes, per current flow.
- [ ] Any signing/verification/compliance gate on the **output** ISO?

### Phase 1 — Slim EE
- [ ] New Dockerfile: tooling only (`xorriso`, python, S3/HTTP client, internal CA
      bundle). **No baked ISO, no extracted tree.**
- [ ] Publish single EE, register in AAP. Retire the two per-version EEs after
      cutover.

### Phase 2 — Vault whitelist
- [ ] Create `vault/iso_whitelist.yml` keyed `vcf8`/`vcf9` with `url` + `sha256`.
- [ ] Wire ansible-vault decryption into the JTs (Credential).

### Phase 3 — JT-Prime (slice=1, target localhost/control)
Order inside the play is mandatory:
1. [ ] **Assert** selected `vcf_version` is in `iso_whitelist` (fail fast).
2. [ ] **Wipe** scratch inventory (paginated GET → DELETE hosts). *After* the gate.
3. [ ] **Populate** this run's hosts: split CSV → resolve IPs → create + associate
       to `geniso` (POST to `/groups/{group_id}/hosts/`; `variables` as a **JSON
       string** carrying `ansible_host`/`bmc_ip`/network info).
4. [ ] **Mode B only:** pull once + `checksum` gate → copy blessed ISO to NFS.
5. [ ] `set_stats` → emit `iso_ref`, `iso_sha`, `iso_version` (Mode A) or
       `staged_iso`, `iso_version` (Mode B).

### Phase 4 — JT-Build (slice=32, inventory=NULL_HOSTS)
- [ ] **Mode A:** `get_url` pull from `iso_ref` with `checksum: "sha256:{{ iso_sha }}"`
      into local scratch. **Mode B:** `copy remote_src: true` from `staged_iso`.
- [ ] Extract → apply host-specific config edits (driven by `inventory_hostname` +
      hostvars) → `xorriso` rebuild → `<HOSTNAME>.iso`.
- [ ] Push output to NFS (`remote_src: true`).
- [ ] `block`/`always`: remove scratch workdir (prevent ephemeral-disk leak).

### Phase 5 — JT-Deploy (slice=32, inventory=NULL_HOSTS)
- [ ] Mount host's ISO to Dell BMC → set boot to virtual CD/DVD → force reboot →
      monitor OS load to completion. (Largely unchanged from current flow.)

### Phase 6 — Workflow assembly + guards
- [ ] Chain Prime → Build → Deploy with **on-success (hard predecessor)** edges.
- [ ] **Disable simultaneous runs** on the Workflow Template (or namespace group per
      run) to prevent wipe/populate races.
- [ ] Ensure `vcf_version` propagates (workflow-level extra var / survey; Prime
      accepts prompt-on-launch).
- [ ] Validate slice count matches this run's host list (mismatch ⇒ population didn't
      commit before slicing, or stale members not cleared).

---

## 6. API cheat-sheet (Controller v2 via `uri`, bearer token)

- Auth header: `Authorization: Bearer {{ controller_token }}`, `Content-Type: application/json`.
- Group lookup: `GET /api/v2/inventories/{inv_id}/groups/?name={group}`
- Group create: `POST /api/v2/inventories/{inv_id}/groups/` body `{name}` → 201
- Host create + associate to group (one call):
  `POST /api/v2/groups/{group_id}/hosts/` body `{name, variables: <JSON string>}` → 201
- Associate existing host: `POST /api/v2/groups/{group_id}/hosts/` body `{id: <host_id>}`
- Disassociate: `POST /api/v2/groups/{group_id}/hosts/` body `{id, disassociate: true}` → 204
- Delete host: `DELETE /api/v2/hosts/{id}/` → 204
- Pagination: `?page_size=200&page=N`; probe `json.count`, or follow `json.next` (null = done).
- `variables` field must be a **serialized JSON string** (`| to_json`), not a dict.

---

## 7. Open risks / to verify next session

- **Artifactory availability** — the single highest-leverage unknown; flips the whole
  distribution design.
- **squashfs presence** — dominates CPU + scratch sizing; unverified.
- **Deploy-node monitor mechanism** — current flow "not sure if ping vs BMC-log
  confirmation." Needs to be pinned down for reliable completion detection.
- **Existing-host association edge case** — if a host already exists in the inventory
  but not in `geniso`, POST-to-group behavior (201 vs 400) needs handling; moot if
  every run starts from a full wipe.
- **EE internal CA trust** — `uri` + `get_url` over internal TLS will fail if the slim
  EE doesn't bundle the internal CA.

---

## 8. Next artifacts to produce
- Full runnable playbooks: Prime (wipe/populate/gate/set_stats), Build
  (extract/customize/xorriso with correct boot-preserving flags), Deploy.
- CSV-split + host-create task block (Phase 3 step 3), fully written.
- Optional: ADR for architecture reviewers (dual-EE → single-EE + runtime pull,
  with the Mode A/B tradeoff table).
