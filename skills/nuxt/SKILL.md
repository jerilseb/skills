---
name: nuxt
description: Guide for building and maintaining Nuxt applications with clear boundaries between app code, Nitro server routes, background jobs, Docker deployment, and Prisma/PostgreSQL data access. Use when Codex needs to scaffold, review, refactor, or extend a Nuxt project that includes server APIs, async processing, containerized deployment, or Prisma ORM.
---

# Nuxt

Use this skill for full-stack Nuxt projects, especially when the app includes Nitro APIs, a separate worker process, Dockerized deployment, or Prisma-backed persistence.

## Classify The Project First

Decide which shape you are working in before editing:

- Pure frontend Nuxt app
- Full-stack Nuxt app with Nitro APIs
- Nuxt app plus a separate background worker
- Nuxt app plus Prisma and PostgreSQL
- Nuxt app prepared for containerized deployment

Read only the slices relevant to that shape before making changes.

## Keep Stable Boundaries

- Keep UI in `app/`.
- Keep request/response handling in `server/`.
- Keep long-running background work in a dedicated worker process when requests should stay fast.
- Keep reusable business logic in shared modules rather than duplicating it in routes, workers, and components.
- Keep environment parsing and validation centralized.
- Keep storage and database access behind helper modules instead of scattering raw calls through the codebase.

## Prefer These Working Rules

- Keep route handlers thin: validate input, authorize, call shared logic, return a response.
- Keep job names, payload types, and queue creation centralized.
- Keep Prisma access centralized behind a data-access layer when the app grows beyond a few queries.
- Keep filesystem or object-storage path creation behind helper functions.
- Keep shared frontend/backend types in one place when the frontend consumes server-owned shapes.

## Load The Matching Reference

- Read `references/directory-structure.md` for file placement and architecture boundaries.
- Read `references/background-jobs.md` for BullMQ-style async processing and worker design.
- Read `references/docker-deployment.md` for container build/runtime patterns.
- Read `references/prisma-orm.md` for Prisma schema, client, and database-layer conventions.
- Read `references/google-auth.md` for implementing Google Auth

## Important Note
- During development, do not attempt to start the dev server for build the app. Ask the user to run the needed commands for starting server.
