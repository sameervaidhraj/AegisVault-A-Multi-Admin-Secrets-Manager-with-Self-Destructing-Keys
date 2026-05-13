# Project Aegis Full Detailed Documentation

## 1. Executive Overview

Project Aegis is a zero-trust secrets operations platform designed to secure sensitive credentials while preserving operational usability for security teams. The platform combines:

- role-based access control,
- threshold-based vault governance,
- Vault Transit encryption and decryption,
- dynamic and static secret lifecycle management,
- immutable hash-chained audit records,
- real-time incident visibility,
- and automatic protective actions (auto-seal).

Aegis is built as a full-stack system:

- Backend: FastAPI + SQLAlchemy + Redis + Celery + Vault client.
- Frontend: React + Vite + Tailwind + Axios + WebSockets.
- Infra: Docker Compose with API, worker, frontend, PostgreSQL, Redis, and Vault services.

The system is designed for secure secret handling under adversarial conditions, not only basic CRUD operations.

---

## 2. Problem Statement and Why This Project Matters

Most internal credential tools fail in one or more areas:

1. Secrets are stored in plaintext or weakly protected.
2. Operators cannot prove whether audit logs were tampered with.
3. Access controls are shallow, often lacking operational roles.
4. Incident response is manual and slow.
5. Secret retrieval does not enforce short-lived exposure.

Project Aegis addresses these gaps by introducing a security-first workflow where the system default is denial while sealed, and every sensitive action is constrained by policy, visibility, and expiry.

### 2.1 Primary security objectives

1. Enforce deny-by-default if vault gatekeeper is sealed.
2. Require threshold collaboration for unsealing (5 shares, threshold 3).
3. Ensure sensitive values are encrypted before database persistence.
4. Limit plaintext exposure to explicit, temporary retrieval windows.
5. Detect abnormal behavior and trigger protective containment.
6. Preserve tamper-evident audit chain integrity.
7. Stream security state to operations dashboard in real time.

---

## 3. Architecture in Depth

## 3.1 Logical architecture layers

1. Presentation layer
- React routes, UI components, operator workflows.

2. API layer
- FastAPI routers for auth, secrets, audit, alerts, vault.

3. Domain services layer
- Vault orchestration, secret lifecycle, audit chaining, alerts.

4. Persistence and cache layer
- PostgreSQL for durable records.
- Redis for TTL-sensitive and high-frequency state.

5. Security and control plane
- JWT auth, RBAC checks, rate limits, gatekeeper seal enforcement.

6. Async control and maintenance layer
- Celery scheduled jobs for lease expiry and audit chain checks.

## 3.2 Physical deployment architecture (Docker Compose)

- `api`: FastAPI service exposing REST and WebSockets.
- `worker`: Celery worker executing scheduled guard/reaper jobs.
- `frontend`: Vite dev server hosting UI.
- `db`: PostgreSQL for records.
- `redis`: in-memory data store for counters and TTL data.
- `vault`: HashiCorp Vault dev server for transit crypto and remote seal diagnostics.

Traffic model:

1. Browser to frontend on port 5173.
2. Frontend to API on port 8000 (`/api/*`).
3. Frontend to API WebSockets (`/ws/*`).
4. API to Redis, DB, and Vault over service network.
5. Worker to Redis and DB.

---

## 4. Core Security Model

Aegis intentionally separates two states:

1. Local application gatekeeper seal state.
2. Remote Vault server seal status.

This prevents false confidence where backend app logic is unsealed but remote vault state differs, or vice versa. The status payload includes both views.

## 4.1 Seal enforcement behavior

When gatekeeper is sealed:

- most HTTP routes return 503 with sealed message,
- only allowlisted paths remain accessible,
- WebSocket connections are closed with policy reason.

Allowlisted behavior exists to permit login and recovery operations.

## 4.2 Role-based permission model

Defined roles include:

- Admin
- Security Officer
- Auditor

Permission highlights:

- Vault init: Admin only.
- Vault unseal: Admin and Security Officer.
- Vault status: Admin, Security Officer, Auditor.
- Secret store/issue/view: Admin and Security Officer (list operations include Auditor).
- User listing: Admin and Security Officer.
- Audit/alerts read: Admin, Security Officer, Auditor.

This design enforces least privilege while still enabling audit visibility for non-admin security roles.

---

## 5. End-to-End Functional Workflows

## 5.1 Startup and warm-up

1. FastAPI app boots.
2. Database connectivity check executes.
3. Redis ping executes (best-effort).
4. Vault transit readiness check executes (best-effort).
5. Gatekeeper remains sealed until explicit unseal flow.

