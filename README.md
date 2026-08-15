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

This guide assumes that you have Docker installed with enough resources to run ChatbotX (PostgreSQL/TimescaleDB, Redis, RustFS, builder, worker, javascript-executor, and realtime).

This repository runs the pre-built ChatbotX images published to `ghcr.io`. If you want to modify ChatbotX itself, use the **[main repository](https://github.com/ChatbotXIO/ChatbotX)** and its local development compose file instead.

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

| Service              | URL                        |
|----------------------|----------------------------|
| Builder              | http://localhost:3123       |
| Realtime             | http://localhost:1999       |
| RustFS               | http://localhost:9000 (S3-compatible API); console http://localhost:9001 |
| MailHog              | http://localhost:8025 (UI); SMTP http://localhost:1025 |
| Adminer              | http://localhost:8080       |
| Postgres             | `localhost:5432` (credentials from compose defaults) |
| Redis                | `localhost:6379`            |
| JavaScript Executor  | internal only — not published to the host; runs sandboxed user JS on an isolated, internal-only Docker network reachable solely from `worker` |

---

## The `docker-compose.yml` file

The full, current stack definition lives in **[`docker-compose.yml`](docker-compose.yml)** in this repository — that file is the source of truth, so it isn't duplicated here (a copy in this README would inevitably drift out of sync with it).

Notable details worth knowing before you edit it:

- All application images (`builder`, `worker`, `realtime`, `javascript-executor`) are pulled from `ghcr.io/chatbotxio/*:latest` — the most recent tagged release. See the file for exact tags.
- `javascript-executor` sandboxes untrusted, user-authored JavaScript flow steps. It sits on its own `internal: true` network with no host-published port, runs `read_only` with `cap_drop: ALL`, `no-new-privileges`, a small `noexec` tmpfs, and CPU/memory/PID limits. Only `worker` can reach it. Do not remove this isolation when customizing the file.
- Set `JAVASCRIPT_EXECUTOR_TOKEN` to a real secret (32+ characters) before running this in anything beyond local/throwaway environments.

## About ChatbotX

ChatbotX is an open omnichannel chatbot stack for flows, AI agents, broadcasts, and integrations. Source and issue tracking live in the **[main ChatbotX repository](https://github.com/ChatbotXIO/ChatbotX)**.

## License

This repository is released under the **[MIT License](LICENSE)**. The main ChatbotX application (see the **[main repository](https://github.com/ChatbotXIO/ChatbotX)**) is MIT-licensed as well, with enterprise features under a separate Commercial License.

This Docker Compose readme follows ChatbotX documentation patterns (tutorial and warnings upfront, Compose prerequisites, environment-variable options, and clone/run workflow).

