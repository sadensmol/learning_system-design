# Requirements — Umbrel Home Backup & Restore

## Summary

Design an efficient, end-to-end encrypted backup and restore system for an
[Umbrel Home](https://umbrel.com/umbrel-home) personal server that stores
photos, videos, notes, and a mix of embedded application databases
(**MongoDB** and **SQLite**) used by self-hosted apps running on the device.
Backups flow to a cloud backend (IaaS-based). Users can also read their data
from secondary devices (phone, web) without the server seeing plaintext.

Framing: **system design learning / interview exercise**, but grounded in realistic
constraints so the design could plausibly be implemented.

## Business goals

- **Durability** — a device loss/theft/failure must not cause data loss.
- **Privacy by default** — server operator cannot read user content.
- **Efficiency on the edge** — the device is the bottleneck; minimize its CPU,
  RAM, disk, bandwidth, and power cost.
- **Recoverability** — users can restore a replacement device, recover individual
  files, roll back to a past point in time, and undo deletes within a window.
- **Cost predictability** at 10K-user scale — storage and egress dominate cost;
  design must keep both bounded.

## Scale targets (for reasoning, not a contract)

| Dimension | Target |
|---|---|
| Users | 100s → 10Ks |
| Devices per user | 1 primary + 1–3 read-only secondaries |
| Data per user | 50 GB avg, 1 TB p99 (rough order of magnitude) |
| New data per user per day | 100 MB avg, 10 GB p99 (burst on events) |
| Read fan-out per user | Low — data is mostly insurance + occasional browsing |

These are order-of-magnitude estimates, not hard SLAs. A real product would
measure first.

## Actors

- **User** — owns the device and data, holds the master secret.
- **Primary device (source)** — **Umbrel Home**: quad-core ARM, ~4 GB RAM,
  NVMe SSD, always-on Wi-Fi/Ethernet. Runs UmbrelOS (Debian-based) with
  self-hosted apps as Docker containers. Captures photos/videos/notes via
  installed apps (e.g., Immich, PhotoPrism, Memos, Trilium) and is the source
  of truth for new content. Hosts a **mix of embedded databases**:
  - One or more **MongoDB** instances used by apps like Rocket.Chat.
  - Many **SQLite** databases — the default store for most lightweight
    Umbrel apps (Memos, Vaultwarden, Uptime Kuma, etc.). Usually one DB
    file per app, often with a WAL sidecar.
  Both classes of DB must be backed up in lockstep with the blob data they
  reference (media, attachments).
- **Secondary devices (readers)** — phone, laptop, web browser. Authenticated
  via the same user account; receive key material through a secure device-pairing
  flow.
- **Cloud backend (zero-knowledge)** — IaaS-hosted services and object storage.
  Sees only ciphertext blobs and encrypted metadata.
- **Operator** — runs the cloud backend. Must never be able to read user content,
  even with full database access.

## Functional requirements

### Backup (device → cloud)

- FR-1. Continuously back up new and changed files from the primary device.
- FR-2. Transfer only what's new (incremental) and only what's unique
  (deduplicated) across the user's own data.
- FR-3. Resume interrupted uploads without re-sending completed chunks.
- FR-4. Survive device reboot, network loss, and long offline periods.
- FR-5. Respect device policy: pause on battery (if applicable), low disk,
  thermal throttling; scheduled quiet hours.

### Restore

- FR-6. **Full device restore** — rebuild a fresh device's state from cloud.
- FR-7. **Selective restore** — restore individual files or folders.
- FR-8. **Time-travel restore** — restore any file or the full state as of any
  past snapshot.
- FR-9. **Trash / deletion recovery** — restore items deleted within a configurable
  retention window, even if no explicit snapshot covers them.

### Multi-device read access

- FR-10. Secondary devices can list, stream, and download any backed-up item.
- FR-11. New secondary devices are enrolled through a user-driven pairing flow
  that transfers key material without the server learning it.
- FR-12. Revoking a secondary device invalidates its future access.

### Versioning & retention

- FR-13. Every upload creates an immutable version. Retention policy decides
  when old versions are garbage-collected.
- FR-14. Users configure retention tiers (e.g., keep all for 30 days, daily for
  90 days, weekly forever).
- FR-15. Explicit deletes land in trash first (soft delete) with a retention
  window before hard delete.

### Notes (small structured data)

- FR-16. Notes sync with lower latency than media and preserve edit history.
- FR-17. Concurrent edits from multiple devices must converge without silent
  data loss. (Hotspot: exact conflict-resolution model — see below.)

### Embedded databases (MongoDB + SQLite)

- FR-18. All local databases — MongoDB instances **and** SQLite files — are
  backed up on a continuous or near-continuous basis, not just nightly dumps,
  so that DB loss window matches blob loss window.
- FR-19. DB backup and blob backup must be **point-in-time consistent** on
  restore: restored DB rows never reference a blob that wasn't also restored,
  and vice-versa. (Hotspot H-11 picks the mechanisms per DB type.)
- FR-20. DB restore can run independently of blob restore so users see a
  working library quickly while large media re-downloads in the background.
- FR-21. DB backup is subject to the same zero-knowledge property: the server
  sees only ciphertext.
- FR-22. Each app's DB is backed up **independently** so that restoring one
  app's data does not require restoring all others. A single misbehaving app
  must not block backup of the rest.
- FR-23. SQLite backup must be consistent against an actively-writing app
  without requiring the app to be stopped (so `cp file.db` is insufficient —
  it races with WAL checkpoints).

## Non-functional requirements

- **Privacy** — end-to-end encrypted. Server sees only ciphertext + opaque IDs.
  Metadata leakage (sizes, timing, counts) must be acknowledged and bounded.
- **Durability** — target 11 nines for stored objects (inherit from object
  storage), with redundancy across availability zones.
- **Availability** — backup path tolerates backend unavailability via local
  queue + retry. Restore path targets 99.9% availability.
- **Device budget** — agent runs within strict RAM, CPU, and disk overhead
  budgets. Exact numbers are a hotspot until the device is chosen concretely,
  but the design must not assume "plenty of resources."
- **Bandwidth efficiency** — upload bytes ≤ (1 + small constant) × unique new
  plaintext bytes. Repeats and unchanged files cost near zero.
- **Cost** — backend cost per user at 10K scale must be dominated by storage,
  not compute or egress, and must be roughly linear in data volume.
- **Security properties**
  - Confidentiality against the server operator.
  - Integrity — tampered ciphertext is detected on read.
  - Authenticity — only authorized devices of the owner can write/read.
  - Forward secrecy on device revocation where feasible.

## Out of scope (for this exercise)

- Sharing content across different users' accounts.
- Collaborative editing with access control beyond a single user's devices.
- Server-side ML features (search, face clustering) — impossible under strict
  zero-knowledge; could be added later as client-side ML only.
- Full-text search over note content server-side (likewise).
- Regulatory/legal-hold workflows beyond basic retention policy.
- Billing, quotas, and account recovery UX.

## Constraints

- Primary device is Umbrel Home **as the reference** for a consumer
  personal-server class device: quad-core ARM, ~4 GB RAM, NVMe SSD,
  always-on Wi-Fi/Ethernet. Other devices in the same class — CasaOS,
  Yunohost, Runtipi, Home Assistant Green, Synology BeeStation — share the
  same constraints. Design decisions should be justified for this class, not
  for UmbrelOS specifically.
- The backup agent runs as a containerized service on the device's
  personal-server OS and competes for RAM/CPU with user apps. It must not
  starve them.
- It reads other apps' data volumes through the host OS's app-data
  directory convention, so no cooperation from individual app authors is
  required.
- Backend is cloud IaaS. Managed object storage is available; managed
  databases are available. "Exotic" infra (custom storage engines) is out.
- Zero-knowledge encryption is hard-required; nothing in the design may depend
  on the server seeing plaintext.

## Hotspots (decisions deferred — not assumed)

The following decisions are explicitly **not** made at requirements time and
should be resolved in the plan or supporting docs with trade-offs.

- **H-1. Cross-user deduplication.** Under E2E, cross-user dedup requires
  convergent encryption, which leaks equality of plaintexts. Choose: per-user
  dedup only (safer) vs. convergent (more efficient but a known leak).
- **H-2. Chunking algorithm.** Fixed-size vs. content-defined (FastCDC /
  Rabin). CDC is more bandwidth-efficient for mutated files but costs more CPU.
- **H-3. Notes conflict model.** Last-writer-wins vs. CRDT vs. operational
  transform. Trade-off: implementation complexity vs. correctness under
  concurrent edits.
- **H-4. Key derivation + recovery.** How the master key is derived and how
  users recover after losing all devices. Options: recovery phrase
  (BIP39-style), social recovery, server-side HSM-backed escrow, none.
- **H-5. Metadata encryption granularity.** File names, sizes, timestamps —
  fully encrypted, partially encrypted, or plaintext? Affects search and server
  features. Leakage vs. usability trade-off.
- **H-6. Versioning model.** Per-file version chain vs. whole-repo snapshots
  (restic-style) vs. hybrid. Affects storage cost and restore semantics.
- **H-7. Storage tiering.** Hot vs. infrequent-access vs. cold storage for old
  versions. Retrieval latency vs. storage cost.
- **H-8. Device budget numbers.** Concrete RAM/CPU/disk caps for the agent
  once the device model is fixed.
- **H-9. Transport framing.** HTTPS + resumable chunked uploads (tus.io-style)
  vs. custom gRPC streaming. Interop vs. efficiency.
- **H-10. Notes storage location.** Are note contents stored inside MongoDB,
  as files alongside media, or both (index in Mongo, content as files)?
  Affects whether notes ride the blob pipeline or the DB pipeline.
- **H-11. Local DB backup mechanisms.** Two DB families, different
  approaches:
  - **MongoDB:** periodic `mongodump` vs. oplog tailing vs. WAL-style logical
    replication. Trade-off is RPO vs. implementation cost.
  - **SQLite:** `VACUUM INTO` snapshots vs. the online backup API vs. WAL
    shipping (Litestream-style continuous replication). Trade-off is how
    painful the "DB is being written right now" case is.
  Decision must also pick the fence mechanism that keeps DB and blob restores
  point-in-time consistent with each other — conceptually one fence, applied
  through each DB type's native change log.
- **H-12. App-DB discovery.** How the backup agent discovers which DBs exist
  to back up. Options: read the host OS's app manifest when one exists
  (declarative), auto-scan app-data directories for `*.db` / `*.sqlite` /
  Mongo data dirs (heuristic), or require apps to opt in via a small
  sidecar config. Trade-off: zero-config vs. correctness (won't silently
  miss a DB).
