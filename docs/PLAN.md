# Sliqtly Server: pilot plan

Goal: run Sliqtly inside a company network as fast as possible. One program
that an assistant connects to over MCP, that keeps presentations on its own
disk, and that gives every presentation a URL where people view it, and
where the assistant (and later people) edit and extend it.

No Google, no cloud account, no database server, no sign-in for the first
pilot. Those come after, in [ENTERPRISE.md](ENTERPRISE.md).

## 1. What it is

`sliqtly-server`: the MCP server that runs `sliqtly.com/mcp` today
(`mcp-go/` in the Sliqtly repo, Ranger compiled to Go), with three changes:

1. **A file-system store** instead of Firestore and Cloud Storage.
2. **The viewer and editor built in** (`web/dist` embedded in the binary),
   so `/s/{id}` opens a presentation from the same server.
3. **Its own address** in every link it hands out (`SLIQTLY_URL`), instead
   of `https://sliqtly.com`.

Delivered two ways from one build:

| | |
| --- | --- |
| Docker image | `docker run -p 8080:8080 -v sliqtly-data:/data sliqtly-server` |
| Static binary | Linux amd64/arm64, Windows, macOS; `./sliqtly-server --data ./data` |

The binary is already static (`CGO_ENABLED=0`, distroless image of a few tens
of MB), so both are cheap.

```
 Claude Code / VS Code / Cursor / own agent
        │  MCP (streamable HTTP)
        ▼
 sliqtly-server :8080 ───────────► ./data/
   /mcp               MCP tools      shares/{id}.json        a presentation
   /s/{id}            viewer         mcp_keys/{id}.json      its edit key (hash)
   /s/{id}?edit       editor         files/shares/{id}/...   pictures, data files
   /api/shares/{id}   deck JSON
   /files/...         pictures, data files
   /s/{id}/{n}.jpg    a slide as a picture (server-side render)
        ▲
        │  browser
 people in the network
```

## 2. Why this is a small change

What the Go server already has, in `mcp-go/`:

- All eleven MCP tools: create, update, get, list, render a slide, render an
  overview, chart data binding, data files, workbooks, the guide.
- Storage behind two interfaces in `host.go` (`DB`: get, set, update,
  delete, query by one field; `Bucket`: save, read). Firestore and GCS are
  just one implementation, and the tests already use an in-memory one.
- Edit keys: `create_presentation` returns a key, `update_presentation`
  needs it. So the assistant can create and keep editing a deck **without
  anyone signing in**.
- Server-side rendering of slides to JPEG (`render.go`), which is what
  "render the documents" needs even without a browser.
- A link-only mode (`SLIQTLY_STORE=link`) where the whole deck travels in the
  URL. It works today, but the link points at `sliqtly.com` and decks are
  not kept, so it is not enough on its own.

What is missing:

| Gap | Where | Size |
| --- | --- | --- |
| File-system `DB` and `Bucket` | new `mcp-go/fsstore.go` | ~200 lines + tests |
| File URLs: `Store.rgr` writes `https://firebasestorage.googleapis.com/...` | a host operator `host_file_url` so the fs store answers `/files/...` | small |
| Serve `web/dist`, `/s/{id}`, `/api/shares/{id}`, `/files/*` | `mcp-go/main.go` + `go:embed` | small |
| The page reads a share through Firebase in the browser (`web/sliqtly.js` `loadShare`) | read `/api/shares/{id}` when the page is served by `sliqtly-server` (a `/config.js` flag) | small |
| PRO, Google sign-in, Sheets/Drive shown on the page | hidden by the same flag | small |
| `main.go` chooses Firestore or link only | `SLIQTLY_STORE=fs` + `SLIQTLY_DATA` | small |

### Where the code lives

