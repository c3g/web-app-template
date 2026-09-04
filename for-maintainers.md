# For sysadmins / app maintainers

How to deploy an app built from this template on the shared webapp VM: podman + systemd,
storage, Postgres, nginx routing, and publishing the container. For what the app itself is
expected to do (install path, DB, data folder), see
**[for-app-developers.md](./for-app-developers.md)**.

The podman containers run in user space, the user is `genome`. Right now all the apps and
databases are running on the same host: `prod-appserver-podman-1`.

## Login to the webapp server

After you login to the webapp server, impersonate the genome user, use `machinectl` instead of
`sudo -s` to make sure that all systemd variables are set properly.
```bash
$ bastion_ssh bastion_user@prod-appserver-podman-1
[...]
[bastion_user@tracking-api ~]$ sudo machinectl shell genome@
[...]
[genome@tracking-api ~]$
```

## Deploying the container

Once the container is ready, it needs to be deployed on the webapp VM. You can use this
template to deploy it with systemd:
```service
[Unit]
Description=<My super webapp>
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
ExecStartPre=/bin/rm -f %t/%n.ctr-id
ExecStart=/usr/bin/podman run \
	--cidfile=%t/%n.ctr-id \
	--cgroups=no-conmon \
	--rm \
	--sdnotify=conmon \
	-d \
	--replace \
	--name <running container name> \
	--label io.containers.autoupdate=registry \
	--network slirp4netns:allow_host_loopback=true \
	-p 8088:8080 quay.io/c3genomics/<my container>:latest -w 1
ExecStop=/usr/bin/podman stop --ignore --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm -f --ignore --cidfile=%t/%n.ctr-id
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
```

This configuration has some interesting options:
`--network slirp4netns:allow_host_loopback=true` will expose localhost on the 10.0.2.2 IP
address. This will let the running app inside the container access postgres on that address, on
the default port 5432.
`--label io.containers.autoupdate=registry` will make sure that once a container in available in
the registry with the exact same tag, in this example the registry is
quay.io/c3genomics/<my container> and the tag latest, it will be downloaded and started, this
will let the developers update their app without having to ask a sysadmin, they simply need to
push a new container to the registry. Note that the check to run the automatic update is running
every 15 minutes.

If you run a official container for a remote app, fix the tag to a specific version, of pick a
tag for a version that will receive security update and wont break dependencies, for example,
for wikimedia mariadb we have picked `docker.io/library/mariadb:lts-noble`.

To activate the systemd for the podman container, run:

```bash
cp <my_new_unit>  .config/systemd/user/.
systemctl --user enable --now <my_new_unit>
```

you can check that it is now running with the podman ps command
```bash
podman ps
22fabdd23301  quay.io/c3genomics/<my new webapp>:latest               2 seconds ago Up 2 seconds 0.0.0.0:8080->8080/tcp  <my new webapp>
2f28e1b3c644  quay.io/c3genomics/project_tracking:latest_release  -w 1        7 weeks ago   Up 7 weeks   0.0.0.0:8000->8000/tcp  traking-api
b0e789e48db0  quay.io/c3genomics/parpal:latest                                10 hours ago  Up 10 hours  0.0.0.0:8001->8000/tcp  PARPAL
613c454b4fec  docker.io/sosedoff/pgweb:0.15.0                                 7 hours ago   Up 7 hours   0.0.0.0:8081->8081/tcp  pg-web
```

## Secrets

