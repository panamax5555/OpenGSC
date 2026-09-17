# Railway Deployment Notes

This document records the Railway deployment state and decisions for the `panamax5555/OpenGSC` fork so the setup can be reconstructed without relying on chat history.

## Repository

- Upstream: `fenjo26/OpenGSC`
- Fork: `panamax5555/OpenGSC`
- Default branch: `main`
- Hosting target: Railway

## Railway project

- Railway project name: `OpenGSC`
- Railway service name: `opengsc`
- Environment: `production`
- Public service domain: `https://opengsc-production.up.railway.app`
- Container port: `3000`
- Region currently observed: Europe (`europe-west4-drams3a`)

## Current verified status — 2026-09-17

Vanilla OpenGSC is now running successfully on Railway.

Verified from Railway runtime logs:

- deployment status: `SUCCESS`
- `/data` volume mounted successfully
- SQLite database created at `file:/data/prod.db`
- `prisma db push` completed successfully
- Next.js `16.2.12` started and reported `Ready`
- OpenGSC in-process schedulers started (`clarity`, `rank`, `aeo`, `alert`, `digest`, `sync`, `drops-watch`, `serpmon`, `warmup`)
- external smoke test `GET /login` returned HTTP `200`

This establishes a working Railway baseline before any SERPentine feature transplantation.

## Persistent storage

OpenGSC uses SQLite and therefore requires persistent storage.

Railway volume:

- Volume name: `data`
- Mount path: `/data`
- Initial size: `500 MB`
- Database path: `/data/prod.db`
- `DATABASE_URL`: `file:/data/prod.db`

Do not deploy OpenGSC on Railway without the `/data` volume, otherwise the SQLite database would not survive replacement deployments.

## Required environment variables

The Railway service currently uses these base variables:

- `DATABASE_URL=file:/data/prod.db`
- `NODE_ENV=production`
- `PORT=3000`
- `NEXTAUTH_URL=https://opengsc-production.up.railway.app`
- `NEXTAUTH_SECRET=<secret stored in Railway>`

Never commit secrets to this repository.

## Google OAuth / first owner login

OpenGSC's authentication code requires:

- `GOOGLE_CLIENT_ID`
- `GOOGLE_CLIENT_SECRET`

Google OAuth must allow:

```text
Authorized JavaScript origin:
https://opengsc-production.up.railway.app

Authorized redirect URI:
https://opengsc-production.up.railway.app/api/auth/callback/google
```

The first successful Google login becomes the owner automatically. OpenGSC requests Google identity plus read-only Search Console and Analytics scopes and requests offline access for refresh-token support.

Current blocker: the Railway service does not yet have the Google OAuth client ID/secret configured. Until those two secrets are supplied, the application itself can run but owner login/GSC connection cannot be completed.

## Dockerfile / Railway compatibility

The upstream Dockerfile contained:

```dockerfile
VOLUME ["/data"]
```

Railway rejected the Dockerfile during validation because persistent volumes are configured by the Railway service instead of via this Dockerfile instruction.

For the Railway fork, that `VOLUME` instruction was removed. The Railway volume mounted at `/data` provides the persistence instead.

The rest of the upstream Docker flow is intentionally kept as close to upstream as possible.

## Prisma startup compatibility

After the Dockerfile validation issue was fixed, the Docker image built successfully but the container crashed during startup.

The upstream `docker-entrypoint.sh` ran:

```sh
npx prisma db push --skip-generate
```

With Prisma `7.9.1`, `prisma db push` no longer accepts `--skip-generate`, so Railway repeatedly exited with:

```text
! unknown or unexpected option: --skip-generate
```

The Railway fork now runs:

```sh
npx prisma db push
```

The Prisma client is already generated during the Docker build (`npm ci` / `npm run build`), so removing the obsolete flag is sufficient and keeps schema application idempotent at container startup.

## Important Railway deployment behavior

Railway's normal **Redeploy** action rebuilds the commit already associated with the existing deployment. It does not necessarily fetch the latest GitHub `main` commit.

After committing a hosting fix to GitHub, use Railway's command palette:

```text
Ctrl/Cmd + K -> Deploy Latest Commit
```

This ensures the newest commit from the connected GitHub branch is used.

## Deployment history / initial issues

### 1. Dockerfile validation failure

Initial Railway deployments failed before the actual Docker build.

Root cause:

```dockerfile
VOLUME ["/data"]
```

Resolution:

1. Create Railway persistent volume mounted at `/data`.
2. Remove the Dockerfile `VOLUME ["/data"]` instruction in this fork.
3. Deploy the latest GitHub commit rather than redeploying the previous Railway snapshot.

### 2. Container crash after successful image build

The subsequent deployment completed the Next.js/Prisma Docker build successfully, mounted `/data`, then crashed before starting Next.js.

Root cause:

```text
prisma db push --skip-generate
```

Prisma 7.9.1 rejected the obsolete option.

Resolution: remove `--skip-generate` from `docker-entrypoint.sh` and retain `npx prisma db push`.

### 3. Successful baseline deployment

After the two compatibility fixes above, deployment `96c08a5c-e804-4cbb-a3f7-9093b874cf0f` reached `SUCCESS`. SQLite schema initialization and Next.js startup completed cleanly, and `/login` returned HTTP 200.

## Current migration strategy

The intended sequence is:

1. Run OpenGSC on Railway as close to upstream as possible. **Done.**
2. Verify application startup and SQLite persistence. **Done.**
3. Configure Google OAuth and complete the first owner login/GSC connectivity. **Next.**
4. Validate an actual GSC property/sync before modifying application behavior.
5. Only after the vanilla deployment is stable, transplant the useful SERPentine decision-engine components.
6. Keep the original SERPentine project intact until functional parity and data migration are verified.

Potential SERPentine components to integrate later:

- frozen `gsc-googleads-profile-relevance-v2` scoring
- project relevance profiles
- versioned opportunity snapshots
- target-page mapping (`gsc-target-page-v1`, `page-inventory-target-v2`)
- SEO proposal guardrails
- action/task evidence semantics
- follow-up measurement guardrails where they improve OpenGSC's existing Content Operations workflow

Avoid duplicating OpenGSC functionality that already exists, especially GSC sync/auth, general keyword storage, sitemap inventory and basic content-operation functionality.

## Update rule

Whenever hosting, persistence, deployment, authentication or SERPentine-integration architecture changes materially, update this file in the same change set so the repository remains the source of truth.
