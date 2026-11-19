# Adaptive Catch-Up Scheduling Design

**Status**: Design Phase  
**Version**: 1.0  
**Date**: 2026-10-16

## Executive Summary

Replace the fixed 30-second poll interval in EventIndexer with an adaptive scheduling system that detects when the indexer falls behind the chain tip and automatically accelerates catch-up. The system maintains backward compatibility while improving responsiveness during peak event periods and RPC degradation.

## Objectives

1. **Reduce catch-up latency** when the indexer falls behind by 100+ ledgers
2. **Gracefully degrade** when RPC is slow or unstable
3. **Maintain backward compatibility** with all existing API consumers and deployed contracts
4. **Add observability** for indexer scheduling decisions and health
5. **Enable operators** to tune behavior via environment variables

## Algorithm Overview

### Catch-Up Modes

The scheduler operates in three modes:

#### 1. **Caught-Up Mode** (lag ≤ CAUGHT_UP_THRESHOLD)
- Poll interval: 30 seconds (normal)
- Batch size: 100 (normal)
- Rationale: Chain is current; periodic polling is sufficient

#### 2. **Behind Mode** (CAUGHT_UP_THRESHOLD < lag ≤ AGGRESSIVE_THRESHOLD)
- Poll interval: 10 seconds (aggressive)
- Batch size: 200 (increased)
- Rationale: Fell behind but not severely; moderate acceleration

#### 3. **Catch-Up Mode** (lag > AGGRESSIVE_THRESHOLD)
- Poll interval: 3 seconds (very aggressive)
- Batch size: 500 (max)
- Rationale: Significantly behind; maximum acceleration until caught up

### Transition Rules

1. **Caught-Up → Behind**: When `lag > CAUGHT_UP_THRESHOLD`
2. **Behind → Catch-Up**: When `lag > AGGRESSIVE_THRESHOLD`
3. **Catch-Up → Behind**: When `lag ≤ AGGRESSIVE_THRESHOLD && lag > CAUGHT_UP_THRESHOLD`
4. **Behind/Catch-Up → Caught-Up**: When `lag ≤ CAUGHT_UP_THRESHOLD`

### Backoff Strategy

When RPC fails or returns 0 ledger:

1. **First failure**: Log warning, keep current mode
2. **Consecutive failures**: Apply exponential backoff
   - Failure 1: wait 5 seconds extra
   - Failure 2: wait 15 seconds extra (3x)
   - Failure 3: wait 45 seconds extra (3x)
   - Failure 4+: wait 135 seconds extra (capped)
3. **Recovery**: After 1 successful poll, reset failure counter

**Backoff Formula**:
```
delay_ms = base_interval + (backoff_base_ms * (3 ^ consecutive_failures))
capped at: base_interval + backoff_max_ms
```

