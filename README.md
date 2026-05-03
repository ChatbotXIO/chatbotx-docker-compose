<p align="center">
  <a href="https://github.com/ChatbotXIO/ChatbotX" target="_blank" rel="noopener">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset=".github/assets/readme/chatbotx-logo-dark.svg">
      <source media="(prefers-color-scheme: light)" srcset=".github/assets/readme/chatbotx-logo-light.svg">
      <img alt="ChatbotX Logo" src=".github/assets/readme/chatbotx-logo-light.svg" width="280">
    </picture>
  </a>
</p>

<p align="center">
  <a href="https://chatbotx.io">Website</a>
  ·
  <a href="https://chatbotx.io/docs">Docs</a>
  ·
  <a href="https://discord.chatbotx.io/">Discord</a>
</p>

## Watch the tutorial for Docker Compose install

Follow the **[Quick Start](https://chatbotx.io/docs/quickstart)** in the ChatbotX documentation to go from Docker Compose to a running workspace.

## Warning

If you are upgrading from an older ChatbotX version or Compose layout, verify that your compose file and environment variables match the current **[documentation](https://chatbotx.io/docs)** before starting services.


## Docker Compose

This guide assumes that you have Docker installed with enough resources to run ChatbotX (PostgreSQL with pgvector, Redis, RustFS, builder, worker, and realtime).

This Docker Compose setup has been tested with:

- Virtual Machine, Ubuntu 24.04, 2 GB RAM, 2 vCPUs (baseline; allocate more RAM for comfortable local use).

### Configuration uses environment variables

The containers here are configured with environment variables:

- **Option A** — Edit environment variables under the YAML anchors (`x-environment`, service blocks) in [`docker-compose.yml`](docker-compose.yml).
- **Option B** — Add a Compose override file (e.g. `docker-compose.override.yml`) layered on top of this project’s compose file.
- **Option C** — Put a `.env` file next to `docker-compose.yml` for Compose variable substitution (e.g. `POSTGRES_*`, `RUSTFS_*`, `ADMINER_PORT`). (Keep secrets out of Git.)

…or mix the above approaches.

Refer to **[ChatbotX documentation](https://chatbotx.io/docs)** for a full picture of installation, channels, and production settings.

Setup:

```
git clone https://github.com/ChatbotXIO/chatbotx-docker-compose.git
cd chatbotx-docker-compose
```

Then run:

```
docker compose up
```

Wait for the stack to become healthy, then open the app and tooling:

| Service    | URL                        |
|-----------|----------------------------|
| Builder   | http://localhost:3123       |
| Realtime  | http://localhost:1999       |
| RustFS    | http://localhost:9000 (S3-compatible API); console http://localhost:9001 |
| MailHog   | http://localhost:8025 (UI); SMTP http://localhost:1025 |
| Adminer   | http://localhost:8080       |
| Postgres  | `localhost:5432` (credentials from compose defaults) |
| Redis     | `localhost:6379`            |

---

## Example `docker-compose.yml` file

```yaml
x-network: &network
  networks:
    - main  

x-depends-datastores: &depends-datastores
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy
    filesystem:
      condition: service_healthy

x-environment: &common-vars
  BETTER_AUTH_SECRET: VLu1KcS63DTTnZU3Dh/ykI9Ld26YQXy/1IMJdZf/kHI=
  BETTER_AUTH_URL: http://localhost:3123
  DATABASE_URL: postgresql://chatbotx:secretkey@postgres:5432/chatbotx?schema:public
  NEXT_PUBLIC_ASSET_URL: http://localhost:9000/chatbotx/
  NEXT_PUBLIC_BUILDER_URL: http://localhost:3123
  NEXT_PUBLIC_EDITION: community
  NEXT_PUBLIC_ENVIRONMENT: dev
  NEXT_PUBLIC_SMTP_FROM: no-reply@localhost
  NEXT_PUBLIC_REALTIME_URL: http://localhost:1999
  NODE_OPTIONS: "--max_old_space_size=4096"
  REALTIME_API_KEY: secretkey
  REDIS_URL: redis://redis:6379
  S3_ACCESS_KEY_ID: chatbotx
  S3_BUCKET: chatbotx
  S3_ENPOINT: http://localhost:9000
  S3_REGION: us-west-2
  S3_SECRET_ACCESS_KEY: secretkey
  SCHEDULER_BUCKET_RANGE: 0-255
  SMTP_SERVER: smtp://username:password@localhost:1025
  RUN_DB_MIGRATE: true
  RUN_DB_SEED: true
  CLICKHOUSE_URL: http://clickhouse:8123
  CLICKHOUSE_USER: chatbotx
  CLICKHOUSE_PASSWORD: secretkey
  CLICKHOUSE_DB: chatbotx_analytics

x-worker: &worker
  <<: [ *network, *depends-datastores ]
  image: realcodesiman/chatbotx-worker:main
  platform: linux/amd64
  restart: unless-stopped
  extra_hosts:
    - "host.docker.internal:host-gateway"
  environment:
    <<: *common-vars

x-rustfs-environment: &rustfs-environment
  RUSTFS_ACCESS_KEY: ${RUSTFS_ACCESS_KEY:-chatbotx}
  RUSTFS_SECRET_KEY: ${RUSTFS_SECRET_KEY:-secretkey}

services:
  postgres:
    <<: *network
    image: pgvector/pgvector:pg18-trixie
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data
    command: [ "postgres", "-c", "log_statement=${POSTGRES_LOG_STATEMENT:-none}" ]
    environment:
      - POSTGRES_DB=${POSTGRES_DB:-chatbotx}
      - POSTGRES_USER=${POSTGRES_USER:-chatbotx}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-secretkey}
      - PGDATA=/var/lib/postgresql/data/pgdata
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}" ]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  adminer:
    <<: *network
    image: adminer:latest
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "${ADMINER_PORT:-8080}:8080"

  mailhog:
    <<: *network
    image: mailhog/mailhog
    ports:
      - 1025:1025
      - 8025:8025

  filesystem:
    <<: *network
    image: rustfs/rustfs:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      <<: *rustfs-environment
    volumes:
      - filesystem-data:/data
    healthcheck:
      test:
        [
          "CMD",
          "sh",
          "-c",
          "curl -f http://127.0.0.1:9000/health && curl -f
            http://127.0.0.1:9001/rustfs/console/health",
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    extra_hosts:
      - "host.docker.internal:host-gateway"

  filesystem-init:
    <<: *network
    image: rustfs/rc:latest
    depends_on:
      filesystem:
        condition: service_healthy
    environment:
      <<: *rustfs-environment
    entrypoint: >
      /bin/sh -c "rc alias set local http://filesystem:9000
      $${RUSTFS_ACCESS_KEY} $${RUSTFS_SECRET_KEY}; rc mb local/chatbotx
      --ignore-existing; touch /tmp/empty && rc cp /tmp/empty
      local/chatbotx/public/; rc anonymous set public local/chatbotx/public;"

  redis:
    <<: *network
    image: redis:8-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: [ "CMD", "redis-cli", "ping" ]
      interval: 10s
      timeout: 5s
      retries: 5

  builder:
    <<: [ *network, *depends-datastores ]
    image: realcodesiman/chatbotx-builder:main
    platform: linux/amd64
    restart: unless-stopped
    environment:
      <<: *common-vars
    ports:
      - "3123:3000"
    healthcheck:
      test: [ "CMD", "curl", "-f", "http://localhost:3000/api/health" ]
      interval: 30s
      timeout: 10s
      retries: 3
    extra_hosts:
      - "host.docker.internal:host-gateway"

  # BullMQ Worker
  worker:
    <<: [ *worker ]
    command: [ "worker", "all" ]

  realtime:
    <<: [ *network ]
    image: realcodesiman/chatbotx-realtime:latest
    platform: linux/amd64
    restart: unless-stopped
    environment:
      <<: *common-vars
    ports:
      - "1999:1999"
    extra_hosts:
      - "host.docker.internal:host-gateway"

networks:
  main:
    driver: bridge

volumes:
  db-data:
  redis-data:
  filesystem-data:
```

## About ChatbotX

ChatbotX is an open omnichannel chatbot stack for flows, AI agents, broadcasts, and integrations. Source and issue tracking live in the **[main ChatbotX repository](https://github.com/ChatbotXIO/ChatbotX)**.

## License

ChatbotX Community Edition is released under **[AGPL-3.0](https://github.com/ChatbotXIO/ChatbotX/blob/main/LICENSE)**.

This Docker Compose readme follows ChatbotX documentation patterns (tutorial and warnings upfront, Compose prerequisites, environment-variable options, clone/run workflow, and an embedded example `docker-compose.yml`).

