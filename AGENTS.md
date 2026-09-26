# AGENTS.md

Instructions for coding agents in this repo.

## Repo purpose

This repo holds Docker Compose stacks. Coolify runs them on the server. Do not run `docker compose` commands against this repo on the workstation. Coolify magic variables like `SERVICE_FQDN_*` only resolve at deploy time.

## Layout

- `apps/<name>/docker-compose.yml` is the canonical stack file.
- `apps/<name>/env.example` holds placeholder values. Commit this file.
- `apps/<name>/env.local` holds real secrets. Do not commit this file. Do not print secrets in chat.

Match this layout for a new app.

## Ingress

Cloudflare Tunnel routes `*.delulo.uk` to Traefik on the Coolify host. Traefik routes to the app by Host header. Reference: <https://coolify.io/docs/integrations/networking/cloudflare/tunnels>.

Rules for a new app:

1. Do not publish container ports to the host. Use `expose:` in the compose file.
2. Set the Coolify **Domains** field to `http://<app>.delulo.uk`. Do not add a port. Do not add a path. Do not use `https://`.
3. Start from the upstream compose file for the project. Adapt for Coolify.

## Secrets

Coolify stores env vars in its own UI. Coolify does not read `env.local`. Use `env.local` as a local reference. Copy values into the Coolify Environment Variables UI.

## In-compose config files

Prefer a top-level `configs:` block with inline `content:`. Do not use bind mounts. Bind mounts couple the stack to a host path that Coolify does not expose.

## Healthchecks

Verify the base image ships the tool the check calls. Examples: `pgrep`, `curl`, `wget`. Alpine and slim images often miss these tools. Use `/proc` scans or `wget` as a fallback.

## Known Rails and Puma issues behind Cloudflare Tunnel

- Error `HTTP Origin ... didn't match request.base_url`. Cause: Rails sees the request as HTTP. The browser sent `Origin: https://`. Fix: set `config.assume_ssl = true` through a `configs:` initializer.
- Error `ERR_TOO_MANY_REDIRECTS`. Cause: `force_ssl` is on and Rails does not see `X-Forwarded-Proto: https`. Fix: same as above.
- Error `Puma::HttpParserError: Are you trying to open an SSL connection to a non-SSL Puma?`. Cause: something sends TLS bytes to plain-HTTP Puma. Fix: set the Coolify Domains field to `http://<app>.delulo.uk`.

## Git

- `env.local` and `.env` are gitignored. If `git status` shows one of these files, stop and check `.gitignore`.
- Commit messages follow Conventional Commits. Examples: `feat(<app>): ...`, `chore(<app>): ...`.

## Do not

- Do not run `docker compose up`, `docker compose down`, or `docker compose pull` on the workstation.
- Do not run `docker compose down -v` on the Coolify host. Volumes hold user data.
- Do not print real secrets in chat. Point to `env.local` instead.
