# Organizations Feature Plan

## Overview

Allow users to be part of organizations that can share quizzes with one another. Organizations have three roles with distinct permissions, separate billing from personal accounts, and quiz search by ID or name.

---

## Roles

| Role | Manage Org | Create Org Quizzes | Edit Org Quizzes | Delete Org Quizzes | Host Org Quizzes | Personal Quizzes/Banks |
|------|------------|-------------------|-----------------|-------------------|-----------------|----------------------|
| **Admin** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (own subscription) |
| **Editor** | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ (own subscription) |
| **Host** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ (own subscription) |

**Key nuance**: A user's org role only governs what they can do with org-scoped resources. Personal quizzes and question banks remain governed entirely by the user's personal subscription, regardless of their org role.

---

## Data Model

### New Tables

#### `organizations`
```sql
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
name        text NOT NULL
slug        text NOT NULL UNIQUE  -- URL-friendly identifier
created_at  timestamptz NOT NULL DEFAULT now()
updated_at  timestamptz NOT NULL DEFAULT now()
-- Billing columns added later (e.g. stripe_customer_id)
```

#### `organization_members`
```sql
org_id      uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE
role        text NOT NULL  -- enum: admin | editor | host
invited_by  uuid REFERENCES users(id)  -- nullable, for audit
joined_at   timestamptz NOT NULL DEFAULT now()
PRIMARY KEY (org_id, user_id)
```

#### `org_invitations` (if using token-based invites)
```sql
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
org_id      uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE
email       citext NOT NULL
role        text NOT NULL
token       text NOT NULL UNIQUE
invited_by  uuid NOT NULL REFERENCES users(id)
expires_at  timestamptz NOT NULL
accepted_at timestamptz  -- nullable
```

#### `org_subscriptions` (billing scaffolding — Stripe wiring deferred)
```sql
id                      uuid PRIMARY KEY DEFAULT gen_random_uuid()
org_id                  uuid NOT NULL UNIQUE REFERENCES organizations(id)
plan                    text  -- e.g. monthly | yearly
status                  text  -- active | trialing | past_due | canceled | ...
current_period_end      timestamptz
stripe_customer_id      text UNIQUE
stripe_subscription_id  text UNIQUE
created_at              timestamptz NOT NULL DEFAULT now()
updated_at              timestamptz NOT NULL DEFAULT now()
```

### Modified Tables

#### `quizzes` — add nullable `org_id`
```sql
org_id  uuid REFERENCES organizations(id) ON DELETE SET NULL
```

- If `org_id` is null → personal quiz, governed by `owner_id`
- If `org_id` is set → org quiz, governed by org membership + role
- `owner_id` is retained on org quizzes as the creator, for audit purposes

---

## Permissions Logic

Permission checks happen in middleware/service layer based on the quiz's `org_id`:

- **Personal quiz**: `owner_id = current_user` → full CRUD
- **Org quiz, admin**: full CRUD + org management
- **Org quiz, editor**: create, edit, host (no delete)
- **Org quiz, host**: read quiz list + host games only

Entitlement checks for games hosted from an org quiz should use the **org's subscription** as the billing subject, not the individual user's personal subscription.

---

## API Surface

### Organization Endpoints
```
POST   /api/organizations                          Create org (any authenticated user)
GET    /api/organizations                          List orgs current user belongs to
GET    /api/organizations/:id                      Org detail (members only)
PATCH  /api/organizations/:id                      Update org (admin only)
DELETE /api/organizations/:id                      Delete org (admin only)

GET    /api/organizations/:id/members              List members
POST   /api/organizations/:id/members              Add/invite member (admin only)
PATCH  /api/organizations/:id/members/:userId      Change member role (admin only)
DELETE /api/organizations/:id/members/:userId      Remove member (admin only)

GET    /api/organizations/:id/quizzes              Search/list org quizzes (all members)
                                                   Query params: ?q=name or ?id=uuid
```