## 5.2 Login and access token issuance

1. Client posts username/email + password.
2. API applies Redis rate limit by client IP key.
3. On limit breach:
- create critical alert,
- trigger auto-seal.
4. Validate credentials against DB using bcrypt.
5. Update device fingerprint tracking.
6. Store access log entry.
7. Return access token + refresh token.

## 5.3 Vault initialization and unseal ritual

1. Admin calls vault init.
2. System derives key material and Shamir-splits into 5 fragments.
3. Fragments returned once for secure distribution.
4. Operators submit fragments with index.
5. Redis stores submitted fragments with TTL to support collaborative window.
6. After 3 unique valid fragments:
- gatekeeper transitions to unsealed,
- in-memory entered set and Redis fragment cache clear,
- unseal event is published.

## 5.4 Secret storage and retrieval

### Store static secret

1. User submits name, category, and plaintext.
2. Backend encrypts plaintext using Vault Transit.
3. Ciphertext stored in `stored_secrets` with metadata.

### Issue dynamic lease

1. User requests dynamic credential lease with TTL minutes.
2. Lease record created with status active.
3. Lease event pushed for realtime consumers.

### View stored secret

1. User requests secret view by ID.
2. Backend decrypts ciphertext via Vault Transit.
3. Generates view lease token and caches in Redis TTL.
4. Appends audit log entry with hash link.
5. Returns plaintext and countdown metadata.
6. Frontend starts countdown and clears revealed value when timer expires.

## 5.5 Audit chain verification and incident action

1. Every log entry stores `prev_hash` and computed `chain_hash`.
2. Verification job recomputes chain from genesis seed.
3. If mismatch found:
- publish alert event,
- trigger auto-seal with tamper metadata,
- return tamper detected signal.

## 5.6 Panic mode

1. Admin invokes panic.
2. All active leases marked revoked.
3. Gatekeeper sealed.
4. Subsequent operations blocked until unseal ritual repeats.

---

## 6. Backend Codebase Deep Explanation

## 6.1 Core application files

### `backend/main.py`
Responsibilities:

1. FastAPI app construction and router registration.
2. CORS middleware setup.
3. Global vault-seal enforcement middleware.
4. Startup initialization hooks.
5. Health endpoint.
6. WebSocket endpoints for audit/vault/leases/alerts.

Important behavior:

- Middleware checks non-allowlisted paths and blocks with 503 while sealed.
- WebSocket gate keeps channels unavailable while sealed.

### `backend/config.py`
Responsibilities:

1. Centralized environment-backed settings.
2. Security defaults and operational knobs.
3. Cache via `lru_cache` to avoid repeated parsing.

Contains:

- JWT settings
- DB and Redis URLs
- Vault parameters
- rate-limit thresholds
- secret and unseal TTLs
- CORS and WebSocket origins
- audit seed hash

### `backend/database.py`
Responsibilities:

1. Async engine and session maker creation.
2. DB liveness check on startup.
3. Dependency provider for route handlers.

### `backend/cache.py`
Responsibilities:

1. Shared Redis async client.
2. Dependency provider for route handlers and services.

### `backend/celery_app.py`
Responsibilities:

1. Celery app wiring with Redis broker/backend.
2. Serializer and startup behavior configuration.
3. Task auto-discovery across domains.

## 6.2 Authentication domain

### `backend/auth/router.py`
Endpoint details:

1. `POST /api/auth/login`
- applies rate limit,
- on hard abuse creates alert and triggers auto-seal,
- authenticates user,
- logs access and device,
- returns token pair.

2. `GET /api/auth/users`
- lists users with role and profile metadata,
- restricted to Admin and Security Officer.

### `backend/auth/service.py`
Core functions:

1. `hash_password` and `verify_password`
- bcrypt-based password lifecycle.

2. `_token_payload`, `issue_access_token`, `issue_refresh_token`
- JWT creation with role and expiry claims.

3. `authenticate_user`
- lookup by username OR email,
- increments failed attempts on bad password,
- resets failed attempts on success.

4. `log_access`
- records user/IP/agent/success outcome.

5. `register_device`
- inserts or updates fingerprinted device records.

6. `get_current_user`
- validates JWT and resolves DB user with role relation.

7. `require_roles`
- reusable RBAC dependency gate.

### `backend/auth/schemas.py`
Defines contract models for login request, token response, and user listing response.

## 6.3 Vault governance domain

