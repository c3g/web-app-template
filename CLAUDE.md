# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **template repository**, not a runnable application. `Containerfile` and `entrypoint.sh` are currently empty stubs meant to be filled in when a new app is bootstrapped from this template. There is no build system, dependency manifest, or test suite here — everything below is the set of conventions a real app built from this template must follow so it can be deployed into the shared C3G infrastructure.

## Documentation layout

Three files, split by audience — read the one that matches the task at hand:

- `README.md` — light: concepts and definitions only (infra overview with one mermaid diagram per tier, auth tiers, COmanage/OIDC/oauth2-proxy/OPA definitions). Not the source of truth for how-to details. If a tier's shape changes, update both its "Infrastructure at a glance" diagram and the combined diagram under "The big picture".
- `for-app-developers.md` — everything to build into the app: install path, DB, secrets, data folder, the GitHub Actions workflow that builds and publishes the container to `ghcr.io`, what to hand a maintainer for first deploy (their own working `podman run` command, above all), and the full authn/authz decision guide by app type (Flask, R Shiny, other).
- `for-maintainers.md` — everything to deploy/run the app on the shared webapp VM: podman/systemd, secrets, storage volumes, Postgres, nginx routing, and the full authn/authz deployment guide (oauth2-proxy/OPA containers, nginx changes per tier).

When updating conventions, edit the relevant audience-specific file, not `README.md`. Each of the two detailed files links back to `README.md` for shared concepts and to each other for the other audience's half of a topic (e.g. authn/authz) — keep those cross-links in sync if a section moves.

## Containerfile conventions