The Sliqtly repository is private and SliqtlyEnterprise is **public**. The
server code and its builds stay in the Sliqtly repository (`mcp-go/`, a
`SLIQTLY_STORE=fs` mode beside the Firestore one, so sliqtly.com and the
company server are one program). Images go to a **private** registry
(`ghcr.io/terotests/sliqtly-server`, pulled with a token) and binaries to
releases of the private repo, or are handed over as files
(`docker save` / the binary).

This repository holds only what is safe to be public: the Compose files,
the Caddy configurations, install and client set-up documentation.

## 3. Storage

**First adapter: the file system.** One JSON file per document, written to a
temporary file and renamed, so a crash never leaves half a file. A query by
owner reads the directory, fine for thousands of decks. Backup is copying
the folder; moving to another server is moving the folder.

```
/data
  shares/k3Fx9a2Q.json
  mcp_keys/k3Fx9a2Q.json
  files/shares/k3Fx9a2Q/media/chart.png
  files/shares/k3Fx9a2Q/data/sales.xlsx
```

Expired anonymous decks (`expires`, what Firestore's TTL deleted) are removed
by a sweep once an hour. Pilot default: no expiry.

**Later, behind the same interface,** in this order:

1. SQLite (pure Go driver, binary stays static): one file, many decks,
   proper indexes.
2. Postgres (`jsonb`): several server instances, the company's managed
   database.
3. MongoDB fits the document-shaped interface just as well; only if a
   customer asks.

## 4. HTTPS in a company network

The server itself speaks plain HTTP on one port. TLS is in front of it.
Options, easiest first for a pilot:

| Option | Certificate trusted by browsers? | What it needs | Fits |
| --- | --- | --- | --- |
| **The company's existing reverse proxy / ingress** | yes (company CA) | IT adds one route to `http://host:8080` | most companies; ask first |
| **Caddy + Let's Encrypt, DNS-01** | yes (public CA) | a name under a public domain the company owns (`sliqtly.intra.example.com`) and an API token for its DNS (Cloudflare, Route 53, Azure DNS, ...). The server does **not** need to be reachable from the internet. Caddy renews by itself. | the best self-contained option |
| **Tailscale** (`tailscale serve`) | yes (`*.ts.net`) | Tailscale allowed in the company | quickest if they use it |
| **Caddy `tls internal`** (own CA) | only after the root certificate is installed on each machine | distributing one file | a small test group |
| **Plain HTTP** `http://host:8080` | no TLS | nothing | first smoke test; Claude Code accepts `http://` MCP URLs |

Certbot does the same as Caddy's ACME, but needs a web server and a renewal
job beside it; Caddy is one container with both. The Compose files in this
repository will have a profile per option:

```sh
docker compose up -d                                  # plain HTTP :8080
docker compose --profile acme-dns up -d               # Caddy + Let's Encrypt DNS-01
docker compose --profile internal-ca up -d            # Caddy with its own CA
```

## 5. Which MCP clients reach a server inside the network

A client that connects **from the user's machine** reaches an internal
address:

- Claude Code: `claude mcp add --transport http sliqtly https://sliqtly.intra.example.com/mcp`
- VS Code (Copilot agent mode), Cursor, Windsurf: the URL in their MCP settings
- a company's own agents (any MCP client library)
- Claude Desktop and other stdio-only set-ups: through a local bridge
  (`npx mcp-remote https://sliqtly.intra.example.com/mcp`)

Clients whose MCP connections are made **from the vendor's cloud** (custom
connectors on claude.ai, ChatGPT connectors) cannot reach a server that is
only inside the network. For those the server needs a public address
(a tunnel or the company's DMZ) and sign-in, which is part of ENTERPRISE.md.
Check each client's current documentation before the pilot; this changes.

## 6. Access in the pilot

- The network is the boundary: whoever reaches the server can view a
  presentation whose id they have.
- Editing through MCP needs the deck's edit key (exists today).
- Optional `SLIQTLY_TOKEN`: when set, `/mcp` requires
  `Authorization: Bearer <token>` (Claude Code: `--header`), so only
  configured assistants create decks.
- Saving edits made in the browser editor back to the server needs knowing
  who is editing. Pilot step 1: the editor shows the deck and edits are kept
  in the browser (as today without PRO); the assistant saves to the server.
  Pilot step 2: browser save with the deck's edit key (`/s/{id}?edit&key=…`,
  the link the assistant already gets). OIDC sign-in replaces this later.