### `backend/vault/router.py`
Endpoints:

1. `POST /api/vault/init`
2. `POST /api/vault/unseal`
3. `POST /api/vault/seal`
4. `GET /api/vault/status`
5. `POST /api/vault/panic`

### `backend/vault/service.py`
This is the most security-critical domain service.

Key structures:

1. `VaultStatus` dataclass
- standard status object (`sealed`, `progress`, `required`).

2. `VaultGatekeeper` class
- internal lock for concurrency safety,
- generated shares map,
- entered share index set,
- sealed flag.

3. `initialize`
- creates Shamir pieces,
- resets entered and seal state,
- clears fragment cache,
- publishes initialized event.

4. `submit_share`
- validates index/fragment correctness,
- records progress in memory + Redis,
- unseals once threshold reached,
- publishes unsealed event.

5. `seal`
- reseals, clears state and fragment cache, publishes sealed event.

6. `status`
- returns gatekeeper status and progress details.

7. `ensure_transit_ready`
- confirms transit secrets engine mount and key existence.

8. `encrypt_with_vault` and `decrypt_with_vault`
- wraps hvac transit operations,
- retries setup on missing path errors.

9. `read_vault_status`
- merges gatekeeper state with optional remote Vault diagnostics.

10. `trigger_auto_seal`
- central protective action for severe incidents.

## 6.4 Secrets management domain

### `backend/secrets/router.py`
Endpoints:

1. `POST /api/secrets/store`
2. `POST /api/secrets/issue`
3. `GET /api/secrets/leases`
4. `GET /api/secrets/static`
5. `POST /api/secrets/static/{id}/view`

### `backend/secrets/service.py`
Core operations:

1. `store_secret`
- transit encrypts secret and stores ciphertext record.

2. `issue_dynamic_secret`
- creates active lease with expiry and publishes lease event.

3. `fetch_leases` and `fetch_static_secrets`
- ordered retrieval helpers.

4. `issue_secret_view`
- decrypts secret,
- creates view token,
- stores token in Redis TTL,
- publishes event,
- returns plaintext + lease info.

5. `expire_overdue_leases`
- periodic status transitions from active to expired.

6. `panic_revoke`
- bulk revocation of active leases for emergency response.

### `backend/secrets/schemas.py`
Validation and shape models:

- store request constraints,
- issue request TTL bounds,
- lease read model,
- static secret read model,
- secret view response model.

### `backend/secrets/tasks.py`
Periodic task:

- scheduled lease reaper every 60 seconds.

## 6.5 Audit domain

### `backend/audit/router.py`
- `GET /api/audit/entries`
- returns logs plus `tamper_detected` boolean.

### `backend/audit/service.py`
Key functions:

1. `append_audit_log`
- computes chain hash from previous hash + payload.

2. `fetch_audit_logs`
- reverse chronological retrieval.

3. `verify_audit_chain`
- deterministic recompute of chain,
- on mismatch publishes alerts and triggers auto-seal.

### `backend/audit/tasks.py`
Periodic task:

- audit chain guard every 180 seconds.

## 6.6 Alerts and shared services domain

### `backend/alerts/router.py`
- `GET /api/alerts` authorized listing.

### `backend/services/alerts.py`
- alert creation utility and real-time publish.

### `backend/services/rate_limiter.py`
- Redis counter + TTL window logic.
- raises HTTP 429 with retry metadata.

### `backend/services/events.py`
- in-process channel-to-websocket fan-out.
- connect/disconnect/publish primitives.

### `backend/services/crypto.py`
Contains three groups:

1. AES-GCM helpers for payload encryption utilities.
2. Shamir share split/combine helpers.
3. Hash-chain function for audit integrity.

## 6.7 Data model layer

### `backend/models/user.py`
Entities:

- `Role`
- `User`
- `Device`

Model purpose:

- identity, role assignment, and endpoint device telemetry.

### `backend/models/secret.py`
Entities:

- `StoredSecret`
- `SecretLease`

Model purpose:

- encrypted static secret records and dynamic lease ledger.

### `backend/models/audit.py`
Entities:

- `AuditLog`
- `AccessLog`

Model purpose:

- tamper-evident action log and authentication access trail.

### `backend/models/alert.py`
Entity and enums:

- `Alert`
- `AlertSeverity`
- `AlertStatus`

Model purpose:

- evented security signal lifecycle.

### `backend/models/base.py`
- SQLAlchemy base class + timestamp mixin.

## 6.8 Migrations and scripts

