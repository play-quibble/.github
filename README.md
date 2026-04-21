# Quibble — Project Brief

## Goal

Self-hosted Kubernetes trivia app where hosts create quizzes and players join via QR code. Built on DOKS initially, with a planned migration path to GKE. Doubles as a k8s learning project. Subscription pricing ($0.99/mo, $9.99/yr) is planned but deferred — the architecture should make it easy to plug in later.

## Stack

- **Backend:** Go (Chi or Echo router, gorilla/websocket or coder/websocket, sqlc + pgx, slog for structured logging, goose or golang-migrate for migrations)
- **Frontend:** Next.js (App Router, @auth0/nextjs-auth0, native browser WebSocket)
- **Database:** Neon Postgres (use branching: `main` for prod, `dev` for development)
- **Cache/Realtime state:** Redis (in-cluster StatefulSet)
- **Auth:** Auth0 (SPA app for Next.js, API app for Go backend's audience; players join unauthenticated via game code + display name)
- **Payments (deferred):** Stripe Billing
- **Hosting:** DigitalOcean Kubernetes (DOKS)
- **Deployment:** Helm chart (`charts/quibble/`) published via chart-releaser to GitHub Pages; Argo CD + Kustomize planned for later GitOps workflow
- **Observability:** kube-prometheus-stack (Prometheus + Grafana + Alertmanager), optionally Loki for logs
- **Ingress:** ingress-nginx + cert-manager (Let's Encrypt)
- **Secrets:** External Secrets Operator or sealed-secrets

## Repo Structure (monorepo)

```
trivia/
  apps/
    api/                    # Go backend
      cmd/server/
      internal/
        auth/               # Auth0 JWT validation
        billing/            # STUB — interface only for now
        bootstrap/          # sample-data seeder (runs at startup when BOOTSTRAP_SAMPLES=true)
        config/             # env-based config
        game/               # game lifecycle, questions, scoring
        migrate/            # goose wrapper; migrations embedded in binary
        realtime/           # WebSocket hub, room management
        store/              # sqlc-generated DB code
        user/               # host accounts
      migrations/           # .sql files (embedded via go:embed)
      queries/              # sqlc source queries
    web/                    # Next.js frontend
  charts/
    quibble/                # Helm chart (API + web + Redis); published via chart-releaser
  docker-compose.yml        # Local full-stack: postgres + redis + api + web
  docs/
```

## Data Model (Postgres)

Migrations are managed with goose and embedded directly into the API binary. The API runs all pending migrations on startup when `AUTO_MIGRATE=true` (default in docker-compose and the Helm chart). A separate migrate job is no longer needed.

- `users` — host accounts, Auth0 `sub` as foreign identity
- `question_banks` — reusable question sets owned by a host
- `questions` — belongs to a bank; `type` enum (`text`, `multiple_choice`), `prompt`, `correct_answer`, `choices` (JSONB, nullable), `points`
- `games` — a play instance; `code` (6-char short code), `status`, `host_id`, `current_question_idx`, `started_at`
- `game_players` — display name, join time, score
- `answers` — one row per player per question for post-game review
- `subscriptions` — **create table now, leave empty**: `user_id`, `plan`, `status`, `current_period_end`, `stripe_customer_id`, `stripe_subscription_id`

## Data Model (Redis — ephemeral live state)

- `game:{code}:state` — current question, phase (lobby/question/reveal/scoreboard), deadline
- `game:{code}:players` — set of connected players with display names
- `game:{code}:answers:{qid}` — hash of player → answer
- `game:{code}:scores` — sorted set for live leaderboard

Flush final scores to Postgres on game end; Redis stays ephemeral.

## Realtime Design

- One WebSocket hub per API pod, rooms keyed by game code
- Message directions:
  - Host → server: start game, advance question, end game
  - Player → server: join, submit answer
  - Server → room: question revealed, timer tick, answer accepted, scoreboard update, game ended
- Start single-replica (no cross-pod coordination). Scale out later with Redis Pub/Sub fan-out — a clean additive change, not a rewrite.
- Sticky sessions on ingress simple at first; not needed once Pub/Sub is in place.

## Subscription-Readiness (deferred implementation)

Define billing as an interface now, implement a no-op version that always grants access:

```go
type EntitlementChecker interface {
    CanCreateGame(ctx context.Context, userID string) (bool, error)
    CanUseFeature(ctx context.Context, userID, feature string) (bool, error)
}
```

Gated handlers call this. Later, swap in a Stripe-webhook-backed implementation reading from the `subscriptions` table. No refactor needed at call sites.

## Kubernetes Layout

**Current approach: Helm chart** (`charts/quibble/`)

The chart bundles all workloads and is published automatically to GitHub Pages via `chart-releaser` on every push to `main` that touches `charts/**`. Install with:

```
helm repo add quibble https://benbotsford.github.io/trivia
helm install quibble quibble/quibble -f my-values.yaml
```

**Workloads in the chart:**

- `api` Deployment (1 replica; HPA planned later)
- `web` Deployment
- `redis` StatefulSet with PVC
- Ingress: `/api/*` → api service, else → web (ingress-nginx, cert-manager TLS)
- Secret manifest for `DATABASE_URL` and optional Redis password

**Platform (not bundled — install separately):** ingress-nginx, cert-manager, kube-prometheus-stack, optionally Loki

**GitOps (planned):** Argo CD app-of-apps with Kustomize overlays (`base/`, `overlays/dev/`, `overlays/prod/`, `overlays/gke/`). Not yet implemented — the Helm chart is the current deployment artifact.

## Day-One Observability

- Prometheus metrics in Go via `promhttp` — request counts/durations, active WebSocket connections, games in progress, questions answered per minute
- Structured JSON logging with `slog` (Loki-friendly)
- Readiness probe checks DB + Redis; liveness checks process health

## CI/CD

- **CI** (`ci.yml`): lint + build for both `api` (Go) and `web` (Next.js) on every push/PR
- **Release** (`release.yml`): semantic-release on `main`; builds and pushes multi-platform Docker images (`linux/amd64`, `linux/arm64`) to GHCR (`ghcr.io/benbotsford/trivia-api` and `trivia-web`); bumps `CHANGELOG.md`
- **Chart release** (`chart-release.yml`): on push to `main` when `charts/**` changes, runs `helm/chart-releaser-action` to package and publish the Helm chart to GitHub Pages
- Image tags follow semver from semantic-release; chart `appVersion` tracks the app release

## Migration Path to GKE

Swap: DOKS → GKE Autopilot/Standard, in-cluster Redis → Memorystore, Neon stays (or → Cloud SQL). Manifests stay almost identical — add `overlays/gke/` patches. Argo CD points at new cluster and reconstructs everything.

## Build Sequence

1. ✅ Data model + migrations + sqlc generation
2. ✅ Basic Go API: question banks, games (HTTP only)
3. ✅ Next.js host dashboard with Auth0 stub, hitting those endpoints
4. ✅ Docker images (multi-platform), docker-compose local stack, Helm chart + CI/CD pipeline
5. WebSocket hub + player join flow
6. Full game loop: questions, answers, scoring, reveal
7. QR generation + join page
8. Wire up real Auth0 in the Next.js frontend
9. Persist final game scores to Postgres
10. Observability: kube-prometheus-stack + structured logging
11. *(Later)* Argo CD GitOps, Kustomize overlays, GKE migration path
12. *(Later)* Stripe + billing implementation + paywall

## Economics Note

At $0.99/mo, break-even on a $24/mo DOKS cluster + $12/mo load balancer is ~40 paid subscribers. GKE Autopilot raises that floor. Consider a free tier (e.g., 1 game/month) to lower the conversion barrier when billing goes live.
