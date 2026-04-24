# Umbrel Home — Backup & Restore

System design exercise: efficient, end-to-end encrypted backup and restore for
an **Umbrel Home** personal server, covering photos, videos, notes, and the
embedded databases (MongoDB + SQLite) that Umbrel apps use to store application
state.

## Why Umbrel Home

[Umbrel Home](https://umbrel.com/umbrel-home) is a real, shipping ARM-based
personal server (~4 GB RAM, NVMe SSD, quad-core) that runs self-hosted apps
as Docker containers. Different apps ship different embedded databases:

- **Immich / PhotoPrism / Plex** — photo + video libraries
- **Trilium / Memos / Joplin Server** — notes
- **Apps that use MongoDB** — e.g., Rocket.Chat, some self-hosted tools
- **Apps that use SQLite** — most lightweight apps (Memos, Uptime Kuma, Vaultwarden, etc.)

This makes Umbrel Home a concrete, realistic target for a design discussion:
constrained edge hardware, mixed data (bulk media + multiple embedded DBs),
a single owner with multiple consumer devices, and a real need for off-device
backup since the device itself is the only copy.

**Closest real-world analogs for techniques:**
iCloud Photos + iPhone Backup (closed), Immich (open source), restic (storage
engine), Litestream (SQLite replication).

## Reading order

1. [`docs/requirements.md`](docs/requirements.md) — actors, goals, constraints, hotspots
2. [`docs/plans/2026-04-23.md`](docs/plans/2026-04-23.md) — main design document (the whole system)
3. **Pipeline stages** (elaborate each stage of the continuous flow in the main plan, in order):
   - [`docs/pipeline-01-detection.md`](docs/pipeline-01-detection.md) — inotify, Mongo tailers, SQLite WAL polling, cursors, overlap prevention
   - [`docs/pipeline-02-file-capture.md`](docs/pipeline-02-file-capture.md) — filesystem walker, stable-read, state DB
   - [`docs/pipeline-03-chunking.md`](docs/pipeline-03-chunking.md) — content-defined chunking + dedup
   - [`docs/pipeline-04-encryption.md`](docs/pipeline-04-encryption.md) — envelope encryption, keys, recovery
   - [`docs/pipeline-05-transport.md`](docs/pipeline-05-transport.md) — resumable uploads, back-pressure, device policy
   - [`docs/pipeline-06-storage.md`](docs/pipeline-06-storage.md) — snapshots, retention, garbage collection
4. **Cross-cutting concerns**:
   - [`docs/databases.md`](docs/databases.md) — MongoDB + SQLite specialized capture & per-app restore
   - [`docs/multi-device.md`](docs/multi-device.md) — pairing, identity, revocation
5. **Additional operational flows** (not in the main plan):
   - [`docs/flow-one-shot.md`](docs/flow-one-shot.md) — on-demand backup *and* whole-device restore
6. **Standalone cross-project references**:
   - [`../cdc-guide.md`](../cdc-guide.md) — CDC algorithms deep dive (Rabin / Buzhash / FastCDC), ARM throughput, worked examples
   - [`../merkle-trees-guide.md`](../merkle-trees-guide.md) — Merkle trees from first principles, proofs, real-world uses, how this system stacks three levels of them
