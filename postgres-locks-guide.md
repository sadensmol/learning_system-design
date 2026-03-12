# PostgreSQL Locks: Complete Guide

> A beginner-friendly guide to PostgreSQL locks - what they are, when they happen, and how to deal with them.

---

## Table of Contents

1. [What Are Locks?](#1-what-are-locks)
2. [Lock Types Overview](#2-lock-types-overview)
3. [Table-Level Locks](#3-table-level-locks)
4. [Row-Level Locks](#4-row-level-locks)
5. [Advisory Locks](#5-advisory-locks)
6. [Deadlocks](#6-deadlocks)
7. [Lock Monitoring & Troubleshooting](#7-lock-monitoring--troubleshooting)
8. [Common Lock Problems & Solutions](#8-common-lock-problems--solutions)
9. [Lock-Safe Migration Patterns](#9-lock-safe-migration-patterns)
10. [Quick Reference Cheat Sheet](#10-quick-reference-cheat-sheet)

---

## 1. What Are Locks?

**A lock is like a "Do Not Disturb" sign on a hotel room door.**

Imagine a shared Google Doc:
- If two people type in the **same sentence** at the same time, the result is a mess
- Locks prevent that mess by making people **take turns** when they touch the same data

### Why Do We Need Locks?

Think of a bank account with $100:

```
WITHOUT LOCKS (disaster):
┌──────────────────────────────────────────────────────┐
│  Person A: reads balance = $100                      │
│  Person B: reads balance = $100                      │
│  Person A: withdraws $80 → writes balance = $20     │
│  Person B: withdraws $70 → writes balance = $30     │  ← WRONG!
│                                                      │
│  Both withdrew $150 from a $100 account! 💥          │
└──────────────────────────────────────────────────────┘

WITH LOCKS (safe):
┌──────────────────────────────────────────────────────┐
│  Person A: LOCKS the account 🔒                      │
│  Person A: reads balance = $100                      │
│  Person B: tries to read → WAITS... ⏳               │
│  Person A: withdraws $80 → writes balance = $20     │
│  Person A: UNLOCKS the account 🔓                    │
│  Person B: reads balance = $20                       │
│  Person B: tries to withdraw $70 → DENIED (only $20)│
└──────────────────────────────────────────────────────┘
```

### The Golden Rule

> PostgreSQL acquires locks **automatically**. You rarely create them manually.
> Every `SELECT`, `UPDATE`, `INSERT`, `DELETE`, `ALTER TABLE` takes some kind of lock.

---

## 2. Lock Types Overview

PostgreSQL has **three families** of locks:

```
┌─────────────────────────────────────────────────────────┐
│                   PostgreSQL Locks                       │
├──────────────────┬──────────────────┬───────────────────┤
│  TABLE-LEVEL     │  ROW-LEVEL       │  ADVISORY         │
│  (whole table)   │  (single rows)   │  (app-defined)    │
├──────────────────┼──────────────────┼───────────────────┤
│ 8 lock modes     │ 4 lock modes     │ session or txn    │
│ heaviest impact  │ most common      │ user-controlled   │
│ DDL operations   │ DML operations   │ custom logic      │
└──────────────────┴──────────────────┴───────────────────┘

DDL = Data Definition Language  (CREATE, ALTER, DROP)
DML = Data Manipulation Language (SELECT, INSERT, UPDATE, DELETE)
```

### The Supermarket Analogy

Think of your database as a **supermarket**:

| Lock Family | Supermarket Analogy |
|---|---|
| **Table-level locks** | Closing an entire aisle for restocking |
| **Row-level locks** | Putting your hand on a specific item so nobody else grabs it |
| **Advisory locks** | A "reserved" sign you place wherever you want |

---

## 3. Table-Level Locks

Table-level locks affect the **entire table**. They range from very permissive to completely exclusive.

### The 8 Table Lock Modes (from weakest to strongest)

```
WEAKEST (allows most concurrency)
  │
  ▼
  1. ACCESS SHARE              ← SELECT
  2. ROW SHARE                 ← SELECT FOR UPDATE/SHARE
  3. ROW EXCLUSIVE             ← INSERT/UPDATE/DELETE
  4. SHARE UPDATE EXCLUSIVE    ← VACUUM, ANALYZE, some ALTER TABLE
  5. SHARE                     ← CREATE INDEX (non-concurrent)
  6. SHARE ROW EXCLUSIVE       ← CREATE TRIGGER, some ALTER TABLE
  7. EXCLUSIVE                 ← REFRESH MATERIALIZED VIEW CONCURRENTLY
  8. ACCESS EXCLUSIVE          ← DROP TABLE, ALTER TABLE (most), TRUNCATE, VACUUM FULL
  │
  ▼
STRONGEST (blocks everything)
```

### What Triggers Each Lock?

| Lock Mode | Triggered By | Real-World Analogy |
|---|---|---|
| **ACCESS SHARE** | `SELECT` | Looking at items on a shelf (anyone can look) |
| **ROW SHARE** | `SELECT FOR UPDATE`, `SELECT FOR SHARE` | Picking up an item to inspect it |
| **ROW EXCLUSIVE** | `INSERT`, `UPDATE`, `DELETE` | Putting items in your cart |
| **SHARE UPDATE EXCLUSIVE** | `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY` | Store employee counting inventory (shoppers can still shop) |
| **SHARE** | `CREATE INDEX` (non-concurrent) | Roping off aisle to build a new shelf (can still look, can't add items) |
| **SHARE ROW EXCLUSIVE** | `CREATE TRIGGER`, `ALTER TABLE` (some forms) | Rearranging an aisle (can look, can't shop) |
| **EXCLUSIVE** | `REFRESH MATERIALIZED VIEW CONCURRENTLY` | Deep cleaning aisle (very limited access) |
| **ACCESS EXCLUSIVE** | `DROP TABLE`, `TRUNCATE`, `ALTER TABLE` (most), `VACUUM FULL` | **Aisle completely closed** - nobody gets in |

### The Conflict Matrix (Who Blocks Whom?)

Read this table as: **"Does lock in ROW conflict with lock in COLUMN?"**

`X` = **CONFLICTS** (must wait), blank = compatible (can run together)

```
                        Requested Lock Mode
                  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
                  │ AS  │ RS  │ RE  │ SUE │  S  │ SRE │  E  │ AE  │
Existing    ┌─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
Lock        │ AS  │     │     │     │     │     │     │     │  X  │
Mode        │ RS  │     │     │     │     │     │     │  X  │  X  │
            │ RE  │     │     │     │     │  X  │  X  │  X  │  X  │
            │ SUE │     │     │     │  X  │  X  │  X  │  X  │  X  │
            │  S  │     │     │  X  │  X  │     │  X  │  X  │  X  │
            │ SRE │     │     │  X  │  X  │  X  │  X  │  X  │  X  │
            │  E  │     │  X  │  X  │  X  │  X  │  X  │  X  │  X  │
            │ AE  │  X  │  X  │  X  │  X  │  X  │  X  │  X  │  X  │
            └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘

Legend:
AS  = ACCESS SHARE              SUE = SHARE UPDATE EXCLUSIVE
RS  = ROW SHARE                 S   = SHARE
RE  = ROW EXCLUSIVE             SRE = SHARE ROW EXCLUSIVE
                                E   = EXCLUSIVE
                                AE  = ACCESS EXCLUSIVE
```

### Key Takeaways from the Matrix

```
┌──────────────────────────────────────────────────────────────────────┐
│  1. SELECT never blocks SELECT  (AS + AS = OK)                      │
│  2. SELECT never blocks INSERT/UPDATE/DELETE  (AS + RE = OK)        │
│  3. INSERT/UPDATE/DELETE never block each other at TABLE level       │
│     (RE + RE = OK) — row locks handle conflicts                     │
│  4. ACCESS EXCLUSIVE blocks EVERYTHING — even SELECT!               │
│  5. DDL operations are the most dangerous for blocking              │
└──────────────────────────────────────────────────────────────────────┘
```

### Real-World Example: Why ALTER TABLE Is Scary

**What's happening:** You want to add a column to a busy table. Your `ALTER TABLE` needs an ACCESS EXCLUSIVE lock (the strongest lock - blocks everything). But a slow report query is already running. Your ALTER waits in line. Now every new query - even tiny fast SELECTs - queues up behind your ALTER. One waiting migration creates a traffic jam for the entire table.

**When does this occur?** Every time you run DDL (ALTER TABLE, DROP, TRUNCATE) on a table that has active queries. The busier the table, the worse the pileup.

**How to avoid:** Always use `SET lock_timeout = '3s'` before DDL. If it can't get the lock in 3 seconds, it fails fast and you retry later instead of creating a queue.

```sql
-- Session 1: Long-running query (holds ACCESS SHARE lock)
SELECT * FROM orders WHERE created_at > '2024-01-01';  -- takes 5 minutes

-- Session 2: You run a migration (needs ACCESS EXCLUSIVE lock)
ALTER TABLE orders ADD COLUMN status VARCHAR(20);
-- ⏳ BLOCKED! Waiting for Session 1 to finish...

-- Session 3: Simple query (needs ACCESS SHARE lock)
SELECT * FROM orders WHERE id = 1;
-- ⏳ BLOCKED! Waiting behind Session 2!
-- Even though Session 1 and Session 3 are compatible,
-- Session 3 queues BEHIND Session 2
```

This is the **lock queue problem**:

```
┌──────────────────────────────────────────────────────────────┐
│                       Lock Queue                             │
│                                                              │
│  Running:  Session 1  [SELECT - ACCESS SHARE]     🟢        │
│  Waiting:  Session 2  [ALTER  - ACCESS EXCLUSIVE] 🔴 ←queue │
│  Waiting:  Session 3  [SELECT - ACCESS SHARE]     🔴 ←queue │
│  Waiting:  Session 4  [INSERT - ROW EXCLUSIVE]    🔴 ←queue │
│  Waiting:  Session 5  [SELECT - ACCESS SHARE]     🔴 ←queue │
│                                                              │
│  One ALTER TABLE can cause a chain reaction! ⚠️               │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Row-Level Locks

Row-level locks only affect **specific rows**, not the whole table. This is how PostgreSQL allows many users to work on the same table simultaneously.

### The 4 Row Lock Modes

| Lock Mode | Triggered By | What It Means |
|---|---|---|
| **FOR KEY SHARE** | Foreign key checks | "I'm checking if this row exists (for FK), don't delete it" |
| **FOR SHARE** | `SELECT ... FOR SHARE` | "I'm reading this row, don't change it" |
| **FOR NO KEY UPDATE** | `UPDATE` (not touching key columns) | "I'm changing non-key columns, but keys stay the same" |
| **FOR UPDATE** | `SELECT ... FOR UPDATE`, `UPDATE` (touching keys), `DELETE` | "I own this row right now, hands off!" |

### Row Lock Conflict Matrix

```
                          Requested Lock
                  ┌──────────┬──────────┬──────────┬──────────┐
                  │ FOR KEY  │   FOR    │ FOR NO   │   FOR    │
                  │  SHARE   │  SHARE   │ KEY UPD  │  UPDATE  │
Existing  ┌───────┼──────────┼──────────┼──────────┼──────────┤
Lock      │ FKS   │          │          │          │    X     │
          │ FS    │          │          │    X     │    X     │
          │ FNKU  │          │    X     │    X     │    X     │
          │ FU    │    X     │    X     │    X     │    X     │
          └───────┴──────────┴──────────┴──────────┴──────────┘

Legend:
FKS  = FOR KEY SHARE        FNKU = FOR NO KEY UPDATE
FS   = FOR SHARE            FU   = FOR UPDATE
```

### Real Examples

**Example 1: Two people updating DIFFERENT rows (no conflict)**

**What's happening:** Two users edit different rows in the same table at the same time. PostgreSQL locks each row independently, so they don't interfere at all. This is the most common case - row-level locking is what makes PostgreSQL fast for concurrent workloads.

**When does this occur?** Almost all the time in normal CRUD operations. This is the happy path.

```sql
-- Session 1                          -- Session 2
UPDATE users SET name = 'Alice'       UPDATE users SET name = 'Charlie'
WHERE id = 1;                         WHERE id = 2;
-- ✅ Both succeed immediately!       -- ✅ No conflict, different rows!
```

**Example 2: Two people updating the SAME row (conflict!)**

**What's happening:** Two users try to change the exact same row. The first one wins and locks the row. The second one has to sit and wait until the first one commits or rolls back. After that, the second one proceeds - but its change will overwrite the first one's (last-writer-wins).

**When does this occur?** Whenever two requests touch the same row - e.g., two people editing the same user profile, two API calls updating the same order, or a batch job colliding with a live user request.

**How to avoid:** You can't fully avoid this (it's correct behavior), but you can minimize wait time by keeping transactions short. If you need to detect overwrites, use optimistic locking with a `version` column.

```sql
-- Session 1                          -- Session 2
BEGIN;                                BEGIN;
UPDATE users SET name = 'Alice'       UPDATE users SET name = 'Bob'
WHERE id = 1;                         WHERE id = 1;
-- ✅ Succeeds, holds row lock        -- ⏳ BLOCKED! Waits for Session 1...
COMMIT;                               -- ✅ Now succeeds (overwrites to 'Bob')
                                      COMMIT;
```

**Example 3: SELECT FOR UPDATE (explicit row locking)**

**What's happening:** You need to read a value, make a decision based on it, then update. Without locking, another session can change the value between your read and your write (race condition). `SELECT FOR UPDATE` locks the row when you read it, so nobody can change it before you do.

**When does this occur?** Any "check-then-act" pattern: booking a seat (check available, then book), processing a payment (check balance, then deduct), claiming a task from a queue.

**How to avoid the race condition:** Always use `FOR UPDATE` when your UPDATE depends on a value you just read. Think of it as: "I'm about to change this row, so lock it NOW while I'm reading it."

```sql
-- The classic "check-then-act" pattern (e.g., booking a seat)

-- WRONG: race condition!
BEGIN;
SELECT available FROM seats WHERE id = 42;        -- reads: true
-- another session could change it RIGHT HERE!
UPDATE seats SET available = false WHERE id = 42;
COMMIT;

-- RIGHT: lock the row first!
BEGIN;
SELECT available FROM seats WHERE id = 42 FOR UPDATE;  -- locks the row
-- now we're safe, nobody else can touch this row
UPDATE seats SET available = false WHERE id = 42;
COMMIT;
```

**Example 4: FOR SHARE (shared read lock)**

**What's happening:** You want to make sure a parent row (like an order) doesn't get deleted or modified while you're inserting a child row (like order items). `FOR SHARE` says "I'm reading this, and I need it to stay exactly as it is until I'm done." Multiple sessions can hold FOR SHARE on the same row (they're all just reading), but nobody can UPDATE or DELETE it.

**When does this occur?** When you need to guarantee referential integrity beyond what foreign keys provide - e.g., inserting child records while ensuring the parent won't change, or reading config that must stay consistent during a multi-step operation.

```sql
-- Ensuring a parent row exists while inserting a child
-- (like checking an order exists before adding line items)

BEGIN;
SELECT * FROM orders WHERE id = 100 FOR SHARE;
-- Nobody can DELETE or UPDATE this order while we work
INSERT INTO order_items (order_id, product_id, qty) VALUES (100, 5, 2);
COMMIT;
```

### SKIP LOCKED: Don't Wait, Skip!

**What's happening:** Normally, if you try to lock a row that's already locked, you wait. `SKIP LOCKED` says "if it's locked, just pretend it doesn't exist and give me the next one." This is perfect for job queues where multiple workers grab tasks - each worker gets a different task instantly, no waiting.

**When does this occur?** Any time you have multiple consumers pulling from a shared queue: background job processing, email sending, notification dispatching, order fulfillment workers.

**How to avoid problems:** Always pair SKIP LOCKED with `LIMIT` and `ORDER BY` so each worker gets a predictable, unique task.

```sql
-- Job queue pattern: multiple workers grab tasks without blocking

-- Worker 1:
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;   -- grabs first available task

-- Worker 2 (runs at same time):
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;   -- skips Worker 1's task, grabs next one!
```

```
┌──────────────────────────────────────────────────────────────────┐
│  tasks table:                                                    │
│  ┌────┬──────────┬────────────┐                                  │
│  │ id │  status  │  locked_by │                                  │
│  ├────┼──────────┼────────────┤                                  │
│  │  1 │ pending  │ Worker 1 🔒│  ← Worker 1 grabs this          │
│  │  2 │ pending  │ Worker 2 🔒│  ← Worker 2 skips #1, grabs #2  │
│  │  3 │ pending  │            │  ← Available for next worker     │
│  │  4 │ pending  │            │                                  │
│  └────┴──────────┴────────────┘                                  │
└──────────────────────────────────────────────────────────────────┘
```

### NOWAIT: Don't Wait, Fail!

**What's happening:** Instead of waiting (potentially forever) for a locked row, `NOWAIT` makes your query fail immediately with an error if the row is already locked. Your app catches the error and can show "this resource is busy, try again later."

**When does this occur?** When you'd rather fail fast than make a user stare at a loading spinner. Great for: seat selection in ticket booking (show "someone else is selecting this seat"), real-time bidding, or any UI where instant feedback is better than waiting.

```sql
-- Instead of waiting for a locked row, fail immediately
SELECT * FROM seats WHERE id = 42 FOR UPDATE NOWAIT;

-- If row is locked:
-- ERROR: could not obtain lock on row in relation "seats"
```

---

## 5. Advisory Locks

Advisory locks are **application-level locks** that PostgreSQL manages for you but doesn't enforce automatically. You decide when to lock and what it means.

### The Apartment Building Analogy

```
Regular locks = Building's automatic door locks (work without you thinking)
Advisory locks = A "meeting in progress" sign (you put it up, you take it down)
```

### Types of Advisory Locks

| Type | Scope | Syntax | Released When |
|---|---|---|---|
| **Session-level** | Entire connection | `pg_advisory_lock(id)` | Explicit unlock or session ends |
| **Transaction-level** | Current transaction | `pg_advisory_xact_lock(id)` | Transaction ends (COMMIT/ROLLBACK) |
| **Try (non-blocking)** | Either | `pg_try_advisory_lock(id)` | Returns true/false instead of waiting |

### Real Examples

**Example 1: Prevent duplicate cron job execution**

**What's happening:** Your cron runs every minute, but sometimes the previous run hasn't finished yet. Without protection, two copies run at the same time and do duplicate work (or corrupt data). An advisory lock acts as a "one at a time" gate - if the lock is taken, the new run skips immediately.

**When does this occur?** Any scheduled job that might overlap: report generation, data sync, cleanup tasks, sending digest emails.

```sql
-- Cron job runs every minute, but should only run ONE instance at a time

-- Try to acquire lock (non-blocking)
SELECT pg_try_advisory_lock(12345);
-- Returns: true  → you got the lock, run the job
-- Returns: false → another instance is running, skip

-- When done:
SELECT pg_advisory_unlock(12345);
```

**Example 2: One migration at a time**

**What's happening:** You have 10 app servers, and all of them try to run database migrations on startup. Without coordination, they'd all run the same migration simultaneously and crash. An advisory lock ensures only one server runs migrations while the others wait their turn.

**When does this occur?** Multi-server deployments where all instances start at once (rolling deploys, Kubernetes pod restarts, autoscaling events).

```sql
-- Ensure only one migration process runs across all app servers

BEGIN;
SELECT pg_advisory_xact_lock(hashtext('db-migration'));
-- Only one connection gets here at a time
-- Run migration...
COMMIT;  -- lock automatically released
```

**Example 3: Per-user rate limiting at DB level**

**What's happening:** Two API requests arrive for the same user at the exact same time. Both try to process the user's data. The advisory lock ensures only one processes at a time, preventing duplicate charges, double emails, or corrupted state.

**When does this occur?** Double-click submissions, mobile app retry storms, webhook redelivery, or any case where the same logical operation can be triggered twice for the same entity.

```sql
-- Lock per user to prevent double-processing of the same user
SELECT pg_advisory_xact_lock(hashtext('process-user'), user_id);
-- Process user's data...
-- Lock released on COMMIT
```

### Advisory Lock Gotchas

```
┌──────────────────────────────────────────────────────────────────────┐
│  ⚠️  WARNING: Session-level advisory locks are NOT released on       │
│     COMMIT/ROLLBACK! They stay until you explicitly unlock           │
│     or disconnect. This is the #1 source of advisory lock bugs.     │
│                                                                      │
│  ⚠️  Advisory locks are re-entrant! Calling pg_advisory_lock(42)     │
│     twice means you must call pg_advisory_unlock(42) twice.         │
│                                                                      │
│  ✅  PREFER pg_advisory_xact_lock() - it's safer (auto-releases).   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. Deadlocks

A **deadlock** is when two transactions are each waiting for the other to release a lock. Neither can proceed.

### The Hallway Problem

```
┌────────────────────────────────────────────────────┐
│                                                    │
│  Person A ──────→  🚪  ←────── Person B            │
│                                                    │
│  A is waiting     Narrow    B is waiting           │
│  for B to move    hallway   for A to move          │
│                                                    │
│         DEADLOCK! Nobody can move! 💀              │
└────────────────────────────────────────────────────┘
```

### Classic Deadlock Example

**What's happening:** Imagine two bank clerks processing money transfers at the same time. Clerk 1 grabs Account #1's file and needs Account #2's file next. Clerk 2 grabs Account #2's file and needs Account #1's file next. They're both holding what the other needs and waiting for what the other has. Nobody moves.

**When does this occur?** Any time two (or more) transactions update the same rows but in **different order**. Common in:
- Money transfers between accounts (A->B and B->A simultaneously)
- Batch jobs that update overlapping sets of rows
- Any code that does multiple UPDATEs/DELETEs within one transaction without consistent ordering

```sql
-- Session 1                          -- Session 2
BEGIN;                                BEGIN;
UPDATE accounts SET balance = 100     UPDATE accounts SET balance = 200
WHERE id = 1;   -- locks row 1       WHERE id = 2;   -- locks row 2

UPDATE accounts SET balance = 300     UPDATE accounts SET balance = 400
WHERE id = 2;   -- ⏳ WAITS (row 2)  WHERE id = 1;   -- ⏳ WAITS (row 1)

-- 💀 DEADLOCK!
-- Session 1 has row 1, wants row 2
-- Session 2 has row 2, wants row 1
-- Neither can proceed!
```

**How to avoid:** Always lock rows in the same order (e.g., by ascending ID). If both clerks always grab the lower-numbered account first, they'll never deadlock.

```
┌──────────────────────────────────────────────────────┐
│           Deadlock Visualized                        │
│                                                      │
│    Session 1                    Session 2             │
│       │                            │                 │
│       │  HOLDS lock on             │  HOLDS lock on  │
│       ▼  row 1                     ▼  row 2          │
│    ┌──────┐                     ┌──────┐             │
│    │Row 1 │────── WANTS ───────→│Row 2 │             │
│    └──────┘                     └──────┘             │
│       ▲                            │                 │
│       │                            │                 │
│       └──────── WANTS ─────────────┘                 │
│                                                      │
│    Circular dependency = DEADLOCK                    │
└──────────────────────────────────────────────────────┘
```

### How PostgreSQL Handles Deadlocks

PostgreSQL has a **deadlock detector** that runs periodically (default: every 1 second, configured by `deadlock_timeout`).

When it finds a deadlock:
1. It picks one transaction as the **victim**
2. Aborts the victim with: `ERROR: deadlock detected`
3. The other transaction proceeds

### How to PREVENT Deadlocks

**Rule #1: Always lock resources in the SAME ORDER**

```sql
-- WRONG: different order = deadlock risk
-- Session 1: locks row 1 then row 2
-- Session 2: locks row 2 then row 1

-- RIGHT: always lock in ascending ID order
-- Session 1: locks row 1 then row 2
-- Session 2: locks row 1 then row 2 (same order!)
```

**Rule #2: Keep transactions short**

```sql
-- WRONG: long transaction holding locks
BEGIN;
UPDATE accounts SET balance = 100 WHERE id = 1;
-- ... do some slow API call here ... (DON'T!)
UPDATE accounts SET balance = 200 WHERE id = 2;
COMMIT;

-- RIGHT: do slow work OUTSIDE the transaction
-- 1. Fetch data and do API calls
-- 2. THEN start a short transaction
BEGIN;
UPDATE accounts SET balance = 100 WHERE id = 1;
UPDATE accounts SET balance = 200 WHERE id = 2;
COMMIT;
```

**Rule #3: Use SELECT FOR UPDATE with ORDER BY**

```sql
-- Lock multiple rows in a predictable order
SELECT * FROM accounts
WHERE id IN (1, 2, 3)
ORDER BY id          -- ← ensures consistent lock order!
FOR UPDATE;
```

---

## 7. Lock Monitoring & Troubleshooting

### View Current Locks

```sql
-- Show all current locks with human-readable info
SELECT
    pid,
    pg_blocking_pids(pid) AS blocked_by,
    usename,
    query,
    mode,
    locktype,
    relation::regclass,
    granted,
    wait_event_type,
    state,
    age(clock_timestamp(), query_start) AS query_age
FROM pg_locks
JOIN pg_stat_activity USING (pid)
WHERE relation IS NOT NULL
ORDER BY query_start;
```

### Find Blocked Queries (Who Is Waiting?)

```sql
-- Show blocked queries and what's blocking them
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocked.query AS blocked_query,
    age(clock_timestamp(), blocked.query_start) AS waiting_duration,
    blocker.pid AS blocker_pid,
    blocker.usename AS blocker_user,
    blocker.query AS blocker_query,
    age(clock_timestamp(), blocker.query_start) AS blocker_duration
FROM pg_stat_activity AS blocked
JOIN LATERAL (
    SELECT *
    FROM pg_stat_activity
    WHERE pid = ANY(pg_blocking_pids(blocked.pid))
) AS blocker ON true
WHERE blocked.wait_event_type = 'Lock';
```

### Find Long-Running Transactions (Lock Hogs)

```sql
-- Transactions running for more than 5 minutes
SELECT
    pid,
    usename,
    state,
    age(clock_timestamp(), xact_start) AS transaction_age,
    age(clock_timestamp(), query_start) AS query_age,
    query
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start < clock_timestamp() - interval '5 minutes'
ORDER BY xact_start;
```

### Kill a Blocking Session

```sql
-- Gentle: cancel the current query (they can retry)
SELECT pg_cancel_backend(<pid>);

-- Forceful: terminate the entire connection
SELECT pg_terminate_backend(<pid>);
```

```
┌────────────────────────────────────────────────────────────────┐
│  Troubleshooting Flow                                          │
│                                                                │
│  "My queries are slow/stuck!"                                  │
│       │                                                        │
│       ▼                                                        │
│  Check pg_stat_activity → Is state = 'active' + wait = 'Lock'?│
│       │                                                        │
│       ├── YES → Find blocker with pg_blocking_pids()           │
│       │         │                                              │
│       │         ├── Blocker is idle-in-transaction?             │
│       │         │   → Likely a forgotten COMMIT. Kill it.      │
│       │         │                                              │
│       │         ├── Blocker is a long ALTER TABLE?              │
│       │         │   → Wait or cancel the migration.            │
│       │         │                                              │
│       │         └── Blocker is a long SELECT?                  │
│       │             → Consider statement_timeout.              │
│       │                                                        │
│       └── NO  → Not a lock issue. Check query plan (EXPLAIN).  │
└────────────────────────────────────────────────────────────────┘
```

### Monitor Advisory Locks

```sql
-- View all held advisory locks
SELECT
    pid,
    classid,
    objid,
    mode,
    granted
FROM pg_locks
WHERE locktype = 'advisory';
```

### Useful PostgreSQL Settings for Lock Management

| Setting | Default | What It Does |
|---|---|---|
| `lock_timeout` | `0` (disabled) | Max time to wait for a lock before giving up |
| `statement_timeout` | `0` (disabled) | Max time for any statement to run |
| `deadlock_timeout` | `1s` | How often deadlock detector runs |
| `idle_in_transaction_session_timeout` | `0` (disabled) | Kill sessions idle inside a transaction |
| `log_lock_waits` | `off` | Log when a query waits for a lock longer than `deadlock_timeout` |

```sql
-- Recommended: set these in your application or session
SET lock_timeout = '5s';        -- don't wait more than 5s for locks
SET statement_timeout = '30s';  -- don't let any query run more than 30s

-- Recommended: set globally to catch idle transactions
ALTER SYSTEM SET idle_in_transaction_session_timeout = '5min';
ALTER SYSTEM SET log_lock_waits = 'on';
SELECT pg_reload_conf();
```

---

## 8. Common Lock Problems & Solutions

### Problem 1: Adding a column with a default value

**What's happening:** Before PostgreSQL 11, adding a column with a DEFAULT rewrote every single row in the table. On a 100M row table, this could take minutes - during which the entire table is locked and nobody can even SELECT from it. In PG 11+, the default is stored in the catalog (metadata), so no rewrite needed. But you still need the ACCESS EXCLUSIVE lock briefly to update the catalog, which can queue up behind existing queries.

```sql
-- ❌ DANGEROUS on large tables (pre-PostgreSQL 11)
-- Rewrites entire table, holds ACCESS EXCLUSIVE lock for a long time!
ALTER TABLE orders ADD COLUMN status VARCHAR(20) DEFAULT 'pending';

-- ✅ SAFE in PostgreSQL 11+
-- PostgreSQL 11+ does NOT rewrite the table for simple defaults
-- The above command is now fast! Just be aware of lock queue.

-- ✅ EXTRA SAFE with lock timeout
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN status VARCHAR(20) DEFAULT 'pending';
-- If it can't get the lock in 3 seconds, it fails instead of blocking everything
```

### Problem 2: Adding an index blocks writes

**What's happening:** A regular `CREATE INDEX` takes a SHARE lock, which blocks all INSERT/UPDATE/DELETE on the table while the index is being built. On a large table, index creation can take minutes or even hours. `CREATE INDEX CONCURRENTLY` builds the index in the background using a weaker lock, so writes continue normally - it just takes longer to build.

```sql
-- ❌ BLOCKS all INSERT/UPDATE/DELETE while building the index
CREATE INDEX idx_orders_email ON orders(email);

-- ✅ SAFE: build index without blocking writes
CREATE INDEX CONCURRENTLY idx_orders_email ON orders(email);
-- Takes longer but doesn't block DML operations
-- Uses SHARE UPDATE EXCLUSIVE instead of SHARE lock
```

### Problem 3: Adding NOT NULL constraint scans entire table

**What's happening:** When you add `NOT NULL`, PostgreSQL has to check every single row to make sure none of them have a NULL value in that column. While doing this scan, it holds an ACCESS EXCLUSIVE lock. The trick: add a CHECK constraint with `NOT VALID` first (instant, skips scanning existing rows), then `VALIDATE` it separately (scans rows but with a much lighter lock that allows reads and writes), then convert to NOT NULL (instant in PG 12+ because it sees the validated constraint).

```sql
-- ❌ Scans entire table while holding ACCESS EXCLUSIVE lock
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;

-- ✅ SAFE: add a CHECK constraint first, then convert
ALTER TABLE orders ADD CONSTRAINT orders_status_not_null
    CHECK (status IS NOT NULL) NOT VALID;          -- doesn't scan existing rows
ALTER TABLE orders VALIDATE CONSTRAINT orders_status_not_null;  -- scans but doesn't block writes
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;             -- instant (PG 12+, knows constraint exists)
ALTER TABLE orders DROP CONSTRAINT orders_status_not_null;       -- cleanup
```

### Problem 4: Foreign key creation locks both tables

**What's happening:** Adding a foreign key must verify that every value in the child column exists in the parent table. This scan locks both tables. The safe pattern: add the FK with `NOT VALID` (instant, doesn't check existing rows, but new inserts are validated), then `VALIDATE` separately (scans existing rows with a weaker lock).

```sql
-- ❌ Locks both parent and child tables
ALTER TABLE order_items ADD CONSTRAINT fk_order
    FOREIGN KEY (order_id) REFERENCES orders(id);

-- ✅ SAFE: add NOT VALID first, then validate separately
ALTER TABLE order_items ADD CONSTRAINT fk_order
    FOREIGN KEY (order_id) REFERENCES orders(id) NOT VALID;
ALTER TABLE order_items VALIDATE CONSTRAINT fk_order;
```

### Problem 5: VACUUM FULL blocks everything

**What's happening:** Regular VACUUM marks dead rows as reusable (lightweight, runs in background). `VACUUM FULL` physically rewrites the entire table to reclaim disk space - it creates a brand new copy of the table, then swaps it in. During the rewrite, nothing can touch the table. Use `pg_repack` instead - it does the same rewrite but online, only taking a brief lock at the very end to swap.

```sql
-- ❌ VACUUM FULL takes ACCESS EXCLUSIVE lock - blocks all queries
VACUUM FULL large_table;

-- ✅ Use pg_repack instead (online repacking, minimal locking)
-- pg_repack -d mydb -t large_table

-- ✅ Or just use regular VACUUM (much lighter lock)
VACUUM large_table;
```

---

## 9. Lock-Safe Migration Patterns

### The Safe Migration Template

```sql
-- Always set a lock timeout in migrations!
-- If you can't acquire the lock quickly, FAIL FAST and retry later.

SET lock_timeout = '5s';
SET statement_timeout = '30s';

-- Your DDL here...
ALTER TABLE orders ADD COLUMN status VARCHAR(20);

-- Reset timeouts
RESET lock_timeout;
RESET statement_timeout;
```

### Migration Decision Tree

```
Want to change a table?
│
├── ADD COLUMN
│   ├── Without default → ✅ Fast (just metadata change)
│   ├── With default (PG 11+) → ✅ Fast (stored in catalog)
│   └── With default (PG <11) → ⚠️ Rewrites table! Use 3-step approach
│
├── DROP COLUMN
│   └── ✅ Fast (just marks column as dropped, no rewrite)
│
├── ADD INDEX
│   └── Always use CREATE INDEX CONCURRENTLY ✅
│
├── ADD CONSTRAINT (NOT NULL)
│   └── Use CHECK + VALIDATE + SET NOT NULL pattern ✅
│
├── ADD CONSTRAINT (FOREIGN KEY)
│   └── Use NOT VALID + VALIDATE pattern ✅
│
├── CHANGE COLUMN TYPE
│   └── ⚠️ Rewrites table! Consider adding new column + backfill + swap
│
├── RENAME COLUMN
│   └── ✅ Fast (just metadata), but breaks app queries!
│
└── DROP TABLE / TRUNCATE
    └── ⚠️ ACCESS EXCLUSIVE - schedule during low traffic
```

### Retry Pattern for Migrations

```python
# Python pseudocode for safe migration with retries
MAX_RETRIES = 5
LOCK_TIMEOUT = "3s"

for attempt in range(MAX_RETRIES):
    try:
        execute(f"SET lock_timeout = '{LOCK_TIMEOUT}'")
        execute("ALTER TABLE orders ADD COLUMN status VARCHAR(20)")
        print("Migration succeeded!")
        break
    except LockTimeoutError:
        print(f"Attempt {attempt + 1} failed, lock busy. Retrying...")
        sleep(2 ** attempt)  # exponential backoff: 1s, 2s, 4s, 8s, 16s
else:
    raise Exception("Migration failed after all retries")
```

---

## 10. Quick Reference Cheat Sheet

### Common Operations & Their Lock Impact

| Operation | Lock Level | Blocks SELECTs? | Blocks DML? | Safe? |
|---|---|---|---|---|
| `SELECT` | ACCESS SHARE | No | No | Always |
| `INSERT/UPDATE/DELETE` | ROW EXCLUSIVE | No | Row-level only | Always |
| `CREATE INDEX` | SHARE | No | **YES** | Use CONCURRENTLY |
| `CREATE INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | No | No | Always |
| `VACUUM` | SHARE UPDATE EXCLUSIVE | No | No | Always |
| `VACUUM FULL` | ACCESS EXCLUSIVE | **YES** | **YES** | Use pg_repack |
| `ALTER TABLE` (add col) | ACCESS EXCLUSIVE | **YES** | **YES** | Set lock_timeout |
| `ALTER TABLE` (change type) | ACCESS EXCLUSIVE | **YES** | **YES** | Avoid on big tables |
| `DROP TABLE` | ACCESS EXCLUSIVE | **YES** | **YES** | Off-peak hours |
| `TRUNCATE` | ACCESS EXCLUSIVE | **YES** | **YES** | Off-peak hours |

### Quick Troubleshooting Commands

```sql
-- "What's blocking me?"
SELECT pg_blocking_pids(pg_backend_pid());

-- "Who holds locks on my table?"
SELECT * FROM pg_locks WHERE relation = 'my_table'::regclass;

-- "How long has this transaction been running?"
SELECT pid, age(clock_timestamp(), xact_start), query
FROM pg_stat_activity WHERE state = 'idle in transaction';

-- "Kill a stuck session"
SELECT pg_terminate_backend(<pid>);
```

### Golden Rules

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  1. Always set lock_timeout in migrations                           │
│                                                                      │
│  2. Always use CREATE INDEX CONCURRENTLY                            │
│                                                                      │
│  3. Keep transactions as SHORT as possible                          │
│                                                                      │
│  4. Never do slow work (API calls, computation) inside a transaction│
│                                                                      │
│  5. Lock resources in a consistent ORDER to prevent deadlocks       │
│                                                                      │
│  6. Use SKIP LOCKED for job queues, NOWAIT for impatient queries    │
│                                                                      │
│  7. Monitor idle-in-transaction sessions — they hold locks silently │
│                                                                      │
│  8. Prefer NOT VALID + VALIDATE for constraints on large tables     │
│                                                                      │
│  9. Enable log_lock_waits to detect lock issues in production       │
│                                                                      │
│  10. Test DDL migrations against production-sized data first        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```