- Named `Containerfile`, not `Dockerfile`, to follow the [OCI](https://opencontainers.org/) naming convention (behaves like a Dockerfile otherwise; `podman build` / `docker build` both work).
- Install the app under `/app` inside the container — this is where sysadmins expect to find it when debugging.
- The app must store its data under `/data` inside the container. This path is mounted from the host at deploy time so it can be backed up; make the path configurable via a flag or env var but default it to `/data`.
- If the app needs a database, it must be Postgres-compatible (the only DB supported on the shared infra). The app must read DB connection info from environment variables (or a podman secret) — either a single DSN-style var (e.g. `MY_APP_DATABASE_URI=postgresql+psycopg2://<user>:<pw>@<host>/<db>?client_encoding=utf8`) or discrete vars (`MY_APP_DB_USER`, `MY_APP_DB_PW`, `MY_APP_DB_HOST`, `MY_APP_DB_NAME`).
- Any credential (DB password, OIDC client secret, API key, cert) should be a podman secret, not a plaintext `-e VAR=value`. Default is `--secret NAME,type=env,target=ENV_VAR` — mounted as a plain env var, not a file, since most apps (including third-party/upstream images) already expect config as env vars and this needs no code change; a file mount (`--secret NAME`, no `type=env`) is the exception, for apps that specifically read from a file path. Secret names must be prefixed with the app's name (e.g. `MY_APP_DB_PASSWORD`): `podman secret ls` is one flat namespace shared by every app on the VM, not scoped per app — `target=` decouples that prefixed name from whatever bare env var name the app actually expects.

## Deployment architecture

Apps built from this template are deployed as rootless Podman containers, managed by systemd user units, on a shared webapp VM (`172.16.8.75`) running under the `genome` user. Key points for anyone authoring deploy configs for such an app (full templates in `for-maintainers.md`):

- Login flow: SSH to the bastion, then to the webapp host via its bastion DNS name `prod-appserver-podman-1`, then `sudo machinectl shell genome@` (not `sudo -s`) so systemd user variables are set correctly. Everywhere else (nginx `proxy_pass`, `OAUTH2_PROXY_UPSTREAMS`, etc.) the same host is referenced by its IP, `172.16.8.75` — the two aren't interchangeable typos, just different names for the same box used in different contexts.
- Each app gets a systemd user unit with:
  - `--label io.containers.autoupdate=registry` — podman auto-update polls the registry (every ~15 min) and redeploys on a new image with the same tag, so a registry push is enough to ship without sysadmin involvement.
  - `--network slirp4netns:allow_host_loopback=true` — lets the container reach host-networked Postgres via `10.0.2.2:5432`.
  - `-v /home/genome/<APPNAME>-volume:/data:Z` — bind-mounts the app's persistent volume onto the container's `/data`.
  - Pin third-party/upstream images (non-C3G) to a specific tag/version rather than `latest`.
  - Units are copied to `~/.config/systemd/user/` and started with `systemctl --user enable --now <unit>`.
- Persistent storage: a dedicated OpenStack volume is partitioned as GPT/XFS, labeled with a ≤12-character label (XFS label limit) matching the app name, and mounted at `/home/genome/<APPNAME>-volume` via `/etc/fstab` (`LABEL=` reference, not a raw device path, since device letters can shift).
- Each app gets its own Postgres DB and role (`CREATE DATABASE`, `CREATE USER ... WITH ENCRYPTED PASSWORD`, `ALTER DATABASE ... OWNER TO ...`, `GRANT ALL PRIVILEGES ON DATABASE ...`) for isolation between apps.
- Routing: nginx on the proxy host (configs in `/etc/nginx/conf.d`) terminates TLS using the wildcard cert for `*.c3g-app.sd4h.ca` and reverse-proxies to `172.16.8.75:<app port>`. New apps just need an nginx server block — no extra DNS/cert work required. For Tier 2 and 3a apps, `proxy_pass` points at the oauth2-proxy container's port instead of the app's; Tier 1 and 3b go straight to the app.
- CI/CD: developers add `.github/workflows/publish-container.yml` (template in `for-app-developers.md`) to their own repo — it builds the `Containerfile` with buildah for `amd64,arm64` and pushes to `ghcr.io`. The template's **default trigger is a published GitHub Release** (not every push to `main`), tagging the image both `latest` and with the release version (e.g. `v1.2.3`) — a deliberate release is what ships, and the version tag doubles as a named rollback target (`podman pull ghcr.io/<org>/<repo>:v1.2.2`). An app can opt into continuous deployment instead (trigger on push to `main`, tag with the commit SHA) — don't assume every app's workflow uses the release trigger. Either way, because of the autoupdate label above, a successful publish is the full release path — no manual redeploy step.

## Authentication & authorization (optional, per app)

Covered across `README.md` (concepts, tiers, definitions), `for-app-developers.md` (decision
guide by app type), and `for-maintainers.md` (oauth2-proxy/OPA deployment) — each links to the
others' relevant section rather than duplicating it. Three tiers, chosen per app, not layered:

1. **Public** — nginx → app directly, no identity check.
2. **Gated** — oauth2-proxy alone in front of the app, gated on one COmanage group. Zero app code; app reads groups from the `X-Forwarded-Groups` header if it wants cosmetic UI differences.
3. **Full authz** — per-route/per-action decisions via OPA, behind either oauth2-proxy (3a — default, browser-only clients) or an in-app OIDC client (3b — only if the app has non-browser clients, e.g. a CLI using device-code login).

Tooling: `c3g-flask-opa` (Flask) and `c3g.shiny.authz` (R Shiny) call out to a per-app OPA sidecar and fail closed if OPA is unreachable (the Shiny one takes a required `policy_path`, e.g. `"my_app/authz/allow"`); `c3g-oidc` provides the in-app OIDC client for Tier 3b (`server/` validates Bearer tokens, `client/` does device-code login for CLIs). R Shiny apps are always behind oauth2-proxy (no 3b). Each app gets its own OPA container that polls its policy bundle, built per app from `c3g-opa-policies`, via `--bundle <URL>`.

From Tier 2 up, each app runs in its own podman pod (a `pod-<app>.service` unit; the containers join it with `--pod-id-file`) with its oauth2-proxy and/or OPA containers. Inside the pod they reach each other at `127.0.0.1` (OPA on `127.0.0.1:8181`, the libraries' default URL). The pod, not the containers, sets `--network` and publishes exactly one port: oauth2-proxy's for Tiers 2/3a, the app's for 3b. Never publish OPA's port or, behind oauth2-proxy, the app's: every container on the VM can reach host ports through `10.0.2.2`, OPA has no authentication, and a reachable app port lets anyone forge `X-Forwarded-Groups`.

Where they live: `c3g-flask-opa` and `c3g-oidc` are public repos published on PyPI; `c3g.shiny.authz` is a public repo installed from a git tag (`remotes::install_github("c3g/c3g.shiny.authz@vX.Y.Z")`); `c3g-opa-policies` is **private**, so never quote real group names or rules from it in this public repo. All are pre-1.0.

`project_tracking` (with its CLI `pt_cli`) is the worked Tier 3b example and the intended first adopter, but it hasn't migrated yet: no app runs Tier 3 in production, so don't describe it as live.
