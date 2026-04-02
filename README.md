# P3DX Platform

## Repositories

| Repository | DX Pipeline | Stack | Role |
|---|---|---|---|
| `p3dx-aaa` | [p3dx-aaa-conman](https://github.com/datakaveri/p3dx-aaa/tree/p3dx-aaa-conman) | Node.js / Express | Authentication backend, contract generation, immuDB audit logging |
| `p3dx-auth-ui` | [p3dx-auth-dx](https://github.com/datakaveri/p3dx-auth-ui/tree/p3dx-auth-dx) | React / Vite | Frontend SPA — login, workload submission, admin panel |
| `p3dx-apd` | [main](https://github.com/datakaveri/p3dx-apd/tree/main) | Go / chi | Access Policy Database — policy storage, access request lifecycle |
| `p3dx-top` | [main](https://github.com/datakaveri/p3dx-top/tree/main) | Go | Trusted Orchestrator Protocol — contract validation, signing, TEE deployment |

Each repository has its own README with full setup and API reference.

---

## System Architecture

```
Browser
  └── nginx  (auth.p3dx.iudx.org.in  :443)
        ├── /assets/*, /app/*, /login, /register
        │     └── p3dx-auth-ui  (React SPA, static files from dist/)
        │
        └── /p3dx/*, /anon/*
              └── p3dx-aaa  (:3001)
                    ├── Keycloak       (:8080)   identity + JWT issuance
                    ├── immuDB         (:3322)   tamper-evident audit log
                    ├── p3dx-apd       (:8082)   policy proxy (data-provider flow)
                    └── p3dx-top       (:8085)   contract submission (run workload flow)
                              ├── p3dx-aaa  (:3001)  token fingerprint verification callback
                              ├── p3dx-apd  (:8082)  policy check before signing
                              └── TEE               enclave deployment
```

**Request path summary:**
- All traffic enters through nginx on port 443.
- Static UI assets and SPA routes are served from the built `dist/` directory.
- All API calls (`/p3dx/*`, `/anon/*`) are proxied to `p3dx-aaa` on port 3001.
- `p3dx-aaa` orchestrates Keycloak, immuDB, APD, and TOP.
- TOP calls back into `p3dx-aaa` and APD during contract ingestion.

---

## User Roles

| Role | Assigned by | Capabilities |
|---|---|---|
| `user` | Auto on registration | Login, run workloads, view own contracts |
| `data-provider` | Admin approval | Everything `user` can do + submit dataset access policies |
| `application-provider` | Admin approval | Everything `user` can do + submit application access policies |
| `admin` | Manually in Keycloak | Approve / deny role requests |

Role requests go through an in-platform workflow (user requests → admin approves → Keycloak role assigned).

---

## Key Flows

### 1. Login and Registration

```
User fills in login form (auth-ui)
  → POST /p3dx/login  (p3dx-aaa)
      → Keycloak ROPC token endpoint
      → Returns access_token, refresh_token, expires_in
  → Tokens stored in localStorage
  → UI reads roles from JWT, routes to appropriate dashboard
```

Registration follows the same path via `POST /p3dx/register`.

---

### 2. Set Policy (data-provider flow)

A data provider must publish a policy for their dataset before any workload can run against it. TOP rejects contracts for datasets without a policy.

```
Data provider opens Set Policy form (auth-ui)
  → POST /p3dx/policy  (p3dx-aaa)  [requires data-provider role]
      → Proxied to APD: POST /api/v1/policy
          → APD stores policy in PostgreSQL
          → Policy dump written to disk (if APD_POLICY_DUMP_DIR is set)
```

---

### 3. Run Workload (main flow)

This is the centrepiece of the platform. The user selects a dataset and application; the system generates a multi-party signed contract and provisions a TEE to run the computation.

```
User selects dataset + application in Run Workload form (auth-ui)
  → POST /p3dx/workloads/run  (p3dx-aaa)  [requires user role]
      1. Go contract generator creates a contract and produces the consumer signature
      2. Contract + consent metadata stored in immuDB
      3. Contract submitted to TOP
           POST /p3dx-top/contract  (payload: access_token + consumer signature + contract)

         Inside TOP — 12-step ingestion pipeline:
           Step 1–2   Contract received; contract_id extracted
           Step 3     SHA-256 fingerprint computed from the user token
           Step 4     Fingerprint verified with p3dx-aaa
                        POST /p3dx/workloads/contracts/:id/token-verify  (X-API-Key)
                        → mismatch → 403 rejected
           Step 5     Contract marshaled for signing
           Step 6     Dataset policy fetched from APD
                        GET /api/v1/policy/by-item/:datasetId
                        → no policy found → 403 rejected
           Step 7     Unsigned contract stored to disk
           Step 8     Orchestrator signs contract  (RSA-PKCS1v15-SHA256)
           Step 9     Signed contract stored to disk (overwrites step 7)
           Step 10    Docker Compose URL resolved from p3dx-aaa
                        GET /p3dx/apps/:appId/compose-url  (X-API-Key)
           Step 11    DeployEnclave called with contract + compose URL
           Step 12    TEE started

  → contract_id returned to UI
  → UI navigates to Workload Result page

User polls for result (auth-ui)
  → GET /p3dx/workloads/contracts/:id/result  (p3dx-aaa)
      → Fetches fully-signed contract from TOP
      → Returns TEE status + signed contract JSON
```

**The signed contract** encodes:
- Dataset terms (data resource ID, dataset name, data URL, format)
- Application terms (app ID, name, container image hash)
- Consumer terms (data blob URL, retention policy)
- Parties (consumer, application provider, data provider — each with public key)
- Two signatures: consumer signature (generated by `p3dx-aaa`) and orchestrator signature (added by TOP)
- Execution platform: `AZURE_AMD_SEV`

---

### 4. Role Request Workflow

```
User opens profile / role request page (auth-ui)
  → POST /p3dx/role-requests  (p3dx-aaa)
      → Role request stored in immuDB

Admin opens Admin Dashboard (auth-ui)
  → GET /p3dx/admin/role-requests  (p3dx-aaa)
  → Sees pending request

Admin approves
  → POST /p3dx/admin/role-requests/:id/decision  { "decision": "approve" }
      → p3dx-aaa calls Keycloak Admin API to assign realm role
      → Decision logged in immuDB

User re-logs in → new token includes the granted role
```

---

## Prerequisites

Before starting the platform, the following infrastructure must be running and configured:

| Service | Default Port | Notes |
|---|---|---|
| Keycloak | 8080 | Realm, client, and roles (`user`, `data-provider`, `application-provider`, `admin`) must be pre-configured |
| immuDB | 3322 | Database `anon_audit` and user `anon_backend` must be provisioned (run `npm run setup:immudb` in p3dx-aaa) |
| PostgreSQL | 5432 | APD database and schema must exist (`psql ... -f schema.sql` in p3dx-apd) |
| nginx | 443 | Configured for `auth.p3dx.iudx.org.in`; serves auth-ui dist and proxies `/p3dx/*` to port 3001 |

---

## Startup Order

Services must start in dependency order:

```
1. Keycloak          # identity provider — everything depends on it
2. immuDB            # audit log — p3dx-aaa connects on startup
3. PostgreSQL        # APD database
4. p3dx-apd  (:8082) # policy store — TOP queries it during contract ingestion
5. p3dx-top  (:8085) # orchestrator — p3dx-aaa calls it on workload run
6. p3dx-aaa  (:3001) # auth backend — p3dx-auth-ui calls it for all API requests
7. nginx             # reverse proxy — serves the UI and routes API calls
```

In production, `p3dx-aaa` is managed by systemd (`p3dx-aaa-auth-backend.service`). All others are started manually or via their own service managers.

---

## Local Development Quick Start

### p3dx-apd

```bash
cd p3dx-apd
cp .env.example .env   # fill in DB_*, JWT_*, APD_SIGNING_KEY_PATH, TEE_ORCHESTRATOR_URL
go run ./cmd/server/main.go
# APD server listening on :8082
```

### p3dx-top

```bash
cd p3dx-top
cp .env.example .env   # fill in BACKEND_URL, BACKEND_API_KEY, APD_BASE_URL, STORE_KEY
go run .
# TOP running on port 8085
```

### p3dx-aaa

```bash
cd p3dx-aaa
npm install
cp .env.example .env   # fill in Keycloak, immuDB, TOP_BASE_URL, CONTRACT_SERVER_SECRET
npm run setup:immudb   # first time only
npm start
# p3dx-aaa auth backend running on port 3001
```

### p3dx-auth-ui

```bash
cd p3dx-auth-ui
npm install
# .env for local dev (optional — defaults to http://localhost:3001):
echo "VITE_BACKEND_URL=http://localhost:3001" > .env
npm run dev
# Dev server at http://localhost:5174
```

See each repository's README for the full environment variable reference.

---

## Production Deployment

### Auth UI

```bash
cd p3dx-auth-ui
npm run build   # outputs to dist/
```

`dist/` is served by nginx as a static SPA. All non-API paths fall through to `dist/index.html`.

### p3dx-aaa (systemd)

```ini
# /etc/systemd/system/p3dx-aaa-auth-backend.service
[Unit]
Description=p3dx-aaa auth backend
After=network.target

[Service]
WorkingDirectory=/home/azureuser/p3dx-aaa
ExecStart=/usr/bin/node src/server.js
EnvironmentFile=/home/azureuser/p3dx-aaa/.env
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable p3dx-aaa-auth-backend
sudo systemctl start  p3dx-aaa-auth-backend
```

### nginx (auth.p3dx.iudx.org.in)

```nginx
server {
    listen 443 ssl;
    server_name auth.p3dx.iudx.org.in;

    # Static assets (long-lived cache)
    location /assets/ {
        root /home/azureuser/p3dx-auth-ui/dist;
        expires 1y;
    }

    # API — proxied to p3dx-aaa
    location /anon/ { proxy_pass http://127.0.0.1:3001; }
    location /p3dx/ { proxy_pass http://127.0.0.1:3001; }

    # SPA fallback
    location / {
        root /home/azureuser/p3dx-auth-ui/dist;
        try_files $uri $uri/ /index.html;
    }
}
```

> **Note:** nginx runs as `www-data`. The `dist/` directory and all parent directories must be traversable by `www-data`:
> ```bash
> chmod o+x /home/azureuser /home/azureuser/p3dx-auth-ui /home/azureuser/p3dx-auth-ui/dist
> ```

For the full nginx config including SSL and the Spider SSO redirect, see `p3dx-aaa/Setup.md §9.6`.

---

## Technology Stack

| Component | Language | Key Dependencies |
|---|---|---|
| p3dx-aaa | Node.js 18+ | Express 5, jose (JWT), immudb-node, axios, uuid |
| p3dx-auth-ui | React 19 + Vite | React Router 7, lucide-react |
| p3dx-apd | Go 1.22+ | chi, pgx (PostgreSQL), ECDSA (P-256) |
| p3dx-top | Go 1.22+ | godotenv, RSA (PKCS1v15 SHA-256), AES-GCM storage |
| Identity | — | Keycloak (OIDC / ROPC) |
| Audit | — | immuDB (tamper-evident key-value ledger) |
| TEE | — | Azure AMD SEV-SNP |

---

## More Information

- **Full setup guide** (Keycloak realm config, immuDB provisioning, env var reference, troubleshooting): `p3dx-aaa/Setup.md`
- **API reference**: each repository's own `README.md`
- **Contract ingestion detail**: `p3dx-top/README.md` — Environment Variables and Pipeline sections
- **Access request lifecycle**: `p3dx-apd/README.md` — Access Request Lifecycle section