Any credential an app needs — a DB password, an OIDC client secret, an API key, a private cert
— should be a podman secret, not a plaintext `-e VAR=value` in the systemd unit: a secret's
value never appears in the unit file, in `podman inspect`, or in shell history the way an
inline `-e` value would. See
[for-app-developers.md](./for-app-developers.md#secrets) for what this means on the app's side
(expecting an env var vs. reading a mounted file).

### Creating a secret

```bash
# from a file
podman secret create MY_APP_DB_PASSWORD /path/to/password/file

# or piped in directly, without writing a plaintext file to disk first
printf '%s' 'the-actual-value' | podman secret create MY_APP_DB_PASSWORD -
```

Secrets are immutable once created — to change a value, remove and recreate it, then restart
the container so it re-mounts the new one (a running container keeps using whatever was
mounted at start; it doesn't pick up a recreated secret on its own):

```bash
podman secret rm MY_APP_DB_PASSWORD
printf '%s' 'the-new-value' | podman secret create MY_APP_DB_PASSWORD -
systemctl --user restart <my_new_unit>
```

### Mounting a secret into the container

Add `--secret` to the `podman run` command in the app's systemd unit, alongside the `-v`
volume mount and `--network` flag already there:

```service
ExecStart=/usr/bin/podman run \
	--cidfile=%t/%n.ctr-id \
	--cgroups=no-conmon \
	--rm \
	--sdnotify=conmon \
	-d \
	--replace \
	--name <running container name> \
	--secret MY_APP_DB_PASSWORD,type=env,target=DB_PASSWORD \
	--label io.containers.autoupdate=registry \
	--network slirp4netns:allow_host_loopback=true \
	-p 8088:8080 quay.io/c3genomics/<my container>:latest -w 1
```

`type=env,target=DB_PASSWORD` exposes the secret to the container as a plain env var named
`DB_PASSWORD` — whatever bare name the app actually expects, decoupled from the secret's own
(prefixed, globally-unique — see Naming below) name in `podman secret ls`. This is the default
for most apps here, including third-party/upstream images you don't control the config surface
of: podman injects the value at container start without it ever landing in the unit file,
`podman inspect`, or the image, so there's no need for the app to support anything beyond a
normal env var. Omit `target=` to reuse the secret's own name as the env var name, if they
happen to match.

If an app specifically reads config from a file instead, drop `type=env` — `--secret
MY_APP_DB_PASSWORD` alone mounts it as a file at `/run/secrets/MY_APP_DB_PASSWORD` inside the
container. Only use this when the app already supports reading from a file path; it's not the
default (see for-app-developers.md).

### Naming

`podman secret ls` is one flat namespace shared by every app on the webapp VM, not scoped per
app — always prefix a new secret with the app's name (e.g. `MY_APP_DB_PASSWORD`) so it can't
collide with another app's secret, and so the list stays readable as it grows. (Existing
secrets on the VM predate this convention and use a mix of naming styles — align new ones to
this pattern rather than matching whichever older secret happens to be nearby.)

### Inspecting what exists

```bash
podman secret ls                          # names only, no values
podman secret inspect MY_APP_DB_PASSWORD --showsecret   # reveals the actual value — use with care
```

## Reserve space for your new app

You might have to create a new volume in the C3G-prod project and mount it on the webapp VM and
expose that volume to the container so it can store data. Make sure to mount that folder in the
/data path of the container. That is where we have told the developers they will find their
data.

Once the volume is exposed to the mv, you need to configure it:
```bash
APPNAME=<app name>
X=<partition letter>
parted /dev/vd${X}   --script  mktable gpt
parted /dev/vd${X}   mkpart primary  xfs 0% 100%
mkfs.xfs /dev/vd${X}1

# Label the filesystem so fstab can reference it by LABEL instead of a raw
# device path (device letters like vdb/vdc can shift if disks are
# reattached in a different order — a label doesn't).
# NOTE: XFS labels are capped at 12 characters. If ${APPNAME} is longer,
# this command will fail — use a short abbreviation for the label instead.
xfs_admin -L ${APPNAME} /dev/vd${X}1

mkdir /home/genome/${APPNAME}-volume
echo "LABEL=${APPNAME} /home/genome/${APPNAME}-volume xfs defaults 0 0" >> /etc/fstab
systemctl daemon-reload
mount -a
chown -R genome:genome /home/genome/${APPNAME}-volume

# Troubleshooting: if the volume doesn't mount as expected —
#   ls /dev/disk/by-label   # confirms the kernel/udev registered the label
#   dmesg                   # confirms the kernel saw the disk get attached
```

Then you need to mount the volume in the container so it can access the data, you do that by
adding the `-v` option to the app service file. Replace `${APPNAME}` accordingly:

```service
ExecStart=/usr/bin/podman run \
	--cidfile=%t/%n.ctr-id \
	--cgroups=no-conmon \
	--rm \
	--sdnotify=conmon \
	-d \
	--replace \
	--name <running container name> \
	-v  /home/genome/${APPNAME}-volume:/data:Z
	--label io.containers.autoupdate=registry \
	--network slirp4netns:allow_host_loopback=true \
	-p 8088:8080 quay.io/c3genomics/<my container>:latest -w 1
```
The `:Z` extra option at the end of the mount option is to make sure that the read write and
cgroup permission are set properly.

### Make more space for the volume

You might need to add more space to the webapp volume. The first step is to set a bigger value
to the openstack volume, then connect to the VM.

Check that your volume, here vdh has some free space at its end (here we got a volume from
2145MB to 10.7GB)
```bash
APPNAME=<app name>
X=<partition letter>

# parted -s -a opt /dev/vd${X} "print free"
Model: Virtio Block Device (virtblk)
Disk /dev/vdh: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  2146MB  2145MB  xfs          primary
        2146MB  10.7GB  8591MB  Free Space

```

We will add that whole space to partition Number 1:

```bash
# If parted warns the GPT doesn't reflect the full disk size, fix it first:
sgdisk -e /dev/vd${X}
partprobe /dev/vd${X}
parted /dev/vd${X} "print free"   # confirm warning is gone

# Unmount, resize, remount, grow
umount /home/genome/${APPNAME}-volume
parted -s -a opt /dev/vd${X} "resizepart 1 100%"
mount -a
xfs_growfs /home/genome/${APPNAME}-volume
```
Then make sure that the partition has grown:

```bash
# parted -s -a opt /dev/vd${X} "print free"
Model: Virtio Block Device (virtblk)
Disk /dev/vdh: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
        17.4kB  1049kB  1031kB  Free Space
 1      1049kB  10.7GB  10.7GB  xfs          primary

```
Check also on the genome user running `df -h` you'll see tghe new size.

## Creating a new user and database in postgres

Every app should have its used in the database so isolation is ensured between apps.
Connect to the postgres server
```bash
sudo -u postgres psql
```
Create the DB and the user for the app to store its data.
```sql
CREATE DATABASE <MY_APP_DB_NAME>;
CREATE USER <MY_APP_DB_USER> WITH ENCRYPTED PASSWORD '<MY_APP_DB_PW>';
ALTER DATABASE <MY_APP_DB_NAME> OWNER TO <MY_APP_DB_USER>;
GRANT ALL PRIVILEGES ON DATABASE <MY_APP_DB_NAME> TO <MY_APP_DB_USER>;
```

## Adding a route to the nginx server

On the web proxy, the configurations are all in `/etc/nginx/conf.d`. You can create a server
directly use the * certificate valid for all the `*.c3g-app.sd4h.ca` addresses installed on the
system, for example, for <my new app>:

```bash
server {
     listen 80 ;
     server_name <my new app>.c3g-app.sd4h.ca;
     return 301 https://<my new app>.c3g-app.sd4h.ca$request_uri;

}

server {
  listen 443 ssl;
  server_name <my new app>.c3g-app.sd4h.ca;

  ssl_certificate /etc/letsencrypt/live/c3g-app.sd4h.ca/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/c3g-app.sd4h.ca/privkey.pem;

  


  location / {
 	proxy_set_header    Host $host;
    	proxy_set_header    X-Real-IP $remote_addr;
    	proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    	proxy_set_header    X-Forwarded-Proto $scheme;
	proxy_pass  http://172.16.8.75:8080;
	proxy_read_timeout  20d;
    	proxy_buffering off;

    	proxy_set_header Upgrade $http_upgrade;
    	proxy_set_header Connection $connection_upgrade;
	proxy_http_version 1.1;

    	proxy_redirect      / $scheme://$host/;
      
  }

}

```

Will work out of the box without any extra DNS or certificate configuration. Also
`172.16.8.75` is where the webapp are running right now, we will get a proper internal DNS
setup at some point, I promise.

## GitHub Actions: publishing the container

Setting up the workflow that builds the container and publishes it to `ghcr.io` is a developer
task, done in the app's own repository — see
[for-app-developers.md](./for-app-developers.md#github-actions-build--publish-your-container)
for the template. What matters on your side: it's tagged `latest`, which is exactly the tag
`--label io.containers.autoupdate=registry` (above) polls for — so once that workflow is wired
up in the app's repo, publishing a GitHub Release (the default trigger, so `latest` only moves
on a deliberate release rather than every commit) **is** the full deploy path, with no redeploy
step for you to perform. It also publishes a tag matching the release version (e.g. `v1.2.3`),
so if a bad release needs rolling back, `podman pull ghcr.io/<org>/<repo>:v1.2.2` gets you an
older, known-good image by name. Some apps opt into a continuous-deployment variant instead
(ships on every push to `main`, tagged with the commit SHA rather than a release version) — the
release path above is the default, but don't assume it's universal; check the app's own
workflow file if in doubt.

## Authentication & authorization

How to deploy oauth2-proxy and/or OPA alongside an app, and what changes in nginx for each
pattern. See [README.md](./README.md#authentication--authorization) first for the tier
definitions and concepts referenced below. What's inside the app's own code isn't the focus
here — see [for-app-developers.md](./for-app-developers.md#authentication--authorization) for
that side; this section only covers what you, as the person deploying and running the app, need
to stand up.

This is *additional* to the base deployment above — most of it (container, DB, volume, GitHub
Actions) doesn't change.

### Which tier is this app?

Check with the app's developer, or see
[for-app-developers.md](./for-app-developers.md#decide-your-tier). It determines what you
deploy:

| Tier | What you deploy |
|---|---|
| 1 · Public | Nothing extra. Same as the base template — nginx routes straight to the app. |
| 2 · Gated | An oauth2-proxy container, in front of the app. |
| 3a · Full authz | An oauth2-proxy container **and** an OPA container. |
| 3b · Full authz, non-browser clients | An OPA container only — no oauth2-proxy. The app validates tokens itself. |

### Deploying oauth2-proxy (Tiers 2 and 3a)

Register the app with whoever administers COmanage first — you'll need an issuer URL, a client
ID, and a client secret before this container can start. For Tier 2, also decide the one
COmanage group this app gates on.

```service
[Unit]
Description=<app>-oauth2-proxy
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
ExecStartPre=/bin/rm -f %t/%n.ctr-id
ExecStart=/usr/bin/podman run \
	--cidfile=%t/%n.ctr-id \
	--cgroups=no-conmon \
	--rm \
	--sdnotify=conmon \
	-d \
	--replace \
	--name <app>-oauth2-proxy \
	--network slirp4netns:allow_host_loopback=true \
	-p <proxy-port>:4180 \
	-e OAUTH2_PROXY_PROVIDER=oidc \
	-e OAUTH2_PROXY_OIDC_ISSUER_URL=<COmanage OIDCop issuer URL> \
	-e OAUTH2_PROXY_CLIENT_ID=<client id, registered in COmanage> \
	-e OAUTH2_PROXY_CLIENT_SECRET=<client secret> \
	-e OAUTH2_PROXY_COOKIE_SECRET=<32-byte random string, `openssl rand -base64 32`> \
	-e OAUTH2_PROXY_EMAIL_DOMAINS=* \
	-e OAUTH2_PROXY_OIDC_GROUPS_CLAIM=groups \
	-e OAUTH2_PROXY_ALLOWED_GROUPS=<app's one group — Tier 2 only, omit for 3a> \
	-e OAUTH2_PROXY_PASS_USER_HEADERS=true \
	-e OAUTH2_PROXY_UPSTREAMS=http://10.0.2.2:<app-port> \
	-e OAUTH2_PROXY_HTTP_ADDRESS=0.0.0.0:4180 \
	quay.io/oauth2-proxy/oauth2-proxy:latest
ExecStop=/usr/bin/podman stop --ignore --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm -f --ignore --cidfile=%t/%n.ctr-id
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
```

Notes:

- `OAUTH2_PROXY_UPSTREAMS` points at `10.0.2.2:<app-port>` — same host-loopback address the
  base deployment uses for reaching Postgres, since the app and oauth2-proxy are separate
  containers, not sharing a network namespace, unless you choose to put them in the same
  [pod](https://docs.podman.io/en/latest/markdown/podman-pod.1.html) instead.
- `OAUTH2_PROXY_PASS_USER_HEADERS=true` is what makes oauth2-proxy forward
  `X-Forwarded-Groups` (and `X-Forwarded-User`, `X-Forwarded-Email`) to the app — without it,
  the app gets no group information at all.
- `OAUTH2_PROXY_ALLOWED_GROUPS` is the entire Tier-2 gate. Leave it unset for a 3a app —
  oauth2-proxy's job there is authentication and group-forwarding only; the actual
  fine-grained decision is OPA's, not the proxy's.
- Confirm with whoever administers COmanage what literal string ends up in the `groups` claim
  for a given group (a short name vs. a full COU path) — that string is what
  `OAUTH2_PROXY_ALLOWED_GROUPS` and, downstream, `data.json` in `c3g-opa-policies` both have to
  match exactly.

### Deploying OPA (Tiers 3a and 3b)

```service
[Unit]
Description=<app>-opa
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
ExecStartPre=/bin/rm -f %t/%n.ctr-id
ExecStart=/usr/bin/podman run \
	--cidfile=%t/%n.ctr-id \
	--cgroups=no-conmon \
	--rm \
	--sdnotify=conmon \
	-d \
	--replace \
	--name <app>-opa \
	--network slirp4netns:allow_host_loopback=true \
	-p 127.0.0.1:8181:8181 \
	openpolicyagent/opa:latest \
	run --server --addr 0.0.0.0:8181 --bundle <bundle URL>
ExecStop=/usr/bin/podman stop --ignore --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm -f --ignore --cidfile=%t/%n.ctr-id
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
```

Notes:

- Bind OPA's port to `127.0.0.1` only — it's a local sidecar, never meant to be reachable
  outside the VM. The app reaches it at `10.0.2.2:8181`, same host-loopback pattern as above.
- `--bundle <bundle URL>` is where OPA polls its policy from. `c3g-opa-policies` builds each
  app's bundle with `opa build -b <app-name>/ ... -o <app-name>-bundle.tar.gz` and publishes it
  to an internal bundle endpoint this VM can poll — point `--bundle` at that URL so policy
  updates roll out on their own, without a redeploy. `opa run --server --watch <app-name>/`
  against a local checkout is still the right choice for local development, just not for this
  VM.
- One OPA container per app, not shared — each app's bundle only contains that app's policy
  (see `c3g-opa-policies`' isolation principle).

### nginx: what changes per tier

The base template's nginx config (`/etc/nginx/conf.d`) proxies straight to the app's port.
That's still exactly right for **Tier 1** and **Tier 3b** apps — nginx never needs to know
about auth for those. For **Tier 2** and **Tier 3a**, point `proxy_pass` at the oauth2-proxy
container's port instead of the app's:

```diff
   location / {
 	proxy_set_header    Host $host;
     	proxy_set_header    X-Real-IP $remote_addr;
     	proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
     	proxy_set_header    X-Forwarded-Proto $scheme;
-	proxy_pass  http://172.16.8.75:8080;
+	proxy_pass  http://172.16.8.75:<proxy-port>;
 	proxy_read_timeout  20d;
     	proxy_buffering off;
   }
```

Everything else in the base nginx template — the redirect-to-HTTPS server block, the
`*.c3g-app.sd4h.ca` certificate, the websocket-upgrade headers — is unchanged regardless of
tier.

### Worked example: `project_tracking`

`project_tracking` is a Tier-3b app — see
[c3g-opa-policies](https://github.com/c3g/c3g-opa-policies) for the full design. Operationally,
that means:

- **No oauth2-proxy container** for this app — nginx proxies straight to it, same as a Tier-1
  app. The app validates Bearer tokens itself.
- **One OPA container**, per the section above.
- `pt_cli`, the CLI companion, talks to the app directly with a bearer token (device-code
  login against COmanage) — nothing for you to deploy for that; it's a client-side flow, not a
  service.

Both halves of this design are ready to deploy: the OPA container above with `c3g-flask-opa` in
the app, and `c3g-oidc` validating `pt_cli`'s bearer tokens. Check
[c3g-opa-policies](https://github.com/c3g/c3g-opa-policies) for current status before treating
any of the above as already live in production.