## 7. Configuration

| Variable | Default | |
| --- | --- | --- |
| `SLIQTLY_URL` | `http://localhost:8080` | the address people and links use |
| `SLIQTLY_STORE` | `fs` | `fs`, `link` (keep nothing), `firestore` (sliqtly.com) |
| `SLIQTLY_DATA` | `/data` | the folder for `fs` |
| `SLIQTLY_TOKEN` | empty | bearer token required on `/mcp` |
| `SLIQTLY_OUTBOUND` | `public` | `public` (pictures and chart data from public https URLs) or `none` |
| `PORT` | `8080` | |

## 8. Steps

Each step is merged when it has tests and runs in Docker.

**P1, in the Sliqtly repo (`mcp-go/`): file-system store**  
_Done on the Sliqtly branch `claude/confident-curie-i6n4po`: `-data` / `SLIQTLY_DATA`, plus a first viewer (`/`, `/s/{id}` as server-drawn slides, `/s/{id}/{n}.jpg`, `/s/{id}/overview.jpg`) and `SLIQTLY_TOKEN`. Usage in `mcp-go/README.md` there._
- `fsstore.go`: `DB` and `Bucket` on a folder; `SLIQTLY_STORE=fs`,
  `SLIQTLY_DATA`.
- `host_file_url` so file URLs point at `SLIQTLY_URL/files/...`.
- `/files/*` served from the folder.
- The existing end-to-end tests (`server_test.go`) run against the fs store
  as well as the in-memory one.
- Done when: `go run .` with `SLIQTLY_STORE=fs`, a deck created and updated
  over MCP survives a restart, `get_presentation` and `render_slide` work.

**P2, Sliqtly repo: viewer from the same server**
- `web/dist` embedded (`go:embed`), `/s/{id}` → the page, `/config.js`.
- `web/sliqtly.js`: when `/config.js` says self-hosted, `loadShare` is a
  `fetch("/api/shares/{id}")`; PRO and Google features hidden.
- Optional: `/s/{id}/{n}.jpg` from `render.go` for previews in chat tools,
  wikis and e-mail.
- Done when: the link `create_presentation` returns opens the deck from the
  server with no request leaving the network (checked with outbound
  blocked in the test).

**P3, Sliqtly repo: build and release**
- `Dockerfile` target producing the one image (web + server).
- Workflow: on a tag, image (amd64 + arm64) to the private GHCR and
  binaries for Linux, Windows, macOS as release assets.
- Note: generating the Go code needs ~4 GB of memory; done in CI, never on
  the company's server.

**P4, this repo: installation**
- `compose.yaml` with the three profiles of section 4, `.env.example`,
  Caddyfiles (DNS-01 for the common DNS providers).
- `docs/install.md`: Docker, the binary as a systemd service, Windows
  service; backups; update (`docker compose pull && docker compose up -d`,
  the data folder is untouched).
- `docs/clients.md`: Claude Code, VS Code, Cursor, mcp-remote, token header.

**→ Pilot in a company network.**

**P5, after the pilot, from what it teaches**
- Browser save with the edit key (section 6, step 2).
- SQLite store.
- OIDC sign-in and OAuth for MCP clients (ENTERPRISE.md M2–M3).

## 9. Open questions

1. Which MCP clients will the pilot company use? (Decides whether a public
   address is needed at all, section 5.)
2. Does the pilot company have a reverse proxy / internal CA we can use, or
   a public domain with a DNS API for Let's Encrypt?
3. Is the network allowed to reach the internet? (Pictures and chart data
   from public URLs; the page itself will not need it after P2.)
