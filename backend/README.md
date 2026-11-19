# Backend

This directory contains the backend services for the project.

## Analytics: Historical Contract-State Snapshots

For audit queries, the analytics service persists **bounded, immutable contract-state
snapshots** at documented ledger checkpoints. Each snapshot is linked to the source
event range it was derived from and is validated against direct contract reads.

### Supported states

A snapshot captures the contract state at a single ledger checkpoint. Supported
snapshot states are:

- `pending` — snapshot record created, reconstruction not yet validated.
- `validated` — reconstruction matched the direct contract read at the checkpoint.
- `mismatch` — reconstruction diverged from the direct contract read; the snapshot is
  retained for audit but flagged as unreliable.
- `failed` — reconstruction or validation could not complete (dependency failure).

Only `validated` snapshots are served to audit queries by default; `mismatch` and
`failed` snapshots are retained for inspection and are never silently promoted.

### Input constraints

Snapshot requests are rejected (without any partial persistence) when:

- the requested ledger checkpoint is outside the documented, bounded checkpoint range;
- the source event range is empty or its start is after its end;
- the event range does not fall within the checkpoint window it claims to describe;
- a snapshot already exists for the same checkpoint and event range (idempotent replay
  returns the existing immutable snapshot rather than writing a new one).

### Authorization / consent

Creating or invalidating snapshots requires the analytics operator role. Read access to
snapshots follows the same authorization as the underlying audit queries. No snapshot
is created on behalf of a user without an explicit operator action.

### Failure behavior

Snapshot creation is transactional: either the snapshot record and its event-range link
are persisted together, or nothing is written. Validation failures mark the snapshot
`mismatch`/`failed` rather than deleting it, preserving the audit trail. Stale requests
(checkpoint already superseded by a newer validated snapshot) are rejected without
side effects.

### Persistence and compatibility

Snapshots are stored alongside the existing event/indexer schema and reference the
source event range by ledger bounds. The snapshot table is additive; existing event and
indexer records are unchanged. Consumers that do not read snapshots are unaffected.
Any future change to the stored snapshot schema must be documented here with its
migration implications before rollout.
