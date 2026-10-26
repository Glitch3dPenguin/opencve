# Klabsinfra fork — prebuilt images + Portainer-ready deploy

Fork of [opencve/opencve](https://github.com/opencve/opencve) (license BSL 1.1, unchanged).
Purpose: pull OpenCVE as prebuilt GHCR images instead of building from source, so the
stack deploys from Portainer with no local docker build.

## What this fork adds (additive only — app code untouched, rebases cheaply)

- `.github/workflows/publish-images.yaml` — builds the two upstream images
  (`web/Dockerfile` -> `opencve-web`, `scheduler/Dockerfile` -> `opencve-airflow`)
  with `OPENCVE_VERSION` baked in (upstream's build-time versioning model, kept),
  pushed to GHCR as `ghcr.io/glitch3dpenguin/opencve/opencve-{web,airflow}:<tag>` (+ `:latest`).
  Trigger: manual run (version input) or tag push `v*`.
- `docker/docker-compose.klabsinfra.yaml` — the deploy file: image-based
  (no `build:`), Traefik labels on nginx (Traefik -> nginx:80 -> gunicorn:8000),
  Airflow ops UI bound to 127.0.0.1 by default, plus two idempotent one-shot
  containers that replace `install.sh`'s first-boot steps:
  - `opencve-init` (web image): generates the Django secret key, migrations,
    collectstatic, one-time KB import (10-30 min), creates the admin user.
  - `opencve-kb-init` (airflow image): clones the 5 KB repositories (~30 min),
    registers the Airflow connections, unpauses the feed DAGs.
  Both re-run on every start in seconds after the first boot — that is also how
  DB migrations get applied on upgrades.
- Committed default configs (upstream generates these via `install.sh prepare`;
  committing them makes the deploy self-contained): `docker/.env`,
  `web/opencve/conf/.env`, `web/opencve/conf/settings.py`, `scheduler/airflow.cfg`.
  All values are documented defaults/placeholders — edit `docker/.env` before
  first start.

Upstream `docker/docker-compose.yaml` and `install.sh` are left untouched —
the stock source-build flow still works from this fork.

## Deploy (Portainer)

1. Edit `docker/.env` in the repo (or in Portainer's stack env): `POSTGRES_PASSWORD`
   (+ mirror in `web/opencve/conf/.env`), `DJANGO_SUPERUSER_*`, `TRAEFIK_RULE`,
   `_AIRFLOW_WWW_USER_PASSWORD`.
2. Add GHCR registry credentials to the stack (GitHub username + PAT with `read:packages`).
3. Git-deploy this repo (the compose file must stay in `docker/` — relative paths),
   or paste `docker/docker-compose.klabsinfra.yaml` as a stack file.
4. First boot takes 30-60 min (KB clone + CVE import). The Airflow ops UI
   (`AIRFLOW_WEBSERVER_PORT`, default 8080) is bound to 127.0.0.1 — SSH tunnel in,
   or set `AIRFLOW_BIND_ADDR` to a non-public interface.

## Upgrade

Bump `OPENCVE_VERSION` in `docker/.env` to the new release tag (e.g. `v3.2.0`),
make sure that tag exists in this fork (upstream tags are inherited; push a local
one if needed) so the image is built, then redeploy. The init containers re-apply
migrations automatically. Back up the `postgres-db` volume first (the crown jewel;
the `repositories` volume is re-clonable).

## Keeping in sync with upstream

The fork is additive-only; sync by adding upstream as a remote and merging
`upstream/master` — conflicts should be rare (only in files this fork never
touches). Rebuild images for the new version after syncing.
