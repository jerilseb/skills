# Directory Structure

Use this reference when deciding where code belongs in a Nuxt project.

## Recommended Top-Level Shape

For a full-stack Nuxt app, this structure scales well:

- `app/`: frontend pages, components, composables, plugins, middleware, and assets
- `server/`: Nitro API handlers, server plugins, middleware, and route-specific backend code
- `worker/`: optional standalone process for queue-backed or long-running work
- `lib/` or `modules/`: shared backend logic used by Nitro routes and workers
- `shared/`: shared TypeScript types or validation schemas used on both client and server
- `prisma/`: Prisma schema and migration artifacts
- `data/` or storage adapter modules: local file storage helpers, seeds, or runtime data directories

Adapt the names to the project, but preserve the responsibility split.

## Placement Rules

Use these defaults:

- Put pages and UI components in `app/`.
- Put HTTP boundary logic in `server/api/` or the equivalent Nitro route location.
- Put startup-only server logic in a server plugin rather than inside arbitrary routes.
- Put long-running or retryable work in `worker/` instead of inside request handlers.
- Put reusable services in `lib/` so routes and workers can share them.
- Put data-access code behind a small database layer instead of calling Prisma everywhere.

## Thin Route Pattern

As projects grow, keep routes small:

1. Parse and validate the request.
2. Resolve auth and permissions.
3. Call a shared service or database function.
4. Return the response shape.

Avoid embedding complex business rules directly in server handlers.

## Shared Logic Pattern

Create shared helpers for:

- environment parsing
- storage path generation
- queue creation
- database access
- domain workflows

This keeps server routes and workers consistent and lowers duplication.

## Optional Worker Split

Introduce a separate `worker/` directory when you have tasks such as:

- document processing
- image generation or transformation
- AI summarization or extraction
- webhook retries
- export generation

The durable lesson is not the folder name itself; it is to separate user-facing request latency from long-running processing.

## Lessons To Carry Forward

- A dedicated shared layer between `server/` and `worker/` makes background work easier to evolve.
- Shared types help when frontend polling or detail views depend on backend status values.
- A startup plugin is a clean place for one-time initialization such as database connection warmup or storage bootstrapping.
