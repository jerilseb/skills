# Prisma ORM

Use this reference when adding or reviewing Prisma in a Nuxt project.

## Schema Design Rules

- Keep the schema in `prisma/schema.prisma`.
- Model explicit status values when the UI or worker logic depends on lifecycle state.
- Add indexes for frequent ownership, lookup, and status queries.
- Use relations to model ownership and dependent records clearly.

When the frontend needs lifecycle-aware data, define statuses deliberately rather than inferring them indirectly.

## Runtime Client Pattern

Create one shared Prisma client factory and reuse it everywhere.

In long-running Node runtimes, prefer a singleton so:

- server handlers do not create repeated clients
- worker processes reuse one configured client
- connection behavior is consistent

Keep Prisma construction in one module, not scattered across the codebase.

## Data-Access Layer Pattern

As the project grows, place Prisma queries behind a database or repository layer that handles:

- common reads
- writes and status transitions
- record mapping
- transactions
- ownership filtering

This keeps route handlers and workers focused on workflow, not query details.

## Mapping Pattern

Do not leak raw ORM shapes everywhere by default.

Map records when needed to:

- normalize dates
- transform JSON columns into typed app structures
- hide internal-only fields
- produce stable API shapes for the frontend

This is especially useful when the frontend polls long-running job status.

## Connection Configuration Pattern

Pick one strategy and keep it aligned across runtime and CLI:

- direct `DATABASE_URL`
- discrete `DATABASE_*` variables assembled into a URL

Either is fine. The key lesson is consistency. If the runtime builds a URL from individual variables, make Prisma CLI use the same inputs so development and deployment do not drift.

## Schema Change Workflow

Choose a workflow that matches the maturity of the project:

- early stage: `prisma generate` plus `prisma db push`
- production change history: Prisma migrations

Whichever path you choose, keep CI, local development, and container startup aligned with it.

## Lessons To Carry Forward

- A shared Prisma module plus a shared database layer makes server and worker code much easier to maintain.
- Explicit status enums work well when background jobs update records over time.
- Environment drift between Prisma CLI and runtime is a common source of bugs; centralize configuration early.