### Quiz Endpoints (extended, not duplicated)
Org quizzes flow through the existing `/api/quizzes` endpoints. The difference is that permission middleware checks org membership + role instead of `owner_id` equality when `org_id` is set on the quiz.

### Quiz Search Within Org
- Search by name: `ILIKE '%query%'` filtered by `org_id`
- Search by ID: exact UUID lookup filtered by `org_id`

---

## Billing Architecture

The existing `EntitlementChecker` interface needs to support two billing contexts:

- **Personal context**: user-level subscription (existing `subscriptions` table)
- **Org context**: org-level subscription (`org_subscriptions` table)

Options for the interface extension:
- Add an optional `orgID` parameter to `CanCreateGame` / `CanUseFeature`
- Or introduce a separate `OrgEntitlementChecker` interface

A `NoopOrgChecker` should be implemented first (same pattern as the existing noop), so the scaffolding exists without Stripe wiring. No handler refactors needed when real billing is added later.

---

## Invite Flow

Two options, in order of complexity:

1. **Simple (v1)**: Admin adds existing users directly by email. Lookup against the `users` table; error if not found.
2. **Token-based (v2)**: Admin sends invite link. Works for users who don't have an account yet. Requires the `org_invitations` table and an invite-acceptance flow.

Start with v1; upgrade to v2 when needed.

---

## Implementation Sequence

1. **Migrations** — `organizations`, `organization_members`, nullable `org_id` on `quizzes`, `org_subscriptions` scaffold. Low risk, no logic yet.
2. **Org CRUD + membership management** — admin-facing flows before any quiz work.
3. **Org quiz permissions** — extend quiz handlers to check org membership and role.
4. **Quiz search within org** — straightforward once data model is in place.
5. **Billing scaffolding** — add `org_subscriptions` table and noop org entitlement checker.
6. **Invite flow** — add-by-email first (v1), token-based invites later (v2).

---

## Open Questions

- **Invite UX**: Add-by-email directly (v1) or token-based invite links (v2) from the start?
- **Org billing model**: What plans/tiers will org accounts have? (Business model TBD)
- **Org quiz delete by editor**: Spec says editors can "create, edit, host" — confirming delete is admin-only.
- **Multiple orgs**: Can a user belong to more than one organization? (Current model supports it; worth confirming this is desired.)
- **Game attribution**: When a host runs an org quiz, should the resulting game record be attributed to the org or just the individual host?

---

## UX Improvements

- [ ] **Inline question creation in QuizBuilder** — allow hosts to add questions directly while building a quiz, without first navigating to a bank. The bank system stays for power users who want reusable libraries, but shouldn't be the only path for new users. Core to the "fastest way to go from idea to live quiz" positioning.
- [ ] **Replace `confirm()` dialogs with inline confirmations** — the native browser dialog looks out of place against the polished UI. Affects quiz delete (`QuizzesView`) and round delete (`QuizBuilder`).

---

## Monetization

- [ ] **Wire up personal Stripe billing** — `EntitlementChecker` interface and `subscriptions` table are already in place. Implement Stripe webhook handler to populate the table and swap the no-op billing stub for a real implementation. (Org billing is tracked separately above.)
- [ ] **Define free vs. paid tier limits** — suggested starting point: free tier gets 1 active game/month or a cap on saved quizzes; paid removes limits. Break-even at $0.99/mo is ~40 subscribers given current DOKS + load balancer costs.
- [ ] **Custom branding (paid tier)** — org-level and personal theming (logo, colors) for companies, pub trivia brands, and classrooms. Priority: before AV support. Natural fit alongside the Organizations feature.
- [ ] **Reliability as a paid differentiator** — consider surfacing session performance as a paid-tier selling point (e.g. no throttling on free tier, priority WebSocket handling). Hosts in front of a crowd will pay to avoid a broken trivia night.

---

## Known Gaps (Non-Billing)

