# Sliqtly Enterprise

Running [Sliqtly](https://sliqtly.com) inside a company: the MCP server and the
presentation viewer as one program in Docker or as a single binary, keeping
presentations on its own disk, reachable from the company's assistants and
browsers.

This repository is public and holds deployment only (Compose, Caddy,
documentation). The server is built from the private Sliqtly repository.

- [docs/PLAN.md](docs/PLAN.md): the pilot, one binary, files on disk, no sign-in
- [docs/ENTERPRISE.md](docs/ENTERPRISE.md): after the pilot, OIDC sign-in,
  OAuth for MCP clients, Postgres, S3, cloud packages

Status: planning.
