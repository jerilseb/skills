# Docker Deployment

Use this reference when containerizing a Nuxt application or changing production runtime behavior.

## Deployment Shapes

Nuxt projects commonly deploy as one of these:

- single app container running the built Nitro server
- app container plus a separate worker container
- app plus worker plus supporting services such as Postgres and Redis

Choose the smallest shape that matches the architecture. If the app uses queues, run the worker as a separate process.

## Image Strategy

A common pattern is:

1. Start from a Node base image.
2. Install only required OS packages.
3. Copy lockfiles first and install dependencies.
4. Copy source.
5. Run any build preparation steps.
6. Build Nuxt.
7. Start the built Nitro server from `.output/server/index.mjs`.

Use one image for both app and worker when they share the same runtime dependencies and source tree. Override the command per service.

## Example Dockerfile

Use this as a starting point for a Nuxt app that builds to Nitro and may also run a worker from the same image:

```dockerfile
FROM node:22-bookworm-slim

WORKDIR /app

RUN apt-get update -y && apt-get install -y openssl && rm -rf /var/lib/apt/lists/*

COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

COPY . .

RUN npm run postinstall
RUN npm run build

EXPOSE 3000

CMD ["node", ".output/server/index.mjs"]
```

Adjust:

- the base image version
- any OS packages required by Prisma or native dependencies
- the exposed port
- the startup command if the project uses pnpm, yarn, or a different worker entrypoint

## Compose Orchestration Pattern

For multi-service local or small-scale deployment, Compose works well with:

- `app`
- `worker`
- `postgres`
- `redis`

Preserve clear dependency ordering and health checks so app and worker do not start before core dependencies are ready.

## Example docker-compose.yml

Use this as a starting point for a Nuxt app with a separate worker, Postgres, and Redis:

```yaml
services:
  app:
    build:
      context: .
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      NODE_ENV: production
      HOST: 0.0.0.0
      PORT: 3000
      NITRO_HOST: 0.0.0.0
      DATABASE_HOST: postgres
      REDIS_HOST: redis
    command: sh -c "npm run db:push && node .output/server/index.mjs"
    ports:
      - "3000:3000"
    volumes:
      - app-storage:/app/data/storage

  worker:
    build:
      context: .
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      NODE_ENV: production
      DATABASE_HOST: postgres
      REDIS_HOST: redis
    command: sh -c "npm run db:push && npm run worker"
    volumes:
      - app-storage:/app/data/storage

  postgres:
    image: postgres:17
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DATABASE_NAME}
      POSTGRES_USER: ${DATABASE_USER}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DATABASE_USER} -d ${DATABASE_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 5s
    volumes:
      - redis-data:/data

volumes:
  app-storage:
  postgres-data:
  redis-data:
```

Adjust:

- the app port
- the schema startup command
- the worker command
- mounted storage paths
- whether Postgres and Redis are local Compose services or external managed services

## Environment Pattern

Keep environment handling consistent across:

- Nuxt runtime
- worker runtime
- Prisma CLI
- Docker and Compose files

Prefer one source of truth for parsing and validation in application code, then mirror only the required variables into deployment manifests.

## Database Startup Strategy

Be deliberate here:

- `prisma db push` is fast and acceptable for prototypes or early-stage internal apps.
- Prisma migrations are usually safer for mature production systems.

If both app and worker run schema synchronization on startup, ensure the approach is intentional and operationally safe.

## Persistence Pattern

If the app stores generated files locally:

- mount a shared volume for both app and worker
- keep path generation centralized in code
- avoid assuming container-local filesystems are durable across restarts

If the app uses object storage instead, preserve the same abstraction boundary in code.

## Lessons To Carry Forward

- Separate app and worker containers when they serve different latency and scaling needs.
- A single shared image reduces drift between app and worker runtimes.
- Health checks are worth keeping even in small Compose deployments because they remove startup races.
