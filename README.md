# Sliqtly Enterprise

[Sliqtly](https://github.com/terotests/sliqtly) packaged for companies to run
in their own account: sign-in through their own identity provider (OIDC),
data in their own Postgres and S3 bucket, and the MCP server open to their
own assistants and services.

First target: a Docker Compose stack that runs everything locally, with
Keycloak standing in for the company's identity provider. Cloud packages
(AWS, Google Cloud, Helm) follow from the same image.

Status: planning. See [docs/PLAN.md](docs/PLAN.md).
