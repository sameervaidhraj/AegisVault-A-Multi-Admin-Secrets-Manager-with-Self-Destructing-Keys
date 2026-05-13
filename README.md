# Project Aegis - Secrets Management & Vault Platform

A comprehensive secrets management platform built with FastAPI and React, featuring HashiCorp Vault integration, tamper-proof audit logging, dynamic secret leasing, and real-time monitoring.

## Overview

Project Aegis provides enterprise-grade secrets management with advanced security features including zero-trust architecture, role-based access control, multi-factor authentication, and cryptographic key management. The platform ensures that all sensitive data is encrypted, securely stored, and audited for compliance.

## Key Features

### Security & Access Control
- **Multi-Factor Authentication (MFA)**: TOTP-based 2FA for enhanced user security
- **Role-Based Access Control (RBAC)**: Admin, Security Officer, and Auditor roles
- **JWT Token Management**: Secure session handling with access and refresh tokens
- **Device Fingerprinting**: Track and authenticate user devices

### Secrets Management
- **Dynamic Leasing**: Automatically expire secrets with configurable TTLs
- **Envelope Encryption**: Vault Transit integration for encryption
- **Secret Storage**: Secure database storage with encryption
- **View Leases**: Time-limited plaintext access with countdown timers

### Vault Operations
- **Shamir Key Sharing**: Distribute master key into 5 fragments, require 3 to unseal
- **Multi-step Initialization**: Secure initialization ritual with distributed shares
- **Panic Mode**: Emergency seal with lease revocation
- **Cold Start Guard**: Sealed vault returns 503 for all operations except login/unseal

### Audit & Monitoring
- **Hash-Chained Logging**: Tamper-proof audit trail with SHA-256 verification
- **Real-time Alerts**: WebSocket streaming of security events
- **Access Tracking**: Monitor user actions and secret access patterns
- **Rate Limiting**: Redis-backed brute-force protection

### Dashboard UI
- Vault status visualization
- Secret management interface
- Audit timeline with detailed logs
- Access control management
- Live alert feed
- Lease velocity charts

## Tech Stack

| Layer | Technology |
|-------|------------|
| **Backend** | FastAPI, SQLAlchemy ORM, Celery, Alembic migrations |
| **Database** | PostgreSQL (production) / SQLite (dev) |
| **Frontend** | React 18+, TypeScript, Vite, Tailwind CSS |
| **Caching/Queue** | Redis, Celery workers |
| **Security** | HashiCorp Vault, Passlib (bcrypt), PyOTP |
| **Real-time** | WebSockets for audit/alerts/vault status |
| **Deployment** | Docker & Docker Compose |

## Architecture

```
Project Aegis
├── Backend (FastAPI)
│   ├── /auth          - Login, MFA, JWT token management
│   ├── /vault         - Vault initialization, unsealing, status
│   ├── /secrets       - Secret CRUD, leasing, encryption
│   ├── /audit         - Audit log retrieval, verification
│   ├── /alerts        - Alert streaming and notifications
│   └── /services      - Crypto, rate limiting, event handlers
│
├── Frontend (React)
│   ├── Dashboard      - Vault & system overview
│   ├── Secrets Manager - Create, view, manage secrets
│   ├── Audit Trail    - View and verify audit logs
│   ├── Access Control - Manage user roles and permissions
│   └── Vault Control  - Initialize, unseal, seal vault
│
├── Core Services
│   ├── PostgreSQL     - Persistent data storage
│   ├── Redis          - Caching, rate limiting, session state
│   ├── Vault          - Key management & encryption
│   └── Celery         - Asynchronous task processing
```

## Prerequisites

- **Docker & Docker Compose** (for containerized deployment)
- **Node.js 18+** (for frontend development)
- **Python 3.11+** (for backend development)
- **PostgreSQL 13+** (for production database)

## Quick Start

### Using Docker Compose (Recommended)

```bash
# 1. Clone and configure environment
cp .env.example .env
# Edit .env with your settings if needed

# 2. Build and start all services
docker compose up --build

# 3. Run database migrations
docker compose exec api alembic upgrade head

# 4. Create admin user (optional)
docker compose exec api python backend/scripts/create_admin.py
```

**Access the application:**
- Frontend UI: http://localhost:5173
- API Docs: http://localhost:8000/docs
- Vault UI: http://localhost:8200 (token in .env)

### Manual Backend Setup

```bash
cd backend

# Create and activate virtual environment
python -m venv .venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # Unix/macOS

# Install dependencies
pip install -r requirements.txt

# Run migrations
alembic upgrade head

# Start the server
uvicorn backend.main:app --reload
```

