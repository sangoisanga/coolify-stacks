# coolify-stacks

Docker Compose stacks deployed through [Coolify](https://coolify.io/).

## Layout

```
apps/
  <app-name>/
    docker-compose.yml   # committed
    env.example          # committed, placeholder values
    env.local            # gitignored, real secrets
```

## Ingress

Cloudflare Tunnel handles `*.delulo.uk`. The tunnel loops back to Traefik on the Coolify host. Traefik routes to the app by Host header.

Reference: <https://coolify.io/docs/integrations/networking/cloudflare/tunnels>

Per app:

- Coolify **Domains** field: `http://<app>.delulo.uk`.
- Compose file: `expose: 3000` only. No `ports:`.

## Env

- `env.example`: placeholders, committed.
- `env.local`: real values, gitignored. Paste into Coolify Environment Variables UI.

## Apps

| App | Notes |
|---|---|
| `apps/dawarich` | [Dawarich](https://dawarich.app/). Rails app. Needs `assume_ssl` initializer shipped via `configs:`. |
| `apps/hermes` | Hermes agent gateway. |
