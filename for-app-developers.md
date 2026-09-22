# For app developers

Everything to build into the app itself. For concepts and definitions referenced below (tiers,
OIDC, OPA, etc.) see **[README.md](./README.md)** — read that first if you haven't. For how the
app actually gets deployed, DNS/nginx routing, and scaling storage, see
**[for-maintainers.md](./for-maintainers.md)**.

## Installation location

Install your app in the `/app` folder — it makes it easier to find our way around if debugging
is needed.

## DB

If you need a database for the app, make sure that it is compatible with Postgres, it is the
only supported DB for now.

Make sure that the app is able to read the username and the password, the location and name of
the database from environment variables or from a podman secret (see
[Secrets](#secrets) below — this is what most apps should actually use for the password part).

Something in a single var like `MY_APP_DATABASE_URI`
```bash
MY_APP_DATABASE_URI="postgresql+psycopg2://<POSTGRESS_USER>:<POSTGRESS_PW>@<POSTGRESS_HOST>/<DB_NAME>?client_encoding=utf8"
```
or in many variables like
```bash
MY_APP_DB_USER=<USER>
MY_APP_DB_PW=<PASSWORD>
MY_APP_DB_HOST=<HOST:PORT>
MY_APP_DB_NAME=<DB NAME>
```

## Secrets

Any credential your app needs — a DB password, an OIDC client secret, an API key, a private
cert — should come from a podman secret, not a literal value baked into the systemd unit's
`-e VAR=value`. See [for-maintainers.md](./for-maintainers.md#secrets) for how secrets actually
get created and mounted; this is what it means for your app's code.

- **Default: expect a plain environment variable.** A maintainer mounts a secret as an env var
  (`--secret NAME,type=env,target=YOUR_ENV_VAR`) — podman injects the value into the process
  environment at container start, without it ever appearing in the unit file, `podman
  inspect`, or the image. Your app just reads a normal env var; no special file-reading support
  needed in your code. This matters most for anything you don't fully control the config
  surface of (a stock upstream image, a third-party base) — most things running on the shared
  webapp VM today already expect config as plain env vars, and this needs no code change to
  work with that.
- **A file mount is also available**, if your app already reads config from a file path, or
  you specifically want a value to never touch the process environment (e.g. to keep it out of
  a stack trace or a debug endpoint that dumps `os.environ`). `--secret NAME` without
  `type=env` mounts it at `/run/secrets/<secret-name>` instead. Prefer this only when your app
  already supports reading from a file — don't build a `_FILE`-suffix convention into a new
  app just to use it; the env var route above needs no such convention.
- **The secret's own name and the env var your app sees don't have to match.** `--secret
  MY_APP_OIDC_CLIENT_ID,type=env,target=OIDC_CLIENT_ID` names the podman secret
  `MY_APP_OIDC_CLIENT_ID` (unique, prefixed — see below) but exposes it to your app as
  `OIDC_CLIENT_ID`, or whatever bare name your app's own config already expects. Tell your
  maintainer both names: the secret name they should create, and the `target` env var your app
  actually reads.
- **Name secrets uniquely, prefixed with your app's name** (e.g. `MY_APP_DB_PASSWORD`,
  `MY_APP_OIDC_CLIENT_SECRET`). `podman secret ls` on the shared webapp VM is one flat
  namespace across every app, not scoped per app — a generic name like `DB_PASSWORD` risks
  colliding with another app's secret. The `target` env var name doesn't need this prefix —
  it's scoped to your own app's process environment already.
- **Agree on the exact secret name(s) and target env var(s) with your maintainer before first
  deploy.** Creating and mounting secrets is their job (see for-maintainers.md), but which
  values your app needs, and what env var it expects each one under, is your call.

## Data

The app should store its data in the container `/data` folder, which means that it has to be
configurable with an option or an environment variable when the app is started. The `/data`
folder will be mounted from the host inside the container and it will be possible to backup its
content.

## GitHub Actions: build & publish your container

Add this workflow to your own app's repository (not this template) so it builds your
`Containerfile` and publishes the image to C3G's registry, `ghcr.io`. Save it as
`.github/workflows/publish-container.yml`. By default it triggers on a **published GitHub
Release**, not on every push to `main` — releases force a deliberate "this version is ready"
moment (with release notes, in the GitHub UI) instead of shipping whatever the latest commit
happens to be.

```yaml
name: Publish Container

on:
  release:
    types: [published]

env:
  REGISTRY: ghcr.io
  REGISTRY_USER: ${{ github.actor }}
  REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
  IMAGE_NAME: ${{ github.repository }}

jobs:
  release:
    runs-on: ubuntu-latest

    # Sets the permissions granted to the `GITHUB_TOKEN` for the actions in this job.
    permissions:
      contents: write
      packages: write
      attestations: write
      id-token: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      - name: Setup Git
        run: |
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git config user.name "github-actions[bot]"
      # This can be ignored if you don't plan to build a container on multiple architectures
      ##
      - name: Update package lists
        run: sudo apt-get update
      - name: Install QEMU
        run: sudo apt-get install --fix-missing qemu-user-static
      ##
      # There is an example with 2 architectures, amd64 and arm64. Adapt to your needs
      - name: Build Container Image
        uses: redhat-actions/buildah-build@v2
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: latest ${{ github.event.release.tag_name }}
          # Make sure to adapt the path to your Containerfile
          containerfiles: ./Containerfile
          archs: amd64,arm64
      - name: Push to repo
        uses: redhat-actions/push-to-registry@v2
        with:
          username: ${{ env.REGISTRY_USER }}
          password: ${{ env.REGISTRY_PASSWORD }}
          registry: ${{ env.REGISTRY }}
          image: ${{ env.IMAGE_NAME }}
          tags: latest ${{ github.event.release.tag_name }}
```

Cutting a release: tag a commit (e.g. `v1.2.3`), then publish a GitHub Release against that tag
— either from the repo's "Releases" page or with `gh release create v1.2.3`. That publish event
is what triggers the workflow.

Notes:

- This publishes two tags per release: `latest` — the moving tag
  `io.containers.autoupdate=registry` polls (see
  [for-maintainers.md](./for-maintainers.md#deploying-the-container)) — and the release's own
  tag name (e.g. `v1.2.3`), an immutable, human-readable tag pinned to that exact release.
  Since `latest` only moves when you cut a release, it's never "whatever the last commit was" —
  and if a release turns out bad, `podman pull ghcr.io/<org>/<repo>:v1.2.2` gets you back to a
  known-good version by name instead of hunting for a commit SHA.
- Because of the autoupdate label on the maintainer's side, publishing a release **is** the
  full deploy step — podman on the webapp VM picks up the new `latest` image on its own (polled
  roughly every 15 minutes). Nothing else to trigger.
- **Want continuous deployment instead** — every push to `main` ships, no release step? Some
  apps prefer that, and it's a fair trade against "every commit is a proper release": swap the
  trigger for `on: push: branches: ['main']`, and swap `${{ github.event.release.tag_name }}`
  for `${{ github.sha }}` in both `tags:` lines (a release doesn't exist yet at push time, so
  there's no version name to use — the commit SHA is the natural pinning tag for this variant
  instead).
- The multi-arch build (QEMU + `archs: amd64,arm64`) can be dropped if you only need one
  architecture — see the comments in the workflow. Adjust `containerfiles: ./Containerfile` if
  your Containerfile lives somewhere else in the repo.

## Handing off to a maintainer

Deploying your app on the shared webapp VM is a maintainer task (see
[for-maintainers.md](./for-maintainers.md)) — but they're writing a systemd unit around a
container they didn't build, so what you hand them matters. When you reach out for a first
deploy (or a change to how the app runs), share:

- **The exact `podman run` command you actually use to run the container yourself** — every
  flag: ports, volumes, env vars, `--secret` flags, everything. This is the single most useful
  thing you can hand over, because it's what actually works, not something reconstructed from
  your Containerfile or guessed at. Copy it verbatim from your shell history or a `run.sh` you
  keep around — don't retype it from memory, and don't hand over a Containerfile alone and
  expect the run flags to be inferred from it.
- The registry/image name (see [GitHub Actions](#github-actions-build--publish-your-container)
  above) and whether your app uses the default release-publish trigger or the
  continuous-deployment variant — it decides which tag (a version, or `latest`) they should
  expect to see move.
- Your database requirements (see [DB](#db) above) and the exact secret names and target env
  vars you need (see [Secrets](#secrets) above), if you haven't already agreed on those.

## Authentication & authorization

Deciding whether your app needs authentication/authorization, and which pattern to use. See
[README.md](./README.md#authentication--authorization) first for the tier definitions and
concepts referenced here. For how these patterns actually get deployed, see
**[for-maintainers.md](./for-maintainers.md#authentication--authorization)**.

### Decide your tier

Ask, in order:

1. **Does this app need to be public?** → Tier 1. Nothing to build. Talk to a maintainer about
   nginx routing and stop reading here.
2. **Does every user just need a yes/no check — no different permissions between users who are
   let in?** → Tier 2. Nothing to build in your app either — the whole check happens at
   oauth2-proxy, in front of it. If you want cosmetic differences (hide an admin tab from
   non-admins), see [the cosmetic layer](#cosmetic-tier-2-in-app-ui) below, but it's optional
   and it's not a security boundary.
3. **Does this app need different users to be able to do different things — some routes or
   actions restricted to specific groups, others open to more?** → Tier 3. Keep reading.

### Tier 3, by app type

#### Flask

Adopt **`c3g-flask-opa`** — a small library, the one piece every Tier-3 Flask app uses
regardless of 3a or 3b. It's a `before_request` hook/decorator built from two things you
provide:

- a **groups source** — a callable returning the current request's groups. For a 3a app,
  that's reading the `X-Forwarded-Groups` header oauth2-proxy already sets. For a 3b app,
  that's reading `g.groups`, populated by `c3g-oidc` after validating a Bearer token.
- an **input builder** — since each app's OPA input shape differs. `project_tracking`'s is
  `{method, path, groups_header}`; yours might be simpler or different. `c3g-opa-policies`'
  own isolation principle is explicit: one input schema per app, don't couple apps together.

`c3g-flask-opa` calls your app's local OPA sidecar with that input and enforces the result. If
OPA is unreachable, the request is **denied**, not allowed through — an outage should fail
safe, not fail open.

**Do you need 3a or 3b?** Default to 3a (oauth2-proxy stays, add `c3g-flask-opa` behind it) —
it's less code, and correct for any app where every client is a person in a browser. Only move
to 3b (drop oauth2-proxy, adopt `c3g-oidc` too) if your app has a real non-browser client — a
CLI, a script, service-to-service calls — that a cookie-based proxy gate can't serve well.
`project_tracking` is 3b specifically because of `pt_cli`, its CLI, which needs a bearer token
it can obtain without an embedded browser (device-code flow) rather than emulating a browser
login to capture a cookie.

`c3g-oidc`, when you need it, has two independent halves: `server/` validates a Bearer token
in your Flask app (signature, expiry, issuer, audience — built on
[Authlib](https://authlib.org)) and hands `c3g-flask-opa` a plain list of groups; `client/` has
no Flask dependency and is what a CLI companion uses to get a token via device-code login,
store it, and refresh it automatically.

#### R Shiny

Most Shiny apps almost certainly don't need real Tier 3. Shiny apps are single-page reactive
UIs, not multi-route APIs — "per-route" decisions rarely apply the way they do in a REST API.
What most Shiny apps actually want is **Tier 2 with a cosmetic edge**: oauth2-proxy's coarse
gate, plus conditional UI inside the app based on the same groups — which needs no library at
all.

##### Cosmetic Tier 2 in-app UI

To be precise about the mechanics: your app never sees oauth2-proxy's session cookie — that
stays between the browser and oauth2-proxy. What your app reads, via
`session$request$HTTP_X_FORWARDED_GROUPS`, is a plain header on requests that already made it
through. And at this level, your app isn't deciding access — oauth2-proxy's gate already did,
before the request reached Shiny at all. You're only choosing what to *show*, not who gets
*in*.

```r
groups <- strsplit(session$request$HTTP_X_FORWARDED_GROUPS, ",")[[1]]
if ("admins" %in% groups) {
  # show an admin-only tab, button, etc.
}
```

That's real authz — just ad hoc, hand-rolled per app, and reinvented each time someone builds
the next one.

##### Real Tier 3 for Shiny

If your app genuinely needs per-action decisions against OPA, adopt the shared package
**`c3g.shiny.authz`** (dotted, not hyphenated — R's package-naming convention, unlike the
Python packages above), which exports two independent layers so an app takes on only what it
needs:

| Layer | Function | Does | Needs OPA |
|---|---|---|---|
| Tier 2, cosmetic | `c3g_groups()` | Reads and parses the forwarded-groups header from `session$request`, once, correctly. | No |
| Tier 3, real authz | `c3g_require_authz(action, session)` | Called at the top of a guarded `observeEvent`/`downloadHandler`. Builds a `{action, groups}` input (no `path`/`method` — Shiny has no routes to key off), calls the app's local OPA sidecar, fails closed. | Yes |

No OIDC-client layer exists for Shiny — oauth2-proxy stays in front regardless of layer, since
Shiny has no non-browser client the way `pt_cli` justifies one for `project_tracking`. That
keeps the Shiny side to one small package, not two.

```r
observeEvent(input$export_btn, {
  c3g_require_authz("export_dataset", session)  # stops here, notifies the user, on deny
  # ... proceed with the export
})
```

#### Other languages

The pattern is language-agnostic; the libraries above aren't. OPA speaks plain HTTP — any
language can `POST` to its REST API. If you're building in something other than Flask or
Shiny and need real Tier 3, the recipe transfers directly: a pluggable groups source, a
pluggable per-app input builder, a fail-closed call to your local OPA sidecar. That's a small,
straight-line reimplementation — not a reason to force your app into Flask, and not something
worth building speculatively before a real case shows up.

### Summary

| Tier | Do you need to write anything? |
|---|---|
| 1 · Public | No — talk to a maintainer about routing. |
| 2 · Gated | No — the gate is entirely oauth2-proxy config. Optionally read the forwarded-groups header for cosmetic UI. |
| 3a · Full authz, browser-only | Yes — adopt `c3g-flask-opa` (or the Shiny/other-language equivalent). |
| 3b · Full authz, non-browser clients too | Yes — adopt `c3g-flask-opa` **and** `c3g-oidc`. |