The API will be available at http://localhost:8000

### Manual Frontend Setup

```bash
cd frontend

# Install dependencies
npm install

# Start development server
npm run dev
```

The UI will be available at http://localhost:5173

## Configuration

Create a `.env` file in the project root (copy from `.env.example`):

```env
# Database
DATABASE_URL=postgresql://user:password@localhost/aegis

# Vault
VAULT_ADDR=http://localhost:8200
VAULT_TOKEN=your-token-here

# Redis
REDIS_URL=redis://localhost:6379

# JWT
SECRET_KEY=your-secret-key-here
ACCESS_TOKEN_EXPIRE_MINUTES=15

# MFA
TOTP_ISSUER=ProjectAegis

# Secrets
UNSEAL_SHARE_TTL_SECONDS=300
SECRET_VIEW_LEASE_TTL_SECONDS=900
```

## Operating the Vault

### Initialize Vault
```
POST /api/vault/init
```
Returns 5 key fragments. Distribute securely to 5 different admins.

### Unseal Vault
```
POST /api/vault/unseal
Body: { "share": "fragment-1" }
```
Repeat with 3 unique fragments. UI shows progress (3/3).

### Seal Vault
```
POST /api/vault/seal
```
Locks the vault, requiring unseal ritual to access secrets.

### Panic Mode
```
POST /api/vault/panic
```
Emergency operation: revokes all active leases and immediately seals the vault.

## Security Features

- **Encryption at Rest**: Vault Transit encrypts all stored secrets
- **TLS/HTTPS**: Secure communication between services
- **Rate Limiting**: Redis-backed protection against brute-force attacks
- **Audit Logging**: Immutable hash-chained logs with tamper detection
- **MFA Enforcement**: TOTP tokens required for privileged operations
- **Session Management**: Secure JWT tokens with expiration
- **Device Tracking**: Fingerprint-based device identification

## Project Status

- ✅ Core secrets management
- ✅ Vault integration
- ✅ Audit logging
- ✅ MFA & RBAC
- ✅ Real-time alerts
- ✅ Docker deployment
- ✅ React dashboard

## Support & Contributing

For issues, feature requests, or contributions, please open an issue on the project repository.

## License

Project Aegis is provided as-is for educational and enterprise use.

## API Surface

| Method | Endpoint               | Description                      |
|--------|------------------------|----------------------------------|
| POST   | /api/auth/login        | Username/password stage          |
| POST   | /api/auth/mfa-verify   | TOTP verification, issues JWTs   |
| GET    | /api/users             | List users + roles               |
| POST   | /api/secrets/store     | Vault-transit static storage     |
| POST   | /api/secrets/issue     | Create dynamic lease             |
| GET    | /api/secrets/leases    | Inspect lease ledger             |
| GET    | /api/secrets/static    | List stored secrets (metadata)   |
| POST   | /api/secrets/static/{id}/view | Vault-transit decrypt + 15-min lease |
| GET    | /api/audit/entries     | Hash-chained audit log           |
| POST   | /api/vault/init        | Generate Shamir shares           |
| POST   | /api/vault/unseal      | Submit key fragment              |
| POST   | /api/vault/seal        | Seal vault + revoke access       |
| POST   | /api/vault/panic       | Revoke leases + seal             |
| GET    | /api/vault/status      | Vault seal state + progress      |
| GET    | /api/alerts            | Active security alerts           |

WebSocket channels:
- `/ws/audit` – live audit feed
- `/ws/vault` – seal/unseal status
- `/ws/leases` – lease issuance and panic revocations
- `/ws/alerts` – security alerts

## Testing & Next Steps

- Add pytest suites for services, Shamir validation, and audit chain verification.
- Integrate production-ready OIDC provider for SSO.
- Harden Vault integration by exchanging real unseal keys and enabling auto-unseal with HSM/cloud KMS.
- Extend Celery beat tasks for anomaly detection jobs.

## Troubleshooting

- **Rate Limit**: 429 responses include `retry_after`. Tune via `RATE_LIMIT_*` env vars.
- **WebSockets**: Ensure `VITE_WS_URL` matches backend host (`ws://localhost:8000`).
- **Vault Dev Token**: Provided via `.env`; rotate before production.
- **Migrations**: If using SQLite, update `alembic.ini` URL accordingly.

Project Aegis is ready for further hardening: plug into centralized identity, wire SIEM exports, and enforce infrastructure-as-code guardrails for enterprise deployment.