### `backend/migrations/versions/202403251200_initial.py`
Creates all foundational tables for roles, users, devices, secrets, leases, audit, access, and alerts.

### `backend/migrations/env.py`
Loads runtime DB URL from app settings to keep Alembic aligned with deployment configuration.

### `backend/scripts/create_admin.py`
Creates role and user safely via CLI with optional generated password.

---

## 7. Frontend Codebase Deep Explanation

## 7.1 App bootstrap and navigation

### `frontend/src/main.tsx`
- React root mount and router activation.

### `frontend/src/App.tsx`
- route table:
  - `/login`
  - `/`
  - `/secrets`
  - `/audit`
  - `/access`
  - `/vault`

### `frontend/src/components/Layout.tsx`
Responsibilities:

1. Side navigation rendering.
2. Route guarding via local token presence.
3. Redirecting unauthenticated users to login.

## 7.2 Frontend API and realtime utilities

### `frontend/src/services/api.ts`
Responsibilities:

1. Axios instance with backend base URL.
2. Request interceptor injecting `Authorization: Bearer <token>`.
3. Typed wrapper methods for Auth, Secret, and Vault APIs.
4. Aggregate dashboard fetch for core cards.

### `frontend/src/services/websocket.ts`
Responsibilities:

1. generic channel factory,
2. parse incoming JSON payloads,
3. listener subscription management,
4. channel close lifecycle.

## 7.3 Page-by-page logic

### `frontend/src/pages/Login.tsx`
- form validation,
- calls login API,
- stores access and refresh tokens,
- redirects to prior protected route.

### `frontend/src/pages/Dashboard.tsx`
- initial fetch for leases/alerts/audit/vault status,
- subscribes to `/ws/audit`, `/ws/alerts`, `/ws/vault`, `/ws/leases`,
- updates card metrics in response to channel events,
- displays both gatekeeper and remote Vault state.

### `frontend/src/pages/SecretsManager.tsx`
Contains three UX sections:

1. static secret storage form,
2. dynamic credential issuing,
3. static secret reveal panel with lease countdown.

Also renders lease ledger table.

### `frontend/src/pages/AuditTrail.tsx`
- retrieves audit chain state and entries,
- displays chain health indicator and timeline.

### `frontend/src/pages/AccessControl.tsx`
- fetches authorized user list and role metadata table.

### `frontend/src/pages/VaultControl.tsx`
- shows seal status and progress,
- allows seal and panic actions,
- embeds unseal fragment component,
- supports generating new 5-share set.

## 7.4 Component-level logic

### `frontend/src/components/VaultStatus.tsx`
- fetches status on mount,
- validates share form input,
- submits fragment,
- refreshes status,
- reports status to parent via callback.

### `frontend/src/components/SecretsForm.tsx`
- collects static secret fields and stores secret.

### `frontend/src/components/LeaseTable.tsx`
- presents lease records and active/expired/revoked status style.

### `frontend/src/components/AlertsFeed.tsx`
- severity-colored alert list.

### `frontend/src/components/AuditTimeline.tsx`
- visual chronological action markers and tamper flags.

### `frontend/src/components/ChartCard.tsx`
- lease velocity area chart using Recharts.

### `frontend/src/components/Card.tsx`
- reusable card shell for consistent panel layout.

### `frontend/src/index.css`
- Tailwind directives and basic typography/theme defaults.

---

## 8. API Endpoint Behavior Reference

## 8.1 Authentication

1. `POST /api/auth/login`
- input: username, password, optional device fingerprint.
- output: access token, refresh token, token type, expiry seconds.
- protections: rate-limit, RBAC on follow-up protected routes.

2. `GET /api/auth/users`
- output: user identity and role metadata.
- roles: Admin, Security Officer.

## 8.2 Vault

1. `POST /api/vault/init`
- output: array of shares and threshold value.
- role: Admin.

2. `POST /api/vault/unseal`
- input: index + fragment.
- output: seal status and progress.
- roles: Admin, Security Officer.

3. `POST /api/vault/seal`
- output: local sealed + remote seal status.
- role: Admin.

4. `GET /api/vault/status`
- output: gatekeeper status and optional remote diagnostics.
- roles: Admin, Security Officer, Auditor.

5. `POST /api/vault/panic`
- output: revoked count + sealed state.
- role: Admin.

## 8.3 Secrets

1. `POST /api/secrets/store`
- stores encrypted static secret.

2. `POST /api/secrets/issue`
- issues dynamic lease.

3. `GET /api/secrets/leases`
- lists lease records.