### State Machine

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌──────────────┐         ┌──────────────┐               │
│  │  Caught-Up   │<───────>│   Behind     │                │
│  │(30s, 100)    │         │ (10s, 200)   │                │
│  └──────────────┘         └──────────────┘                │
│        ^                         │                         │
│        │ lag ≤ threshold         │ lag > aggressive       │
│        │                         ▼                         │
│        │                   ┌──────────────┐               │
│        └──────────────────│  Catch-Up    │               │
│                           │ (3s, 500)    │               │
│                           └──────────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Backoff applied to all modes on RPC failures
```

## State Persistence

### New Table: `indexer_scheduler_state`

```sql
CREATE TABLE indexer_scheduler_state (
  id SERIAL PRIMARY KEY,
  contract VARCHAR(56) NOT NULL UNIQUE,
  
  -- Current mode: 'caught_up', 'behind', or 'catch_up'
  mode VARCHAR(20) NOT NULL DEFAULT 'caught_up',
  
  -- Backoff tracking
  consecutive_failures INT NOT NULL DEFAULT 0,
  last_poll_time TIMESTAMP,
  last_successful_poll_time TIMESTAMP,
  
  -- Diagnostic metadata
  last_lag_ledgers INT,
  last_error_message TEXT,
  
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**Why**: Survive restarts without losing catch-up context. Allows operational diagnostics.

## Configuration

### Environment Variables

```env
# Adaptive scheduler thresholds (ledgers)
INDEXER_CAUGHT_UP_THRESHOLD=50        # Switch from catch-up to behind
INDEXER_AGGRESSIVE_THRESHOLD=100      # Switch from behind to catch-up

# Mode-specific intervals (milliseconds)
INDEXER_INTERVAL_CAUGHT_UP_MS=30000   # Normal polling
INDEXER_INTERVAL_BEHIND_MS=10000      # Moderate acceleration
INDEXER_INTERVAL_CATCH_UP_MS=3000     # Aggressive catch-up

# Mode-specific batch sizes (ledgers)
INDEXER_BATCH_SIZE_CAUGHT_UP=100      # Normal batch
INDEXER_BATCH_SIZE_BEHIND=200         # Increased during behind
INDEXER_BATCH_SIZE_CATCH_UP=500       # Max batch during catch-up

# Backoff configuration
INDEXER_BACKOFF_BASE_MS=5000          # Initial backoff
INDEXER_BACKOFF_MAX_MS=135000         # Max total backoff
INDEXER_BACKOFF_MULTIPLIER=3          # Exponential multiplier

# Feature flag for adaptive mode (safe to disable for rollback)
INDEXER_ADAPTIVE_SCHEDULING=true
```

**Backward Compatibility**: If `INDEXER_ADAPTIVE_SCHEDULING=false`, fall back to fixed interval/batch (current behavior).

## Metrics

### New Gauges

- `indexer_scheduler_mode` (gauge) - Current mode: 0=caught_up, 1=behind, 2=catch_up
- `indexer_backoff_level` (gauge) - Current backoff multiplier (0-N)
- `indexer_consecutive_failures` (counter) - Consecutive RPC failures since last success

### New Counters

- `indexer_mode_transitions` (counter) - Transitions between modes (label: from→to)
- `indexer_backoff_triggered` (counter) - Times backoff was activated

### Existing Metrics (Enhanced)

- `indexer_lag_ledgers` (gauge) - Already tracked; will drive mode transitions
- `indexer_poll_duration_ms` (histogram) - Already tracked; used to tune intervals

## Logging

### Structured Log Events

1. **Mode Transition**
   ```json
   {
     "event": "indexer_mode_transition",
     "from": "caught_up",
     "to": "behind",
     "lag": 120,
     "threshold": 100,
     "interval_ms": 10000,
     "batch_size": 200
   }
   ```

2. **Backoff Triggered**
   ```json
   {
     "event": "indexer_backoff_triggered",
     "consecutive_failures": 2,
     "backoff_ms": 45000,
     "total_wait": 55000,
     "reason": "RPC getLatestLedger failed"
   }
   ```

3. **Catch-Up Completed**
   ```json
   {
     "event": "indexer_catch_up_complete",
     "lag_before": 500,
     "lag_after": 0,
     "duration_ms": 15000,
     "events_processed": 1250
   }
   ```

## Authorization & Security

### API Access

- `/indexer/status` returns `scheduler_mode` and `backoff_level` in response (no auth change)
- `/admin/indexer/reset-backoff` to reset consecutive_failures (admin:indexer scope, new)
- No sensitive data exposed in metrics or logs

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| DoS by rapid config changes | Environment variables only; no runtime API |
| Unbounded memory in failure tracking | Consecutive_failures capped; backoff capped |
| Information disclosure | All metrics are operational health; no PII/secrets |
| Unintended catch-up spam | Thresholds tuned conservatively; metrics monitored |

## Failure Scenarios

### Scenario 1: RPC Becomes Slow
- **Initial**: Caught-Up mode, 30s interval
- **Minute 1**: RPC latency 5s → poll takes 8s, still 30s interval (lag grows)
- **Minute 2**: Lag > 50 → Enter Behind mode, 10s interval
- **Minute 3**: Lag > 100 → Enter Catch-Up mode, 3s interval (plus backoff)
- **Minute 5**: RPC recovers → Lag shrinks, exit Catch-Up, shrink batch, restore 30s

### Scenario 2: RPC Outage During Restart
- **Restart**: `indexer_scheduler_state` loaded from DB
- **State**: Previous mode (e.g., "catch_up") restored
- **Backoff**: consecutive_failures = 0, backoff_level reset
- **Polling**: Resumes with adaptive intervals despite RPC down
- **Recovery**: After RPC recovers, scheduler adjusts based on lag

### Scenario 3: Ledger Gap + Slow Recovery
- **Gap detected**: 200-ledger gap
- **Mode**: Catch-Up (3s interval, 500 batch)
- **Progress**: Each poll advances ~500 ledgers
- **Recovery**: Gap closes in ~2 polls (~6s + RPC latency)
- **Backoff**: If RPC fails mid-recovery, backoff applies but mode stays Catch-Up

## Historical Contract-State Snapshots (Audit Queries)

### Purpose

Audit queries need to reconstruct the contract's state as of a specific ledger without replaying the entire event history. This section defines bounded, immutable snapshots persisted at documented ledger checkpoints, linked to the source event range they summarize, and validated against direct contract reads.

### Supported States

A snapshot captures the indexer's materialized contract state at a checkpoint ledger. Supported states are exactly those the indexer already persists for the contract (e.g. balances, allowances, and any other indexed aggregates). No new state is derived; snapshots are a point-in-time projection of existing indexed data.

### Input Constraints

- **Checkpoint ledger** must be a documented checkpoint (see below) and must be `<=` the indexer's last fully-processed ledger. Requests for future or unprocessed ledgers are rejected.
- **Event range** must be ordered: `start_ledger <= end_ledger`, and `end_ledger` must equal the checkpoint ledger.
- **Staleness**: a snapshot request whose checkpoint is older than the retained snapshot window is rejected as stale rather than silently recomputed.
- **Bounded size**: the number of events in the source range is capped by `INDEXER_SNAPSHOT_MAX_EVENTS`; ranges exceeding the cap are rejected.

### Authorization & Consent

- Snapshot creation is an operator action requiring the `admin:indexer` scope (same scope as `/admin/indexer/reset-backoff`).
- Read access to snapshots for audit queries requires the existing analytics read scope; no new PII is exposed.
- No snapshot is created implicitly; consent is the explicit admin call.

### Failure Behavior

- Invalid inputs (unordered range, non-checkpoint ledger, future ledger, oversized range) are rejected **before** any persistence, so there are no partial side effects.
- If validation against a direct contract read fails, the snapshot row is written with `validation_status = 'failed'` and the mismatch details recorded; it is never marked `valid`.
- Snapshot writes are idempotent per `(contract, checkpoint_ledger)`: re-requesting an existing checkpoint returns the stored snapshot rather than creating a duplicate.

### Persistence

```sql
CREATE TABLE contract_state_snapshots (
  id SERIAL PRIMARY KEY,
  contract VARCHAR(56) NOT NULL,
  checkpoint_ledger BIGINT NOT NULL,

  -- Source event range this snapshot summarizes
  start_ledger BIGINT NOT NULL,
  end_ledger BIGINT NOT NULL,

  -- Immutable serialized state (exact money units preserved)
  state JSONB NOT NULL,

  -- Validation against direct contract read
  validation_status VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending|valid|failed
  validation_details JSONB,

  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT uq_snapshot_checkpoint UNIQUE (contract, checkpoint_ledger),
  CONSTRAINT ck_snapshot_range CHECK (start_ledger <= end_ledger AND end_ledger = checkpoint_ledger)
);
```

Snapshots are immutable: rows are insert-only and never updated except to record the one-time validation outcome. Money values are stored in the same exact units used by the event/indexer schema (no floating point).

### Validation

After writing a snapshot, the indexer performs a direct contract read at the checkpoint ledger and compares the reconstructed state. The outcome is recorded in `validation_status`/`validation_details`. A `failed` snapshot remains queryable for audit but is flagged so consumers do not treat it as authoritative.

### Compatibility & Migration

- This is a new table; no existing event/indexer columns change, so existing consumers are unaffected.
- Migration is additive (`CREATE TABLE` only) and safe to roll back by dropping the table.
- Snapshot `state` mirrors the existing indexed representation; any future change to that representation must bump a `schema_version` field (reserved) and document the migration.

### Configuration

```env
# Documented checkpoint interval (ledgers between snapshots)
INDEXER_SNAPSHOT_CHECKPOINT_INTERVAL=1000
# Max events allowed in a snapshot's source range
INDEXER_SNAPSHOT_MAX_EVENTS=10000
# Retained snapshot window (ledgers); older checkpoints are rejected as stale
INDEXER_SNAPSHOT_RETENTION_LEDGERS=100000
```

### Test Coverage

- Successful snapshot creation at a documented checkpoint with a valid event range.
- Boundary: `start_ledger == end_ledger == checkpoint_ledger`.
- Rejection: unordered range, non-checkpoint ledger, future ledger, oversized range, stale checkpoint — each with no partial persistence.
- Idempotency: re-requesting an existing checkpoint returns the stored snapshot.
- Access control: creation without `admin:indexer` is denied.
- Dependency failure: direct contract read fails → snapshot recorded as `failed`, not `valid`.
