# Cloudflare cache setup

How `wattsonbot.org` is served, cached, and flushed on deploy.

## Topology

```
visitor ──▶ Cloudflare (proxied, orange cloud) ──▶ GitHub Pages (origin)
                 │
                 └── edge cache holds HTML up to 1h (rule: cache-html-edge)
```

- **DNS / registrar** — nameservers were moved from NameSilo to Cloudflare.
  All DNS records for the zone are now edited in the Cloudflare dashboard,
  **not** at NameSilo.
- **`wattsonbot.org` / `www`** — proxied through Cloudflare (orange cloud).
  Origin is GitHub Pages, building this repo's `main` branch. `CNAME` in the
  repo root pins the custom domain.
- **SSL mode** — Full (strict). GitHub Pages serves a valid cert for the apex,
  so Cloudflare validates the origin chain.
- **`api.wattsonbot.org`** — MUST stay **DNS-only** (grey cloud). It points at
  our own server (nginx + Let's Encrypt) and terminates Telegram webhooks and
  Tesla Fleet telemetry. Proxying it through Cloudflare would break the LE
  HTTP-01 challenge and the webhook TLS. Do not flip it to orange.

## Why an explicit purge is needed

Cloudflare caches the static HTML at the edge with an Edge TTL of ~1h
(page rule / cache rule `cache-html-edge`). A push to `main` rebuilds GitHub
Pages within seconds, but visitors keep getting the stale HTML from Cloudflare
until the TTL expires. Flushing the edge cache right after each deploy makes
changes go live immediately.

## Auto-purge workflow

`.github/workflows/purge-cloudflare.yml` (added in commit `339c4ee`).

- **Triggers**
  - `deployment_status` — fires on every deployment event; the job's `if`
    guard runs the purge only when `state == 'success'` and
    `environment == 'github-pages'` (i.e. the Pages build that follows a push
    to `main` actually finished).
  - `workflow_dispatch` — manual re-run from the Actions tab, e.g. after
    rotating secrets or a one-off cache clear.
- **Action** — one `POST` to the Cloudflare API:
  `POST /client/v4/zones/{CF_ZONE_ID}/purge_cache` with
  `{"purge_everything": true}`. The step fails if the response is not
  `"success":true`.
- **Concurrency** — `group: cf-purge`, `cancel-in-progress: false`, so
  overlapping deploys purge in series instead of cancelling each other.

## Secrets

Set as **repository secrets** (Settings → Secrets and variables → Actions):

| Secret | What it is |
|--------|------------|
| `CF_ZONE_ID` | Zone ID for `wattsonbot.org` (Cloudflare dashboard → Overview, right sidebar). |
| `CF_PURGE_TOKEN` | Cloudflare API token scoped to **Zone → Cache Purge → Purge** for this zone only. |

The workflow fails fast with a clear error if either is missing — that is
exactly what happened on 2026-09-09 for the first Pages deploy after the
workflow landed, before the secrets were added. Adding both secrets and
re-running via `workflow_dispatch` resolved it.

### Rotating the token

1. Create a new token in Cloudflare with the same **Cache Purge** scope.
2. Update `CF_PURGE_TOKEN` in the repo secrets.
3. Trigger the workflow manually (`workflow_dispatch`) to confirm it still
   returns `"success":true`.
4. Revoke the old token.

## Manual purge without CI

```sh
curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/purge_cache" \
  -H "Authorization: Bearer ${CF_PURGE_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{"purge_everything":true}'
```
