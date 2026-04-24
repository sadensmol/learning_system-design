# Change Watcher

> Referenced from [`plans/2026-04-23.md`](plans/2026-04-23.md) D-1 / D-6 / D-10.

## Scope

The agent's **change-detection subsystem**: how new data is spotted for
each source (files, MongoDB, SQLite), what cursors track shipped-vs-
unshipped state, and how periodic base jobs avoid overlapping
themselves.

The wire protocol for actually uploading detected changes lives in
[`pipeline-05-transport.md`](pipeline-05-transport.md). The DB-specific backup strategy
(base snapshots, fences, restore) lives in
[`databases.md`](databases.md). The **on-demand
/ first-time / post-offline** path that doesn't rely on this subsystem
at all lives in [`flow-one-shot.md`](flow-one-shot.md). This doc is
about **detection and pacing only**.

## Model

The agent is **continuous, not scheduled**: every observed change flows
through the pipeline as it arrives. There is no "backup window." The
agent maintains durable state so it always knows exactly what has
already been shipped and what is new.

Three detection mechanisms, each matched to what the source natively
supports:

| Source | Mechanism | Wake-up |
|---|---|---|
| Files | inotify / fanotify | Kernel push on filesystem events |
| MongoDB | Tailable `awaitData` cursor on `oplog.rs` | Server long-poll — blocks until new oplog entries |
| SQLite | inotify on `-wal` sidecar + ~500 ms polling fallback | Kernel hint; polling is the safety net |

Common invariant across all three:

> **detect → debounce/batch → chunk → upload → advance cursor only on ack**

This is the single rule that makes "remember where we were" and
"at-least-once, never lose data" both true without any coordinating
state beyond the agent-state DB.

## Filesystem watcher

On Linux (UmbrelOS) the watcher uses **inotify** recursively over each
resource group's data directory, falling back to **fanotify** when more
than a few thousand directories are involved (fanotify is whole-mount,
so no per-directory cost). Its job is **detecting which paths need
attention**, not enumerating or capturing files — that's the
filesystem walker's responsibility (see
[`pipeline-02-file-capture.md`](pipeline-02-file-capture.md)).

On each inotify event the watcher hands the affected path to the
walker's single-file capture entry point:

- `IN_CLOSE_WRITE` / `IN_MOVED_TO` → after a **500 ms quiet period** on
  that path (editors write temp files and rename), call
  `capture_file(path)`. The walker itself does the `(size, mtime)`
  comparison against the state DB and decides whether to re-chunk.
- `IN_DELETE` / `IN_MOVED_FROM` → walker marks the entry deleted in
  the next snapshot manifest; historical chunks stay per retention
  policy.
- `IN_Q_OVERFLOW` → inotify dropped events under load. Trigger a
  **full reconciliation scan** of the resource group via the walker's
  top-level `walk()` entry point. Rare under normal use; this scan is
  the only safety net for missed events.

On agent startup a reconciliation scan also runs for each resource
group before the live watcher takes over, so changes made while the
agent was down are still captured. The scan is just a normal
`walk()` invocation; nothing watcher-specific.

The walker's state schema (`files`, `walker_runs`), per-file capture
with stable-read, include/exclude rules, and resumability are all
covered in [`pipeline-02-file-capture.md`](pipeline-02-file-capture.md).

## MongoDB oplog tailer

The tailer opens a **tailable, awaitData** cursor on `oplog.rs`:

```js
db.oplog.rs.find({ ts: { $gt: lastCursor } })
  .addOption(DBQuery.Option.tailable)
  .addOption(DBQuery.Option.awaitData)
```

This is Mongo's native long-poll — the cursor blocks server-side until
new entries arrive (up to `maxAwaitTimeMS`), then returns them in a
single batch. No client-side polling and no wake-up timer: the tailer
wakes only when there is real work.

Cursor checkpointing piggy-backs on the shared upload pipeline. The
last oplog `ts` covered by a chunk is recorded in the chunk's manifest
entry, and the tailer's persisted cursor in the agent-state DB advances
only after that chunk's 2xx ack.

## SQLite WAL shipper

SQLite doesn't publish notifications when the WAL grows. The shipper
detects new frames with a two-tier trigger:

1. **inotify on the `-wal` sidecar** as a wake-up hint — `IN_MODIFY` on
   `app.db-wal` means the app probably just committed. This keeps the
   shipper idle when the DB is idle.