4. `GET /api/secrets/static`
- lists static secret metadata.

5. `POST /api/secrets/static/{id}/view`
- decrypts and returns secret with temporary view lease.

## 8.4 Audit and alerts

1. `GET /api/audit/entries`
- returns logs and tamper flag.

2. `GET /api/alerts`
- returns alerts ordered newest first.

## 8.5 WebSocket endpoints

1. `/ws/audit`
2. `/ws/vault`
3. `/ws/leases`
4. `/ws/alerts`

All are unavailable while gatekeeper is sealed.

---

## 9. Data Design and Storage Strategy

## 9.1 PostgreSQL entities and purpose

- roles: authorization taxonomy.
- users: identities and auth attributes.
- devices: per-user fingerprint/IP history.
- stored_secrets: encrypted secret records.
- secret_leases: issued lease lifecycle.
- audit_logs: immutable chain-linked event log.
- access_logs: login and access outcomes.
- alerts: security signal records.

## 9.2 Redis key classes

1. rate-limit keys
- counter and TTL for abuse prevention.

2. unseal fragment hash key
- share index to fragment with expiring ritual window.

3. secret view token keys
- maps ephemeral token to secret ID with short TTL.

## 9.3 Why split DB and Redis duties

- DB handles durable, queryable history.
- Redis handles high-churn and TTL-sensitive control data.
- separation improves speed, correctness, and recovery behavior.

---

## 10. Real-Time Eventing Model

Aegis uses an in-process pub/sub broker to push security events to dashboards.

Channels:

1. `audit`
- published after audit append or integrity events.

2. `vault`
- published on init, unseal, seal, auto-seal, and secret view-related vault events.

3. `leases`
- published on lease issue and panic revocation.

4. `alerts`
- published on alert creation and audit tamper notifications.

Frontend subscribes and updates cards/tables incrementally for operator situational awareness.

---

## 11. Background Tasks and Maintenance Jobs

## 11.1 Lease reaper task

- schedule: every 60 seconds.
- action: mark overdue active leases as expired.
- benefit: keeps lease ledger accurate without user interaction.

## 11.2 Audit chain guard task

- schedule: every 180 seconds.
- action: recompute and verify audit hash chain.
- failure action: publish tamper event and auto-seal.

---

## 12. Failure Scenarios and Recovery Behavior

## 12.1 Redis unavailable

Potential effects:

- rate limiting degraded,
- unseal progress persistence degraded,
- secret view token cache unavailable.

Current behavior:

- some operations fall back gracefully with warnings,
- system remains mostly operational but with reduced safeguards.

## 12.2 Vault unavailable

Potential effects:

- cannot encrypt/decrypt secrets,
- status remote diagnostics may be null.

Current behavior:

- encryption/decryption requests fail,
- status endpoint still provides gatekeeper perspective.

## 12.3 Audit chain mismatch

Behavior:

1. tamper alert event,
2. auto-seal,
3. dashboard reflects incident state.

## 12.4 Rate-limit abuse

Behavior:

1. HTTP 429 with retry metadata,
2. brute-force alert creation,
3. auto-seal triggered.

---

## 13. Security Strengths and Gaps

## 13.1 Strengths

1. Gatekeeper-driven deny-by-default model.
2. Threshold unseal requiring collaboration.
3. Vault Transit encryption path.
4. Immutable chain-like audit integrity model.
5. Rate-limit abuse tied to immediate containment.
6. Short-lived secret view tokens and UI purge behavior.

## 13.2 Current limitations

1. Vault service is configured in dev mode in compose.
2. Event bus is process-local and not cluster-shared.
3. Frontend route guard checks token presence, not full claim freshness.
4. MFA route exists in allowlist but full two-step flow is not currently active in login path.

---

## 14. Deployment and Operations Runbook

## 14.1 First-time local bring-up

1. copy env template to `.env`.
2. run `docker compose up --build -d`.
3. run migrations: `docker compose exec api alembic upgrade head`.
4. create admin user if required using create_admin script.
5. login via UI.
6. initialize shares and unseal with threshold fragments.

## 14.2 Daily operational routine

1. verify API health endpoint.
2. verify vault status card (gatekeeper and remote).
3. review alerts feed and audit tamper flag.
4. monitor lease activity and expirations.

## 14.3 Incident handling routine

1. confirm alert severity and context.
2. if suspicious compromise risk, execute panic mode.
3. inspect audit chain logs and access logs.
4. reseal and re-unseal with controlled share holders.
5. rotate exposed credentials outside this system as needed.

