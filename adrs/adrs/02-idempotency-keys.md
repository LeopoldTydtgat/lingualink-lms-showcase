# ADR 2: Idempotency keys on the booking money-path

**Status:** Adopted, in production
**Date:** 2026

## Context

Booking, cancelling, and rescheduling a class all move value: hours are deducted from or refunded to a student's balance, and the class record drives teacher pay. These operations arrive from a web UI, which means retries are a fact of life - double-clicks, flaky connections, a browser resending a request, a user pressing the button again because nothing seemed to happen.

Without protection, a retried cancellation can refund the same hours twice, and a retried booking can deduct twice or create duplicate classes. On a platform handling real balances, "it usually does not happen" is not a standard.

## Decision

Make every money-path operation atomic and idempotent at the database layer.

- Each operation (book, cancel, reschedule) is a single Postgres function. All of its steps - checks, class record changes, balance movement - commit together or not at all. There is no state where hours moved but the class did not change.
- The client generates an idempotency key per user action and sends it with the request. The database function records the key on first execution and, on any repeat with the same key, returns the original result without performing the work again.
- Every path that can trigger the operation carries the key - there is no unkeyed side door.

## Alternatives considered

- **Disable the button after click.** UI-only protection. Rejected as the sole defense: it does nothing against network-level retries or a second tab.
- **Check current state before acting ("is this class already cancelled?").** Helps, but two concurrent requests can both pass the check before either commits. Rejected as the sole defense: it narrows the window without closing it.
- **Application-level locks or queues.** Serialises requests in the API layer. Rejected: more moving parts, and it still trusts every code path to go through the lock. The database is the only place every path must pass through.

## Consequences

**Good**
- Retries become harmless. The worst case of a double-click is the same answer twice.
- Money movements and record changes cannot drift apart, because they are one transaction.
- The guarantee lives in one place per operation, testable in isolation.

**Costs and sharp edges**
- Keys must be generated and threaded through every call site. When a new path to the same operation is added, wiring the key is a required step, and this is verified with tests rather than trusted to memory.
- Stored keys accumulate and need a retention rule.
- Idempotent functions must return the same shaped result on replay as on first run, which takes deliberate design.

## What I would tell someone building this

Assume every request arrives twice. If your money-path operations are single database transactions keyed by an idempotency token, that assumption costs you nothing. If they are not, you will find out from a user whose balance is wrong, and you will reconstruct what happened from logs. Building it in from the start is far cheaper than the forensic version.
