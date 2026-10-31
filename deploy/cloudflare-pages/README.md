# Static frontend on Cloudflare Pages

The CoreScope UI is static files. This directory and `scripts/pages/assemble.sh`
let it be served from Cloudflare Pages (or any static host) while the existing
containers keep serving the API and the classic, embedded UI.

This is the **anonymous** build (`AUTH=false`): it reads everything a logged-out
visitor can read, live updates included, and links to the classic site for
logging in and admin work. It needs **no server code change**, only one env var
on the server.

## How it works

- `public/deployment.js` says where the API is. The embedded UI ships it with
  `apiBase: ''` (same origin, as today). `assemble.sh` rewrites it for the
  static build.
- `public/api-base.js` (`window.CSApi`) is the only place that builds API and
  WebSocket URLs. A lint rule keeps raw `fetch('/api…')` out of every other file.
- The anonymous build sends no cookies, sends its three JSON `POST`s as
  `text/plain` (so the browser sends no preflight), and hides login, account and
  admin controls in favour of an "available on classic" link.

## Cloudflare Pages project (Git integration)

These settings live in the Cloudflare dashboard, not in the repository, so
record any change here.

1. Install the Cloudflare Pages GitHub App on this repository only.
2. Create a Pages project from the repository on the Free plan:
   - production branch: `master`;
   - build command: `bash scripts/pages/assemble.sh`;
   - build output directory: `dist/pages`; root directory: blank;
   - build watch paths, include: `public/*`, `scripts/pages/*`,
     `deploy/cloudflare-pages/*`.
3. Environment variables (Production, and Preview if previews are on):

   | Name | Example | Notes |
   |---|---|---|
   | `API_BASE` | `https://analyzer.example.org` | The CoreScope server. https, lowercase, no path |
   | `AUTH` | `false` | Required, no default. `false` until the server supports credentialed CORS |
   | `CHANNEL` | `beta` | `beta`, `next` or `dev` show a badge; `static` shows none |
   | `CLASSIC_URL` | `https://analyzer.example.org` | Optional. Target of the "Open in classic" links |
   | `SKIP_DEPENDENCY_INSTALL` | `1` | `assemble.sh` needs no npm packages |

   `COMMIT` defaults to the first 7 characters of `CF_PAGES_COMMIT_SHA`.
4. Add the custom domain (for example `beta.example.org`). The UI uses absolute
   paths such as `/icons/…`, so it must be served at the root of a hostname.

Pages builds every push to `master` that touches the watch paths. Pull requests
from forks get no preview deployment.

## Server

Append the static origin to the embed CORS list and restart:

```bash
# Keep every origin already listed; add the static one exactly. Never "*".
CORS_ALLOWED_ORIGINS=https://existing-embed.example.org,https://beta.example.org
```

- The env var replaces `corsAllowedOrigins` from `config.json`, so it must list
  every origin, old and new.
- Matching is exact: lowercase, scheme included, no trailing slash.
- The same list admits the origin to `/ws`. A `*` entry does not work for `/ws`.

Check it:

```bash
curl -sI -H "Origin: https://beta.example.org" https://analyzer.example.org/api/stats | grep -i access-control-allow-origin
```

It must print the origin, and no `Access-Control-Allow-Credentials` line.

## Local and CI

```bash
make dev-split
```

The UI on `http://localhost:8788` calls an API-only server on
`http://localhost:3000` cross-origin, with no proxy. CI runs
`tests/e2e/test-split-anonymous-e2e.js` the same way on E2E shard 1.

## Roll back

- **The UI:** in the Pages dashboard, roll back to the previous deployment. To
  hold it there while `master` is fixed, pause automatic production deployments.
- **Access from the static origin:** remove it from `CORS_ALLOWED_ORIGINS` and
  restart. The static UI then shows "Can't reach the CoreScope API"; the classic
  UI and embeds are unaffected.
