# Concurrency Strategy — Issue #975

Documents the guards, failure modes, and recovery steps for race-prone mutations in PromptHash Stellar.

## Guards in use

| Mutation | Guard | Error on conflict |
|---|---|---|
| `buy_prompt` (purchase insert) | Unique compound index `{ buyerAddress, promptId }` | `DuplicateKeyError` → HTTP 409 |
| Price / metadata update | Optimistic version field `_version` | `StaleVersionError` → HTTP 409 |
| Webhook / event delivery | Idempotency key store (atomic check-and-set) | Replay → return cached response; conflict → HTTP 409 |
| Entitlement grant | Upsert with `$setOnInsert` | No-op (idempotent) |
| Lifecycle transition | Versioned conditional update | `StaleVersionError` → HTTP 409 |
| Timeline page | `(createdAt, _id)` composite cursor after visibility filtering | Reuse the cursor; inserts or permission changes do not shift the page |
| Maintainer action | Explicit approval linked to signed intent and outcome receipts | Block until approved; review the receipt on success or failure |

## Retry storm handling

- Clients must use exponential back-off with jitter after a 409.
- Maximum retries: 5. After that, surface a user-visible error.
- The idempotency key must be the same on every retry for the same logical operation.

## Failure modes

1. **Missing unique index**: duplicate records may be inserted. Detect via the dedup monitor (`server/scripts/dataIntegrityMonitor.ts`). Recover with `server/scripts/repairEntitlement.ts`.
2. **Version field absent**: last-writer-wins silently overwrites. Detect via the change-history audit trail. Recover by reverting to the last recorded version in `ChangeHistoryEntry`.
3. **Idempotency store unavailable**: fall back to processing the request as new (fail-open). Alert on the `idempotency.store_miss` metric and switch to fail-closed mode by returning 503.

## Rollback steps

1. Revert the deploy that removed a guard.
2. Run `server/scripts/repairEntitlement.ts --dry-run` to estimate affected records.
3. Apply repair, then re-run integrity checks.
4. Issue refunds for any duplicate charges via the payout ledger API.

## Operational recovery notes

- A timeline cursor is opaque and remains valid when records are inserted,
	deleted, or lose visibility. Malformed legacy cursors fall back to the
	historical `_id` boundary; clients can restart from the first page after an
	invalid response.
- Never retry a high-impact action with a new approval or idempotency key.
	Keep the original signed intent receipt, inspect its terminal outcome
	receipt, and create a new approval only after a maintainer has determined
	recovery is safe.
- If receipt persistence fails, treat the protected action as failed and use
	the approval record plus request ID to reconcile before retrying. Do not
	bypass approval during rollback.
