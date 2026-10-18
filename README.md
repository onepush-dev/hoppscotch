# Hoppscotch — open-source API development platform (Postman alternative), Community Edition 2026.9.0. Deploy on Railway with one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/TEMPLATE_CODE)

[Hoppscotch](https://github.com/hoppscotch/hoppscotch) (MIT) is an open-source API client that runs in the browser: REST, GraphQL, WebSocket, SSE, Socket.IO, MQTT and gRPC-web requests, collections, environments, pre-request and test scripts, team workspaces with roles, shared requests, history, mock servers and published API docs. The self-hosted **Community Edition** adds accounts, sync and teams on your own Postgres, plus an **admin dashboard** for users, teams, auth providers and SMTP. This template runs the **official all-in-one image** `hoppscotch/hoppscotch:2026.9.0` (app + admin + backend in one container), pinned by digest, on **one domain** with Railway Postgres. It needs **no deploy-time input**.

## Why this template
- **Current and pinned.** Hoppscotch **2026.9.0** (released 2026-09-30), pinned by tag **and** digest. Railway Postgres is pinned too (`postgres-ssl:18.6@sha256:…`). Nothing floats on `latest`.
- **Migrations run on every start.** The AIO image doesn't migrate the database by itself: upstream tells you to `docker exec` into the container and run `pnpm exec prisma migrate deploy`, and the backend refuses to start on an empty database. The start command here runs `prisma migrate deploy` first, then hands over to the image's own entrypoint, so the first deploy and every upgrade migrate automatically.
- **One domain, no extra services.** `ENABLE_SUBPATH_BASED_ACCESS=true` makes the bundled Caddy serve the app at `/`, the admin dashboard at `/admin`, and the API at `/backend` (GraphQL, WebSocket subscriptions, REST, auth callbacks). This is the mode the Hoppscotch Desktop app needs too.
- **Zero inputs.** `DATA_ENCRYPTION_KEY` (exactly 32 characters) and the desktop-bundle signing secret are generated per deploy. All URLs come from `RAILWAY_PUBLIC_DOMAIN`. The backend generates its own JWT and session secrets in the database on first boot.
- **Auth is set up in the browser, not in variables.** Current Hoppscotch releases keep auth providers and SMTP in the database and configures them in an **onboarding wizard** at `/admin`. You don't redeploy to turn on GitHub, Google, Microsoft or e-mail login.
- **Restarts the way Hoppscotch expects.** On first boot, and every time you save settings in the admin dashboard, the backend writes its config and exits with code 0 so the container restarts with the new settings (upstream uses `restart: unless-stopped`). The service uses restart policy **ALWAYS**. With `ON_FAILURE` the container would stay down after the first boot.
- **Stateless app container.** Everything (users, collections, teams, settings) is in Postgres, so the Hoppscotch service has no volume.

## Services
| Service | Source | Port | Public | Volume | Healthcheck |
|---|---|---|---|---|---|
| Hoppscotch | `hoppscotch/hoppscotch:2026.9.0@sha256:3160f5b3acf49478552a66a0785ee6c75f64311066f149f7d946fbd540f8388d` | domain → **80** (Caddy); `PORT=8080` = backend | yes | none | `/ping` on 8080 (300 s) |
| Postgres | Railway Postgres `ghcr.io/railwayapp-templates/postgres-ssl:18.6@sha256:fcc6686a3f4f1962fe4ed77a15884553b7bd611ae1cea3204687968c5c09d517` | 5432 | no | `/var/lib/postgresql/data` | none |

**Hoppscotch start command** (also in [`railway.toml`](railway.toml)):
```
/bin/sh -c "cd /dist/backend && pnpm exec prisma migrate deploy && exec tini -s -- node /usr/src/app/aio_run.mjs"
```
A Railway start command replaces the image ENTRYPOINT (`tini --`), so the command calls `tini` itself. Restart policy: **ALWAYS**. Postgres keeps the Railway template defaults.

Inside the container: Caddy on **:80** (`/` app, `/admin` admin, `/backend` → backend, `/desktop-app-server` → bundle server), the NestJS backend on **:8080** (`PORT`), and the desktop bundle server on :3200.

## Variables
### Hoppscotch
| Variable | Template value | Notes |
|---|---|---|
| DATABASE_URL | `${{Postgres.DATABASE_URL}}` | private network |
| DATA_ENCRYPTION_KEY | `${{secret(32)}}` | encrypts secrets stored in the DB. Exactly 32 chars. **Never change it** after the first boot |
| WEBAPP_SERVER_SIGNING_SECRET | `${{secret(48)}}` | derives the desktop-bundle signing key, so it survives redeploys without a volume |
| ENABLE_SUBPATH_BASED_ACCESS | `true` | one domain: `/`, `/admin`, `/backend` |
| PORT | `8080` | backend's internal port (Caddy proxies to it). Don't set it to 80 |
| TRUST_PROXY | `true` | real client IPs behind Railway's proxy (rate limiting) |
| WHITELISTED_ORIGINS | `https://${{RAILWAY_PUBLIC_DOMAIN}},app://localhost_3200,app://hoppscotch` | CORS + desktop app origins |
| VITE_BASE_URL | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | |
| VITE_SHORTCODE_BASE_URL | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | shared-request links |
| VITE_ADMIN_URL | `https://${{RAILWAY_PUBLIC_DOMAIN}}/admin` | |
| VITE_BACKEND_GQL_URL | `https://${{RAILWAY_PUBLIC_DOMAIN}}/backend/graphql` | |
| VITE_BACKEND_WS_URL | `wss://${{RAILWAY_PUBLIC_DOMAIN}}/backend/graphql` | GraphQL subscriptions (live sync) |
| VITE_BACKEND_API_URL | `https://${{RAILWAY_PUBLIC_DOMAIN}}/backend/v1` | REST + OAuth callbacks |

`VITE_*` values are injected into the built frontends when the container starts, so changing them only needs a redeploy, not a rebuild.

### Postgres
Stock Railway Postgres variables (`POSTGRES_PASSWORD` = `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}`, `DATABASE_URL` on the private domain). The full list is in [`env.example`](env.example).

## After deploy: first login
Hoppscotch has **no password login**. You sign in with a magic link by e-mail (needs SMTP) or with GitHub, Google or Microsoft OAuth. You pick these in the onboarding wizard:

1. Wait for the Hoppscotch service to turn green (about 1 minute; on the very first boot it seeds its config, exits once and restarts on its own).
2. Open `https://<your-domain>/admin`. A fresh instance opens the **onboarding** wizard. **Do this right away**: until onboarding is finished, anyone who finds the URL can run it.
3. Choose at least one sign-in method and enter its credentials:
   - **GitHub OAuth (fastest):** create an OAuth app at github.com/settings/developers. Homepage `https://<your-domain>`, callback `https://<your-domain>/backend/v1/auth/github/callback`.
   - **Google:** callback `https://<your-domain>/backend/v1/auth/google/callback`.
   - **Microsoft:** callback `https://<your-domain>/backend/v1/auth/microsoft/callback`.
   - **E-mail magic link:** SMTP URL (e.g. `smtps://user:pass@smtp.resend.com:465`) or host/port/user/password, plus a From address. Any SMTP provider works (Resend, Postmark, SES, Mailgun, Gmail app password).
4. Save. The backend restarts itself with the new providers, which takes a few seconds. Until the first account exists you can run onboarding again, for example if you mistyped a client secret.
5. Sign in on `/admin` with that provider **before anyone else signs up**. Hoppscotch makes you the instance admin if your account is the only one when you open the admin dashboard.
6. Open `https://<your-domain>` and sign in to the app. Invite teammates from `/admin` → Users (invite e-mails need SMTP; otherwise teammates just sign in with the OAuth provider). In `/admin` → Settings you can change auth providers, SMTP, the proxy URL, analytics and history later. Each save restarts the backend.

## Notes
- **Desktop app:** point Hoppscotch Desktop at `https://<your-domain>`. If it reports a CORS/origin error, add `app://<your_domain>` to `WHITELISTED_ORIGINS`, with every `.` and `-` replaced by `_` (e.g. `app://hoppscotch_production_ab12_up_railway_app`).
- **Custom domain:** add it to the Hoppscotch service (target port **80**). The URL variables follow `RAILWAY_PUBLIC_DOMAIN`. Then update your OAuth apps' callback URLs.
- **Upgrades:** change the image tag and digest and redeploy. The start command applies new migrations. Back up Postgres first.
- **Proxy:** browser requests to APIs without CORS go through `https://proxy.hoppscotch.io` by default. You can change it or run your own proxy (`PROXY_APP_URL`, or `/admin` → Settings). Or use the browser extension or the desktop app.
- **Resources:** idle, the container uses about 0.5 GB RAM (Caddy + NestJS backend + Go bundle server). Postgres is small.
- Upstream: https://github.com/hoppscotch/hoppscotch · Docs: https://docs.hoppscotch.io/documentation/self-host/community-edition/getting-started