---

## 15. Environment Variable Reference

High-impact backend controls:

- `APP_SECRET_KEY`
- `DATABASE_URL`
- `REDIS_URL`
- `VAULT_ADDR`
- `VAULT_TOKEN`
- `VAULT_TRANSIT_KEY`
- `VAULT_KEY_SHARES`
- `VAULT_KEY_THRESHOLD`
- `UNSEAL_SHARE_TTL_SECONDS`
- `SECRET_VIEW_TTL_SECONDS`
- `RATE_LIMIT_REQUESTS`
- `RATE_LIMIT_WINDOW_SECONDS`
- `AUDIT_SEED_HASH`

Frontend controls:

- `VITE_API_URL`
- `VITE_WS_URL`

Operational note:

- changing thresholds and TTL values directly changes security posture and operator workflow friction.

---

## 16. Testing and Quality Strategy (Current and Recommended)

## 16.1 Current observed state

- Build and compile checks are available.
- Full automated test coverage is not yet present.

## 16.2 Recommended test layers

1. Unit tests
- auth service, vault gatekeeper, secrets service, audit hashing.

2. Integration tests
- API route behavior with role-based tokens.

3. Security tests
- rate-limit threshold behavior, auto-seal triggers, expired token behavior.

4. End-to-end tests
- login, unseal, store secret, view secret, panic mode.

---

## 17. Scaling and Production Hardening Recommendations

1. Replace Vault dev mode with HA Vault and secure auth method.
2. Externalize event bus to Redis pub/sub or Kafka for multi-instance API.
3. Add token rotation and revocation list strategy.
4. Introduce centralized observability stack (metrics, tracing, logs).
5. Add secret rotation workflows and key lifecycle automation.
6. Add stronger frontend session management and silent refresh handling.
7. Introduce formal DR and backup procedures for DB and critical config.

---

## 18. Code Mapping Index (Quick Lookup)

Backend core:

- `backend/main.py`
- `backend/config.py`
- `backend/database.py`
- `backend/cache.py`
- `backend/celery_app.py`

Backend domains:

- `backend/auth/router.py`
- `backend/auth/service.py`
- `backend/auth/schemas.py`
- `backend/vault/router.py`
- `backend/vault/service.py`
- `backend/secrets/router.py`
- `backend/secrets/service.py`
- `backend/secrets/schemas.py`
- `backend/audit/router.py`
- `backend/audit/service.py`
- `backend/alerts/router.py`

Backend shared:

- `backend/services/alerts.py`
- `backend/services/rate_limiter.py`
- `backend/services/events.py`
- `backend/services/crypto.py`

Data and maintenance:

- `backend/models/base.py`
- `backend/models/user.py`
- `backend/models/secret.py`
- `backend/models/audit.py`
- `backend/models/alert.py`
- `backend/migrations/env.py`
- `backend/migrations/versions/202403251200_initial.py`
- `backend/secrets/tasks.py`
- `backend/audit/tasks.py`
- `backend/scripts/create_admin.py`

Frontend app:

- `frontend/src/main.tsx`
- `frontend/src/App.tsx`
- `frontend/src/index.css`

Frontend services:

- `frontend/src/services/api.ts`
- `frontend/src/services/websocket.ts`

Frontend pages and components:

- `frontend/src/pages/Login.tsx`
- `frontend/src/pages/Dashboard.tsx`
- `frontend/src/pages/SecretsManager.tsx`
- `frontend/src/pages/AuditTrail.tsx`
- `frontend/src/pages/AccessControl.tsx`
- `frontend/src/pages/VaultControl.tsx`
- `frontend/src/components/Layout.tsx`
- `frontend/src/components/Card.tsx`
- `frontend/src/components/VaultStatus.tsx`
- `frontend/src/components/SecretsForm.tsx`
- `frontend/src/components/LeaseTable.tsx`
- `frontend/src/components/AlertsFeed.tsx`
- `frontend/src/components/AuditTimeline.tsx`
- `frontend/src/components/ChartCard.tsx`

---

## 19. Plain Language Final Summary

Project Aegis is not just a secret store. It is a controlled operational security system that:

1. prevents sensitive operations unless trust conditions are met,
2. requires collaborative unseal control,
3. encrypts and limits secret exposure duration,
4. continuously checks log integrity,
5. can automatically shift to protective sealed mode during suspicious events,
6. and gives real-time visibility to security operators.

This design reduces blast radius, improves incident responsiveness, and establishes stronger accountability for secrets operations.
