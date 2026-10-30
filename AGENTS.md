# Base44 sandbox setup (OneKey monorepo)

The preview runs **`@onekeyhq/web`** (the web wallet) with the webpack dev server on port 3000.
There is no local backend: the app calls OneKey's hosted APIs (market data, etc.), so the preview
needs working network access but no database or credentials.

`docker-compose.base44.yml` is the runbook. Start it with:

```sh
docker compose -f docker-compose.base44.yml up -d
curl -fsS http://localhost:3000/ >/dev/null && echo ok
```

## Non-obvious setup facts

- **Yarn 4 is pinned in the repo** (`.yarn/releases/yarn-4.1.0.cjs`). Corepack is not used; the
  compose startup writes a small `yarn` shim that runs that release.
- **Dependencies are installed at container start** (`yarn install --immutable`) into the
  bind-mounted checkout, because `node_modules/` (≈765 MiB, gitignored) is not in git. First boot
  takes ~5 minutes; later boots re-sync in seconds.
- **The postinstall `web-embed` build is skipped.** `development/scripts/copy-injected.js` runs
  `yarn app:web-embed:build`, a production webpack build that exhausts this sandbox's RAM. Its
  output (`apps/web-embed/web-build`, gitignored) is only consumed when packaging the native
  iOS/Android apps, so the compose command pre-creates that directory to satisfy the existence
  check in `development/scripts/web-embed.js`.
- **`cross-env` / `webpack` live in the hoisted root `node_modules/.bin`.** Without
  `PATH=/app/node_modules/.bin:$PATH`, `yarn workspace @onekeyhq/web start` fails with
  `command not found: cross-env`.
- **`WATCHPACK_POLLING=true`** is required — the bind-mounted source is on a virtual filesystem, so
  inotify alone does not trigger rebuilds.
- The dev server already allows any host (`allowedHosts: ['all']` in
  `development/webpack/webpack.development.config.js`) and binds `0.0.0.0` via the compose command.
- `.env` is created from `.env.example` by `yarn setup:env` during install. Every variable there
  (Sentry DSNs, Covalent key, JPush, …) is optional; the app boots and runs without them.

## Verifying

- `docker compose -f docker-compose.base44.yml ps` → `web` must report `healthy`.
- `curl -s http://localhost:3000/` returns the app HTML (`<title>OneKey</title>`).
- After an edit, the log prints `webpack compiled` and `OneKey Build … completed` — that is the
  hot-reload loop. A full first compile takes 10–20 s.
- Tests/lint must run inside the container, e.g.
  `docker compose -f docker-compose.base44.yml exec -T web yarn jest <path>`.

## Caveats

- The main compile peaks around 6 GB of RAM on this sandbox; an unusually large rebuild can be
  killed. `restart: on-failure` brings the server back.
- Only the web target is set up. Desktop (`apps/desktop`), extension (`apps/ext`), mobile
  (`apps/mobile`) and `apps/web-embed` are untouched.