2. **Polling fallback** every ~500 ms — inotify hints can be missed on
   some filesystems, and the fallback is cheap (a single read of the
   WAL header inside the shipper's existing read transaction).

On either trigger the shipper reads the WAL header to get the current
frame count, compares against `(wal_generation, frame_offset)` in the
agent-state DB, and streams any new frames. The cursor advances only
after the chunk containing those frames is durably acked.
`wal_generation` bumps on every WAL checkpoint so frame offsets don't
collide across generations.

Net effect matches Mongo's awaitData: idle DB → no work; active DB →
shipper lags by at most one chunk's worth of latency.

## Cursors (the "last-uploaded" memory)

Every change source has a single monotonic cursor, persisted in the
agent-state DB, advanced **only after the corresponding chunk is
durably acked by the ingest API**:

| Source | Cursor |
|---|---|
| File walker | Per-file `(size, mtime, last_chunk_root)` + per-group last-scan timestamp. |
| Mongo oplog tailer | Last acked oplog `ts` (BSON timestamp) per instance. |
| SQLite WAL shipper | `(wal_generation, frame_offset)` per DB file. Generation bumps on every WAL checkpoint. |
| Snapshot committer | Last committed snapshot root hash per resource group. |

On startup the agent reads each cursor and resumes from there. Crash-
safety follows directly from the upload state machine: acked chunks
stay acked, in-flight chunks retry, and the cursor never advances past
what the server has durably stored. At-least-once delivery on the wire;
the restore side is idempotent to tolerate overlap.

## Periodic jobs & overlap prevention

Not everything is streaming. Two periodic jobs run per DB:

- **SQLite base snapshot** (`VACUUM INTO`) — default every 6 hours.
- **MongoDB base snapshot** (logical dump) — default weekly.

These fire on an internal tick, but each runs under a **single-flight
guard**: a `jobs(kind, resource_group, state, started_at)` row in the
agent-state DB. The scheduler refuses to start a new run while the
previous one is `running`, which directly answers the "it's time for
the next backup but the old one hasn't finished" case — the tick is
just skipped. A stalled job (process killed mid-run) is recovered on
next startup: the `running` row is flipped to `failed` and the job
retries on its next tick.

Three additional safeguards:

- **Skip-if-no-change.** If the source cursor hasn't advanced since the
  last successful base snapshot, skip this cycle. Avoids periodic churn
  on an idle DB.
- **Don't catch up.** If the device was offline and N cycles were
  missed, run the next one at most once; don't back-fill.
- **Yield to live traffic.** Base snapshots ride the same encrypt-and-
  upload pipeline at lower priority, so if the continuous stream is
  backed up the base waits. Back-pressure
  ([`pipeline-05-transport.md` §Back-pressure](pipeline-05-transport.md#back-pressure))
  handles this naturally; no extra mechanism needed.

There is no "next backup" cron for the continuous stream itself — the
stream either has bytes to ship or it doesn't. The overlap question
("what if the previous backup is still running?") only applies to the
periodic bases, and the single-flight guard is the complete answer.

## Agent-state DB schema (sketch)

A single SQLite file on the device, WAL-mode, owned by the agent
process only. Not itself backed up — it's derived state that can be
rebuilt from a reconciliation scan plus the server's snapshot history.

```sql
-- One row per change source; generic cursor column
CREATE TABLE cursors (
  source_id TEXT PRIMARY KEY,           -- e.g. "mongo:rocketchat", "sqlite:vaultwarden"
  cursor    BLOB NOT NULL,              -- opaque to the scheduler; source-specific encoding
  updated_at INTEGER NOT NULL
);

-- Single-flight guard for periodic base jobs
CREATE TABLE jobs (
  kind            TEXT NOT NULL,        -- "sqlite_base" | "mongo_base"
  resource_group  TEXT NOT NULL,
  state           TEXT NOT NULL,        -- "running" | "failed" | "done"
  started_at      INTEGER NOT NULL,
  finished_at     INTEGER,
  PRIMARY KEY (kind, resource_group, started_at)
);
```

The walker-owned `files` and `walker_runs` tables live in the same
agent-state DB; their definitions are in
[`pipeline-02-file-capture.md` §What it stores](pipeline-02-file-capture.md#what-it-stores)
and [§Resumability](pipeline-02-file-capture.md#resumability).

## Edge cases

- **inotify watch limit.** `fs.inotify.max_user_watches` is a real
  ceiling; the agent reports watch count as a health metric and
  switches to fanotify once a resource group exceeds a threshold.
- **Agent crash mid-chunk.** In-flight upload offsets retry from the
  server's staging state (§Resumability in `pipeline-05-transport.md`); the
  source cursor never advanced, so the same range is re-read from the
  source. At-least-once, idempotent on restore.
- **Clock skew.** All cursors are either server-issued (blob watermarks)
  or source-issued (Mongo `ts`, SQLite frame offset). Device wall-clock
  is only used for UI timestamps, never for correctness.
- **Source rename / path change.** The filesystem walker treats this as
  delete + create by default; a heuristic "rename detection" pass can
  match by `(size, content-hash)` to avoid re-uploading unchanged bytes,
  but correctness doesn't depend on it — CDC dedup makes the re-upload
  cheap regardless.

## Summary

- **Files** — inotify pushes events; debounce, dedup against prior
  chunk root, reconcile on overflow / startup.
- **MongoDB** — tailable awaitData cursor; server blocks until new
  oplog entries; cursor advances on chunk ack.
- **SQLite** — inotify hint on `-wal` plus 500 ms polling safety net;
  WAL header read tells us the new frame range.
- **Cursors** — one per source, persisted in the agent-state DB,
  advanced only after the server acks. This is the "last backup"
  memory.
- **Overlap prevention** — only matters for periodic base jobs;
  single-flight guard in the `jobs` table is the complete answer.
