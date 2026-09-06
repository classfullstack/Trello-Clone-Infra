# Trello Clone — Infra

Docker Compose stack: edge gateway, 3 frontends, api, postgres, redis, minio — plus an optional monitoring overlay (Prometheus/Grafana/Loki/Tempo/Alertmanager).

> Dev VPS has only an IP (no DNS/TLS) — apps are reachable directly by **host port**. Prod sits behind Cloudflare with the `gateway` nginx routing by **subdomain** (see below) and TLS termination.

## Repo layout

All repos must be **siblings** in the same parent dir:

```
trello-clone/
├── Trello-Clone-Frontend/
├── Trello-Clone-Backend/
└── Trello-Clone-Infra/   # run compose from here
```

## Quick start

```bash
cp .env.example .env
make dev       # build + start all services
make health    # check api + all 3 frontends
```

## Services & Ports

| Service        | Host Port   | Notes |
|----------------|-------------|-------|
| gateway        | 80 / 443    | nginx edge proxy, routes by subdomain (Prod, behind Cloudflare) |
| frontend-user  | 8080        | user SPA; nginx proxies `/api/` + `/socket.io/` -> api |
| frontend-admin | 8081        | admin SPA; nginx proxies `/api/` + `/socket.io/` -> api |
| landing        | 3000        | Next.js standalone (no api proxy) |
| api            | 4000        | built from ../Trello-Clone-Backend |
| postgres       | (internal)  | backend network only |
| redis          | (internal)  | backend network only |
| minio          | 9000 / 9001 | S3 api / console (minioadmin/minioadmin) |

SPAs call same-origin `/api` and `/socket.io`; each app's nginx proxies them to
`http://api:4000` over the `backend` network. `createbuckets` runs once to
create the `trello` bucket.

## Edge gateway subdomain routing (`nginx/gateway.conf`)

| Subdomain | Target | Notes |
|---|---|---|
| `app.*` | frontend-user | main app |
| `admin.*` | frontend-admin | admin console |
| (root) | landing | marketing site, `default_server` |
| `media.*` | minio:9000 | Host header preserved so presigned URLs match |
| `bot.*` | api `/api/` | own Let's Encrypt cert, DNS-only (bypasses Cloudflare) |
| `internal.*` | grafana:3000 | Prod-only observability access |

Wildcard cert covers all subdomains except `bot.*`, which uses its own Let's Encrypt certificate.

## Make targets

- `make dev` — build + start (detached)
- `make down` — stop and remove
- `make prod` — currently identical to `make dev`; add a real prod override before relying on it
- `make migrate` — `prisma migrate deploy` inside api
- `make seed` — `npm run seed` inside api
- `make health` — check health endpoint
- `make logs` — follow logs

## Monitoring overlay (optional)

```bash
cp .env.monitoring.example .env.monitoring
docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d
```

Adds Prometheus (scrapes `api:4000/metrics` + node/postgres/redis exporters), Grafana (bound to `127.0.0.1:3001` only — never expose directly), Loki + Promtail (logs), Tempo (traces), Alertmanager (→ Telegram). Configs under `monitoring/`.

## Networks

- `frontend`: gateway + frontends + api (public-facing tier)
- `backend`: api + postgres + redis + minio + gateway + frontends + monitoring stack (so everything can reach `api:4000`, `postgres`, etc.)

Neither network is marked `internal`; Postgres/Redis/MinIO are simply never published to the host.
