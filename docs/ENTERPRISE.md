# Sliqtly Enterprise: full plan (after the pilot)

The pilot in [PLAN.md](PLAN.md) comes first: one binary, files on disk, no
sign-in. This document is where it goes after that.

Sliqtly that a company runs in its own account (cloud or on premises), with
sign-in through its own identity provider, its data in its own database and
bucket, and the MCP server open to its own assistants and services. The first
deliverable is a Docker Compose stack that runs the whole thing on a laptop,
including a stand-in enterprise identity provider, so every part can be tested
locally before it is packaged for a cloud.

Status: plan. Nothing here is built yet.

## 1. Goals

1. `docker compose up` gives a working Sliqtly at `https://sliqtly.localhost`:
   the editor, shared decks, pictures and data files, the MCP server.
2. Sign-in is OpenID Connect against the company's identity provider (Entra
   ID, Okta, Google Workspace, Keycloak, anything OIDC). Locally a Keycloak
   container plays that part, with demo users and groups.
3. The MCP endpoint (`/mcp`) is protected by OAuth 2.1 the way MCP clients
   expect (Claude, ChatGPT, Copilot, Cursor, a company's own agents), and the
   person behind each token is a user of the company's identity provider.
4. Services, not just people, can call Sliqtly: client-credentials tokens
   for MCP and a REST API, and an allowlist of internal data sources that
   charts may read from.
5. No call home. Nothing leaves the deployment unless an admin configures it.
6. The same image later runs on Google Cloud Run, AWS ECS and Kubernetes;
   only the storage drivers and the installer differ.

Not in the first version: SCIM provisioning, multi-tenant hosting (one
deployment = one company), SAML directly (Keycloak can broker SAML into
OIDC), Google Sheets/Drive live data (needs Google; stays optional).

## 2. What upstream Sliqtly is today

Source: [terotests/sliqtly](https://github.com/terotests/sliqtly).

| Part | Today | What ties it to Google |
| --- | --- | --- |
| Editor (`web/dist`) | static, built with Ranger, served by Firebase Hosting | `web/sliqtly.js` loads the Firebase SDK and `/__/firebase/init.js` (exists only on Firebase Hosting); `REDIRECT_HOSTS` hard-coded |
| Sign-in | Firebase Auth, Google accounts only | the browser talks to Firebase Auth directly |
| Data | Firestore `decks`, `shares`, `mcp_keys`, `mcp_oauth_*`; Storage `shares/{id}/…`, `users/{uid}/…` | the browser reads and writes Firestore and Storage directly, guarded by `firestore.rules` / `storage.rules` |
| MCP server | `mcp-go/` (Ranger compiled to Go, Cloud Run, in production since 2026-10-03; the Node server is gone) | Firestore, GCS and Firebase ID tokens, all behind interfaces in `mcp-go/host.go` |
| MCP sign-in | the server is its own OAuth 2.1 authorization server (RFC 8414, 9728, 7591, PKCE S256); `/oauth.html` signs the person in with Firebase and hands back an ID token | identity comes from Firebase |
| Chart data | `bind_chart_data` and pictures fetch public https URLs only (SSRF guard in `mcp-go/net.go`) | none, but internal sources are refused |

The Go server is the base for Enterprise: it is one static binary in a
distroless image (12 MB compressed, cold start well under a second), its
storage is already behind two small interfaces, and identity behind one
function:

```go
type DB interface {           // mcp-go/host.go
    Get, Set, Update, Delete, WhereEq, ServerTime
}
type Bucket interface { Name, Save, Read }
VerifyIDToken func(ctx, token string) (*IDToken, error)
```

Postgres, S3 and OIDC implementations slot in beside `firebase.go` without
touching the Ranger code.

## 3. Architecture

```
                 company IdP (Entra / Okta / Keycloak)
                        ▲  OIDC code flow + PKCE
                        │
 browser ──https──► Caddy ──► sliqtly (one Go binary)
 MCP clients ──────►  │        ├─ /              editor (web/dist)
 internal services ──►│        ├─ /config.js     runtime config for the editor
                      │        ├─ /auth/*        OIDC login, session cookie
                      │        ├─ /api/v1/*      decks, shares, files (REST, OpenAPI)
                      │        ├─ /mcp           MCP (streamable HTTP)
                      │        ├─ /oauth/*       OAuth 2.1 AS for MCP clients
                      │        ├─ /.well-known/* RFC 8414 / 9728 metadata
                      │        └─ /healthz /readyz /metrics
                      │              │            │
                      │          Postgres      S3 (MinIO locally)
```

One image, `sliqtly-enterprise:X.Y.Z`, built from this repository and a
pinned upstream Sliqtly ref.

### 3.1 Storage drivers

Postgres is the default everywhere, so local, AWS and Google run and are
tested the same way. Firestore stays as an option for a Google customer who
prefers it (the existing driver). Selected by environment:

| Setting | Values | Local | GCP | AWS |
| --- | --- | --- | --- | --- |
| `SLIQTLY_DB` | `postgres`, `firestore` | postgres | Cloud SQL postgres (default) or Firestore | RDS postgres |
| `SLIQTLY_FILES` | `s3`, `disk`, `gcs` | s3 (MinIO) | gcs | s3 |

The `DB` interface is document-shaped (collection, id, JSON). Postgres keeps
one table per collection with `id text primary key, doc jsonb`, plus
generated columns and indexes for the fields queried (`owner`, `expires`).
TTL that Firestore does by policy becomes a periodic delete job in the
server. Schema versions are tracked in a `schema_migrations` table; the
server runs pending migrations at start and refuses to start against a
schema newer than it knows.

### 3.2 The editor without Firebase

`web/sliqtly.js` becomes one of two backends behind the same functions it
exports today (`share`, `loadShare`, `listMine`, `saveShare`, `deleteShare`,
`putFile`, sign-in):

- `firebase` (sliqtly.com, unchanged)
- `api` (Enterprise): `fetch` to `/api/v1/*` with the session cookie;
  sign-in is a redirect to `/auth/login`

`/config.js` tells the page which backend, the base URL, the company name and
which features are on (Sheets/Drive off unless configured). This change goes
to upstream Sliqtly so both products build from one editor; Enterprise only
sets the config.

Access rules that live in `firestore.rules` / `storage.rules` today move into
the API handlers (owner checks, size limits) and get their own tests.

### 3.3 Sign-in for people

- OIDC authorization code flow with PKCE against `OIDC_ISSUER`, confidential
  client (`OIDC_CLIENT_ID` / `OIDC_CLIENT_SECRET`), discovery from
  `/.well-known/openid-configuration`.
- After login the server keeps a session (Postgres) and sets an `HttpOnly`,
  `Secure`, `SameSite=Lax` cookie. No tokens in the browser.
- Users are created on first login from `sub`, `email`, `name`. The user id
  is `issuer + sub`, never the email.
- Roles from a configurable claim (`OIDC_ROLES_CLAIM`, e.g. `groups` or
  `roles`) mapped by `ROLE_MAP`:
  - `admin`: settings, data sources, audit log, all decks
  - `editor`: create and share decks
  - `viewer`: open decks shared inside the company
- Optional `OIDC_ALLOWED_GROUPS`: anyone outside them is refused.
- Logout ends the session and calls the IdP's `end_session_endpoint`.

### 3.4 Sign-in for MCP clients

The server stays the OAuth 2.1 authorization server MCP clients discover
(`/.well-known/oauth-protected-resource`, `/.well-known/oauth-authorization-server`,
dynamic client registration, client ID metadata documents, PKCE S256, rotating
refresh tokens), as upstream does now. What changes is who vouches for the
person: `/oauth/authorize` sends the browser through the same OIDC login as
the editor (or reuses the session), then shows a consent page naming the
client and the scopes.

Two modes:

| Mode | When | How |
| --- | --- | --- |
| `broker` (default) | most companies; works with every MCP client today | Sliqtly issues its own tokens after IdP login |
| `external` | the company wants its IdP to issue all tokens (e.g. Entra app registration) | Sliqtly is only a resource server: validates the IdP's JWT access tokens (issuer, audience, signature from JWKS) and advertises the IdP in RFC 9728 metadata |

Scopes: `decks:read`, `decks:write`, `files:write`, `admin`. A token never
gets more than the user's role allows.

Admin controls: allowed redirect URI patterns, allowed client ids (or open
dynamic registration), token lifetimes, revoke all tokens of a user or client.

### 3.5 Services that call Sliqtly

- Service clients created by an admin: `client_id` + secret, OAuth
  client-credentials grant, scopes as above, acting as a named service user.
- The same tokens work on `/mcp` (an internal agent) and on `/api/v1`
  (a report generator, a CI job).
- `/api/v1` is described by an OpenAPI document served at `/api/v1/openapi.json`.

### 3.6 Services Sliqtly reads from

Charts (`bind_chart_data`, `vega-lite` data URLs) and pictures fetch by URL.
Upstream refuses private addresses. Enterprise adds admin-defined data
sources:

```yaml
datasources:
  - name: finance-api
    match: https://finance.internal.example.com/reports/
    headers:
      Authorization: "Bearer ${secret:finance_token}"
```

A URL that matches a source may reach that private host, with the
configured headers added on the server; the secret never reaches the
browser or the deck. Everything else keeps the public-only rule (and the
metadata-server block always applies). Outbound access can also be switched
off entirely (`SLIQTLY_OUTBOUND=none`).

### 3.7 Sharing

Both inside and outside the company. The owner picks per share:

- **Company**: opens only for signed-in users of this deployment
- **Link**: anyone with the link, as upstream, for customers, partners and
  the public; optional expiry date and revoke

Admin settings decide what owners may choose:

| Setting | Values | Default |
| --- | --- | --- |
| `SHARE_DEFAULT` | `org`, `link` | `org` |
| `SHARE_EXTERNAL` | `on`, `off`, `role:<role>` (only that role and above) | `on` |
| `SHARE_LINK_MAX_DAYS` | days, empty for no limit | empty |

Every external share is in the audit log, and an admin can list and revoke
them.

### 3.8 Operations

- Audit log (Postgres): sign-ins, token grants and revocations, deck
  create/share/delete, admin changes, data-source use. Export as JSON lines.
- Structured JSON logs on stdout, Prometheus `/metrics`, OpenTelemetry
  traces when `OTEL_EXPORTER_OTLP_ENDPOINT` is set.
- `/healthz` (process up) and `/readyz` (database and bucket reachable).
- Backups are the database's and bucket's own (pg_dump / snapshots, bucket
  versioning); `sliqtly-enterprise backup` / `restore` for the Compose setup.

## 4. The local Docker stack

```
compose.yaml
├─ caddy       TLS for *.localhost (its own local CA), reverse proxy
├─ sliqtly     the server (built here, or the released image)
├─ postgres    16
├─ minio       S3 API + console; a bucket created on start
└─ keycloak    stand-in IdP: realm "acme" imported from dev/keycloak/
```

Demo realm `acme` (dev only):

| User | Password | Groups | Sliqtly role |
| --- | --- | --- | --- |
| alice@acme.test | alice | sliqtly-admins | admin |
| bob@acme.test | bob | sliqtly-editors | editor |
| carol@acme.test | carol | sliqtly-viewers | viewer |
| dave@acme.test | dave | (none) | refused |

Plus a service client `acme-reporting` with client-credentials enabled.

Addresses:

| | |
| --- | --- |
| `https://sliqtly.localhost` | the editor |
| `https://sliqtly.localhost/mcp` | MCP |
| `https://auth.localhost` | Keycloak (admin console: admin / admin) |
| `https://minio.localhost` | MinIO console |

Try it:

```sh
cp .env.example .env
docker compose up -d
open https://sliqtly.localhost              # sign in as bob@acme.test

# an MCP client
claude mcp add --transport http sliqtly https://sliqtly.localhost/mcp
npx @modelcontextprotocol/inspector         # or the MCP Inspector

# a service
TOKEN=$(curl -s https://sliqtly.localhost/oauth/token \
  -d grant_type=client_credentials -d client_id=acme-reporting \
  -d client_secret=... -d scope=decks:write | jq -r .access_token)
curl -H "Authorization: Bearer $TOKEN" https://sliqtly.localhost/api/v1/decks
```

To try a real IdP, set `OIDC_ISSUER`, `OIDC_CLIENT_ID`,
`OIDC_CLIENT_SECRET` in `.env` and start without the Keycloak profile
(`docker compose --profile no-idp up`).

Configuration (all environment, documented in `.env.example`):

```
SLIQTLY_URL=https://sliqtly.localhost
SLIQTLY_DB=postgres            DATABASE_URL=postgres://...
SLIQTLY_FILES=s3               S3_ENDPOINT= S3_BUCKET= S3_ACCESS_KEY= S3_SECRET_KEY=
OIDC_ISSUER=  OIDC_CLIENT_ID=  OIDC_CLIENT_SECRET=
OIDC_ROLES_CLAIM=groups        ROLE_MAP=sliqtly-admins:admin,sliqtly-editors:editor,sliqtly-viewers:viewer
OIDC_ALLOWED_GROUPS=
MCP_AUTH_MODE=broker           # or external
SHARE_DEFAULT=org              SHARE_EXTERNAL=on
SLIQTLY_OUTBOUND=public        # public | none
SESSION_SECRET=                # 32 random bytes, base64
```

## 5. Repository layout

```
SliqtlyEnterprise/
├─ docs/PLAN.md            this file
├─ upstream.json           Sliqtly repo + ref the image is built from
├─ server/                 Go: main, config, drivers, auth, api, audit
│  ├─ store/postgres/      DB interface on Postgres (+ migrations/)
│  ├─ store/firestore/     (wraps upstream firebase.go)
│  ├─ files/s3/  files/disk/
│  ├─ auth/oidc/           login, sessions, roles
│  ├─ auth/mcp/            broker and external modes
│  ├─ api/                 /api/v1 + openapi.json
│  └─ datasources/
├─ Dockerfile              stage 1 fetches upstream at upstream.json's ref,
│                          builds web/dist and the Ranger → Go code;
│                          stage 2 builds server/; stage 3 distroless
├─ compose.yaml  .env.example  Caddyfile
├─ dev/keycloak/acme-realm.json
├─ test/e2e/               Playwright + MCP client tests against the stack
└─ .github/workflows/      build, test, e2e on Compose, signed image
```

How Enterprise uses upstream: the Go host in `mcp-go/` and the editor are
upstream code. Changes that every deployment needs (the editor backend
switch, `/config.js`, serving `web/dist` from the Go binary, injectable
`Client` allowlist) go to upstream as PRs. Enterprise-only code (OIDC,
Postgres, S3, admin, audit, data sources) lives here and wraps upstream's
`Env`. If wrapping needs upstream's Go to be importable as a package rather
than `package main`, that refactor is upstream PR #1.

## 6. Milestones

Each one ends with something that runs in Compose and an automated test.

**M0: stack skeleton**
- `upstream.json`, Dockerfile building upstream `mcp-go` and `web/dist`.
- Compose with Caddy, Postgres, MinIO, Keycloak (realm imported), sliqtly.
- Upstream PR: the Go server serves `web/dist` and `/config.js`.
- Done when: the editor opens at `https://sliqtly.localhost` and `/mcp`
  answers `initialize` (link mode, no sign-in).

**M1: storage**
- Postgres `DB` and S3/disk `Bucket` drivers, migrations, TTL job.
- Upstream PR: `web/sliqtly.js` backend switch (`firebase` | `api`).
- `/api/v1` decks, shares, files with owner checks ported from the rules.
- Done when: create, share, reopen and delete a deck with pictures; MCP
  `create_presentation` keeps it in Postgres; driver tests run against
  real Postgres and MinIO in CI.

**M2: people sign in**
- OIDC login, sessions, users, role mapping, allowed groups, logout.
- Company and link shares with the admin settings of 3.7.
- Done when: e2e test signs in as alice, bob, carol, dave and sees the four
  expected outcomes.

**M3: MCP clients sign in**
- `/oauth/authorize` through the OIDC session; consent page; scopes.
- `external` mode with JWKS validation.
- Done when: an MCP client test (the Go MCP client upstream already uses)
  completes registration, authorize, token, refresh and a tool call as bob;
  Claude Code connects by hand.

**M4: services**
- Client-credentials grant, admin-created service clients.
- Data-source allowlist with server-side secrets.
- OpenAPI document.
- Done when: `acme-reporting` creates a deck through `/api/v1` and through
  `/mcp`, and a chart reads from a mock internal service in the stack.

**M5: admin and audit**
- Admin page: users, roles seen, tokens and clients (revoke), data sources,
  settings; audit log view and export.
- Metrics, readiness, OTel.

**M6: release and updates**
- Versioned, signed image (cosign) with SBOM; release manifest (version,
  digest, oldest version it upgrades from, migrations).
- Update = back up, pull, `docker compose up -d`; migrations are
  expand/contract so the previous version still runs on the new schema
  and a rollback is a version change.
- Editor tabs left open get a "new version, reload" notice from
  `/api/v1/version` (the build already cache-busts every file with `?v=`).

**M7: cloud packages** (after the Compose version is solid)
- Helm chart.
- AWS: CloudFormation/CDK (ECS Fargate, RDS, S3, Secrets Manager, ALB).
- Google: Terraform (Cloud Run, Cloud SQL postgres or Firestore as the
  customer chooses, GCS, Secret Manager).
- Marketplace listings.

## 7. Security checklist

- TLS everywhere, HSTS; cookies `HttpOnly; Secure; SameSite=Lax`; CSRF token
  on state-changing cookie requests.
- OIDC: PKCE, `state`, `nonce`, issuer and audience checks, JWKS rotation.
- OAuth AS: exact redirect URI match (loopback any port, as upstream),
  short-lived codes, hashed tokens at rest (upstream already hashes),
  refresh rotation with reuse detection.
- Server-side authorization on every API route; no rule lives only in the
  browser.
- SSRF: public-only unless a data source matches; metadata addresses always
  blocked; redirects re-checked.
- Secrets from environment or files (`*_FILE`), never logged.
- Rate limits per user and per client (upstream `Limiter`, `Quota`).
- Distroless, non-root image; dependency and image scanning in CI.

## 8. Decisions

| Question | Decision |
| --- | --- |
| Licence for customer builds | not an issue (copyright holder's call) |
| Database | Postgres by default everywhere; Firestore optional on Google |
| Sharing | company and external link shares, per share, limited by admin settings (3.7) |
| MCP server | the Go server (`mcp-go`) only; the Node server is not part of Enterprise |

## 9. Open questions

1. Which MCP clients must be verified for the first release (Claude,
   ChatGPT, Copilot Studio, Cursor)?
