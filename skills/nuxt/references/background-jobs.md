# Background Jobs

Use this reference when a Nuxt project needs asynchronous or retryable processing.

Always use BullMQ for background jobs in Nuxt projects covered by this skill. Do not introduce alternative queue libraries unless the user explicitly overrides that constraint.

## When To Add A Worker

Move work out of the request path when it is:

- slow
- retryable
- resource-intensive
- dependent on external APIs
- better tracked through explicit status transitions

Common examples: uploads, AI processing, report generation, large imports, and media conversion.

## Recommended Structure

Use a split like this:

- `server/`: accepts the request and enqueues work
- `lib/queue/`: queue names, payload types, Redis connection, queue factories
- `worker/`: queue consumers and process bootstrap
- `lib/services/`: job logic shared by workers and sometimes by server code
- `lib/database.*`: status updates and durable job output writes

Use BullMQ as the queue implementation for this structure.

## BullMQ Pattern

Centralize:

- queue names
- job names
- payload types
- queue default options
- Redis connection creation

Do not scatter queue configuration across routes and workers.

## Job Lifecycle Pattern

Prefer this lifecycle:

1. Persist the primary record first.
2. Mark it as queued or uploaded.
3. Enqueue a named job with a small payload, usually IDs and optional instructions.
4. In the worker, load the full state from the database or storage.
5. Mark the record as processing.
6. Run the long task.
7. Persist durable outputs.
8. Mark the record completed or failed.

Keep payloads small and reload authoritative state during execution.

## Failure And Retry Pattern

- Persist a user-visible failure message before rethrowing.
- Let BullMQ own retries.
- Use explicit job IDs when deduplication matters.
- Use backoff rather than tight retry loops.
- Retain a limited number of completed and failed jobs for debugging.

If a job can be safely retried, write the processing service to be idempotent or to tolerate repeated execution.

## Worker Design Pattern

Keep worker code minimal:

- bootstrap dependencies
- start queue consumers
- dispatch by job name
- log lifecycle events

Put business logic in shared service modules so the worker entrypoint stays simple.

## Storage And Ops Lessons

- The worker and app must share access to whatever durable storage contains input and output artifacts.
- Redis availability becomes production-critical once jobs are part of core flows.
- Concurrency should reflect CPU, memory, external API limits, and idempotency guarantees rather than guesswork.
- Because BullMQ depends on Redis, treat Redis health, persistence, and connectivity as part of the core application path.
