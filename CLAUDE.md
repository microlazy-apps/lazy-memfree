# lazy-memfree maintainer notes

A 懒猫微服 (LazyCat) wrapper for [MemFree](https://github.com/memfreeme/memfree),
a self-hosted Hybrid AI Search Engine.

## Lazycat appstore identifiers

- **package id**: `cloud.lazycat.app.memfree`
- **app_id**: `5360` (recorded 2026-05-08)
- **subdomain**: `memfree` → `https://memfree.<box-domain>`
- **bootstrap workflow**: when re-running `bootstrap-app.yml` to
  resubmit a fix, pass `app_id=5360` so the workflow skips
  `/app/create` (which would 500 on duplicate package).

## Architecture

Three lazycat services in this stack:

```
                   ┌────────────────────────────────────────────┐
   browser ───────►│ main (this image)                          │
                   │  ┌──nginx :80──┐                           │
                   │  │ /           ├─► next.js :3000           │
                   │  │ /api/.../   │                           │
                   │  │  index/     ├─► vector (bun) :3001 ───┐ │
                   │  │  local-file │                          │ │
                   │  └─────────────┘                          │ │
                   │     supervisord (nodaemon=true)           │ │
                   └────┬───────────────────────────────────────┼┘
                        │ UPSTASH_REDIS_REST_URL                │
                        ▼                                       │
                  ┌──redis-http :80──┐                          │
                  │ hiett/serverless-│                          │
                  │ redis-http       │                          │
                  └──────┬───────────┘                          │
                         │ redis://:pass@redis:6379/0           │
                         ▼                                      │
                    ┌─redis :6379─┐                             │
                    │ requirepass │◄────────────────────────────┘
                    │ aof + rdb   │
                    └─────────────┘
```

**Why three services?**

- `main`: bundles the Next.js frontend + Bun vector service + nginx +
  supervisord into one image. Frontend talks to vector over loopback;
  nginx exposes only `/` (frontend) and `/api/index/local-file`
  (vector — the only vector endpoint the browser hits directly, used
  for file uploads).
- `redis-http`: [hiett/serverless-redis-http](https://github.com/hiett/serverless-redis-http)
  is a self-hosted clone of the Upstash REST API. Memfree uses
  `@upstash/redis` (REST client) and `@auth/upstash-redis-adapter`
  (next-auth adapter) which both speak the Upstash REST protocol —
  not raw Redis. This proxy lets us keep the upstream code unmodified.
- `redis`: regular Redis 7 with `requirepass`, AOF + RDB persistence.

## Required deploy params

- `OPENAI_API_KEY` — used for embeddings (text-embedding-3-large) AND
  default chat. Any OpenAI-compatible key works.
- `OPENAI_BASE_URL` — defaults to `https://api.openai.com/v1`. Switch
  for Azure / DeepSeek / Aliyun Bailian / etc.
- `SERPER_API_KEY` — needed for the live web-search half of the
  hybrid retrieval. Optional in the manifest but the feature is dead
  without it.

All other params (Anthropic / DeepSeek / Qwen / Gemini / OpenRouter /
Jina rerank / GitHub OAuth / Google OAuth) are optional.

## Stable secrets (auto-generated, persist across upgrades)

- `AUTH_SECRET` — next-auth signing key. Reused as `JWT_SECRET` on the
  vector service so vector can decrypt the next-auth session cookie.
- `API_TOKEN` — random token for the frontend → vector internal API.
- `SRH_TOKEN` — bearer token between the upstash REST emulator and
  its clients (frontend's @upstash/redis client, vector's @upstash/redis
  client). Acts as `UPSTASH_REDIS_REST_TOKEN`.
- `REDIS_PASSWORD` — Redis `requirepass`.

## Patches

`patches/01-lazycat-dockerfile.patch` adds 5 new files into `vendor/memfree/`:

- `Dockerfile` — multi-stage. Stage 1 builds `frontend` (`bun run build`).
  Stage 2 installs `vector` deps. Final stage installs nginx + supervisor
  and copies the artifacts.
- `.dockerignore` — drops node_modules, READMEs, .git, etc.
- `nginx.conf` — see architecture diagram. Routes `/api/index/local-file`
  to vector, everything else to next.js.
- `supervisord.conf` — runs nginx + frontend + vector with `nodaemon=true`.
- `lazycat-entrypoint.sh` — creates persist dirs, symlinks
  `/app/vector/data` → `${PERSIST_DIR}/lancedb` so LanceDB tables survive
  restarts, waits for the redis-http emulator to be reachable, then
  execs supervisord.

The upstream tree (frontend/, vector/, etc.) is **not modified**.

## Build-time placeholders

Several upstream modules have `if (!X) throw` guards at module load
time (`frontend/lib/env.ts`). These are read by Next.js during
`bun run build` while it prerenders pages. We satisfy them with
placeholder values in the Dockerfile's `ENV` block:

- `VECTOR_HOST=http://127.0.0.1:3001` (real value at runtime via manifest)
- `UPSTASH_REDIS_REST_URL=http://127.0.0.1:8079` (placeholder)
- `UPSTASH_REDIS_REST_TOKEN=placeholder`
- `API_TOKEN=placeholder`
- `AUTH_SECRET=placeholder`
- `OPENAI_API_KEY=placeholder`
- `NEXT_PUBLIC_APP_URL=""` — intentionally empty so emitted absolute
  URLs fall back to relative paths (works behind any reverse-proxy
  domain). The lazycat box's subdomain varies per user, so we cannot
  bake it in.
- `NEXT_PUBLIC_VECTOR_HOST=""` — same reasoning.

These placeholders are **not** secrets and are **not** used at runtime —
the manifest's `services.main.environment` overrides every one.

## Pre-mirrored images

Mirrored once via `lzc-cli appstore copy-image`:

- `redis:7` — `registry.lazycat.cloud/lee/library/redis:ef2791ca3ea21111`
- `hiett/serverless-redis-http` — `registry.lazycat.cloud/lee/hiett/serverless-redis-http:43935e81cef4c3b4`

If you need to bump either, run:

```sh
lzc-cli appstore copy-image docker.io/library/redis:7-alpine
lzc-cli appstore copy-image docker.io/hiett/serverless-redis-http:latest
```

Then update the digest in `lazycat/lzc-manifest.template.yml`.

## Updating upstream

```sh
git subtree pull --prefix=vendor/memfree \
  https://github.com/memfreeme/memfree.git main --squash
git apply --check patches/*.patch -p1 --directory=vendor/memfree
```

If upstream changes:

- `frontend/package.json` deps that affect `bun install` — patch should
  still apply (we don't touch package.json).
- `frontend/next.config.js` — patch doesn't touch it; runtime should
  still work.
- `vector/index.ts` — patch doesn't touch it.
- The list of `app/api/...` routes — re-audit `nginx.conf` for any new
  routes that need to go to vector vs frontend.

## Local lpk smoke build

```sh
bash ~/lazycat-ci/scripts/build-lpk.sh \
  --image lazy-memfree:dev \
  --version 0.0.0-dev \
  --docker-context ./vendor/memfree \
  --patches-target vendor/memfree
```

The build is heavy (Next.js full build + bun install for vector).
Expect ~10-15 min on a fresh build.

## Auth: Lazycat OIDC integration

The primary login path is the lazycat platform's built-in OIDC
provider, registered automatically when the manifest declares
`application.oidc_redirect_path`. Our `auth.ts` patch adds a custom
`'lazycat'` provider that's only registered when all five
`LAZYCAT_AUTH_OIDC_*` env vars are non-empty (so non-lazycat
deployments — Vercel etc. — keep upstream behavior unchanged).

End-to-end flow:

```
POST /api/auth/signin/lazycat
  → 302 https://<box-domain>/sys/oauth/auth
       ?response_type=code
       &client_id=cloud.lazycat.app.memfree
       &redirect_uri=https://memfree.<box-domain>/api/auth/callback/lazycat
       &scope=openid+profile+email
       &state=...
  → user consents on the lazycat login page
  → 302 back to /api/auth/callback/lazycat?code=...&state=...
  → next-auth exchanges code at LAZYCAT_AUTH_OIDC_TOKEN_URI
  → fetches userinfo at LAZYCAT_AUTH_OIDC_USERINFO_URI
  → upserts the user (sub, email, name, picture) via
    UpstashRedisAdapter and sets the session cookie
```

The lazycat dex line `add oauth client cloud.lazycat.app.memfree
[https://memfree.{box-domain}/api/auth/callback/lazycat]` in
`journalctl` confirms registration at install time. The redirect URI
must match `application.oidc_redirect_path` exactly — next-auth
generates `/api/auth/callback/<provider-id>` by convention, so
`oidc_redirect_path` is hardcoded to `/api/auth/callback/lazycat`.

`AUTH_URL=https://${LAZYCAT_APP_DOMAIN}` and `AUTH_TRUST_HOST=true`
are required so next-auth mints absolute callback URLs against the
real reverse-proxy domain, not `localhost`.

## Other auth providers

- **GitHub / Google OAuth**: optional, configured per-deploy via
  `AUTH_GITHUB_*` / `AUTH_GOOGLE_*` deploy params. Works alongside
  lazycat OIDC.
- **Resend (email magic link)**: requires a Resend API key, currently
  not exposed as a deploy param. Disabled in practice.
- **Google One Tap (`googleonetap` Credentials provider)**: upstream
  calls `${NEXT_PUBLIC_APP_URL}/api/one-tap-login` server-side. With
  `NEXT_PUBLIC_APP_URL=""` this becomes a relative URL, which fails
  server-side. Effectively disabled on lazycat.
- `JWT_SECRET` (vector) is set to the same value as `AUTH_SECRET`
  (next-auth) — the lazycat-entrypoint.sh fills it in if unset.
  They MUST match or vector rejects every authenticated request.

## Subdomain

`memfree` — accessible at `https://memfree.{box-domain}`.
