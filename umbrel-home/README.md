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
2. [`docs/plans/2026-04-23.md`](docs/plans/2026-04-23.md) — main design document
3. Supporting docs (linked from the main plan):
   - [`docs/chunking-dedup.md`](docs/chunking-dedup.md)
4. Standalone cross-project reference: [`../cdc-guide.md`](../cdc-guide.md) — content-defined chunking algorithms, per-data-type behavior, pipeline.
   - [`docs/encryption.md`](docs/encryption.md)
   - [`docs/versioning-retention.md`](docs/versioning-retention.md)
   - [`docs/sync-protocol.md`](docs/sync-protocol.md)
   - [`docs/multi-device-access.md`](docs/multi-device-access.md)
   - [`docs/local-databases-backup.md`](docs/local-databases-backup.md)