### Critical
- **Auth0 frontend not wired up** — `@auth0/nextjs-auth0` is not installed; `lib/session.ts` always returns a hardcoded dev user. The backend JWT validation works correctly, but the Next.js frontend has zero authentication — anyone can access the host dashboard. Must be resolved before any real users.
- ~~**No deployment infrastructure**~~ — **RESOLVED (v1.1.0–v1.3.0).** Dockerfiles exist for both `api` and `web`; multi-platform images (`linux/amd64`, `linux/arm64`) are built and pushed to GHCR on every release. `docker-compose.yml` brings up the full local stack (postgres + redis + api + web). A Helm chart (`charts/quibble/`) is published to GitHub Pages via `chart-releaser`. GitHub Actions runs CI on every PR and semantic-release on `main`. Migrations are embedded in the API binary and run automatically on startup (`AUTO_MIGRATE=true`).

### High
- ~~**Game scores not persisted**~~ — **RESOLVED.** `onReleaseScores` in `realtime/hub.go` now calls `store.RecordAnswer` for every player answer after applying scores to `game_players`. Individual answer rows (question, player, answer text, correctness, points) are written to the `answers` table at the end of each round. The `GET /games/{gameID}/results` endpoint exposes these as a per-player breakdown, and the frontend Results page at `/games/[gameID]/results` links from the Games list for completed games.
- **Tests — partial coverage** — Go handler and unit tests exist for the `game` package (`internal/game/types_test.go`, `internal/game/service_test.go`). No Jest/Vitest config, no testing libraries in `package.json`, and no WebSocket flow coverage yet. Details:
  - **What's covered**: type conversion helpers (`nullText`, `bankFromStore`, `questionFromStore`, `gameFromStore`) and HTTP handlers for game CRUD (`listGames`, `getGame`, `cancelGame`, `listPlayers`, `gameResults`) — including auth (401), ownership (403), not-found (404), and conflict (409) cases.
  - **How it works**: `Service.q` and `Service.users` are interfaces (`querier`, `userResolver`). Tests use a `stubQuerier` that embeds the interface so uncalled methods panic loudly. Auth claims are injected via `auth.ContextWithClaims`. No database or network required.
  - **What's missing**: bank/question/quiz handler tests, realtime hub logic, frontend (no Jest/Vitest).
  - **Frontend WebSocket testing is now feasible** — `useGameSocket`, `useHostSocket`, and `usePlayerSocket` are extracted hooks (`apps/web/src/lib/`) that can be tested with a mock `ws` server without mounting any React tree. `HostGame` and `PlayerGame` are now thin UI shells; hook tests would cover the majority of game flow logic. Blocked only on adding Jest/Vitest + `@testing-library/react` to `package.json`.

### Medium
- **No email** — No SMTP, no templates, no email service in the backend. Needed for org invites (per the organizations plan) and eventually other notifications.
- **No QR codes** — Listed as planned in the README. Players must manually type the 6-character game code; a QR code on the host screen would significantly improve in-person use.
- **Thin frontend error handling** — No global `error.tsx` boundary, no `not-found.tsx` page, no loading skeletons. Some API failures silently show an empty state with no indication to the user.
- **No rate limiting** — Neither API endpoints nor WebSocket message handling have rate limiting.
- **CORS** — `Access-Control-Allow-Origin: *` in dev; needs to be locked down before production.

### Low
- **Accessibility** — `aria-label` attributes exist on buttons, but no ARIA live regions for real-time game updates (important for screen readers) and no keyboard navigation testing.
- **i18n** — All text is hardcoded English; no translation library or locale detection.
- **AV question support** — images, audio clips, and video embedded in question prompts. Nice differentiator but secondary to creation speed.

---

## Infrastructure Backlog

- [ ] Argo CD GitOps + Kustomize overlays (`base/`, `overlays/dev/`, `overlays/prod/`)
- [ ] GKE migration path (`overlays/gke/`) — swap DOKS → GKE Autopilot, Redis → Memorystore
