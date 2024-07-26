# \<web-app-template\>

First find a clear name for your app, and replace the header of this section!

Your app will run into a [podman container](https://podman.io/docs), it will make its deployement easyer in our infratructure.

## Some considerations for app developers

We user the Containerfile instead of the Dockerfile naming convention to follow the 
[Open Container Initiative (OCI)](https://opencontainers.org/) conventions. But if you already know how to work with dockerfiles, it is more or less the same thing.

1. Install the app in the /app folder 

Install you app in the /app folder, it will make it easier to find our way around if debugging is needed.

2. DB 
If you need a database for the app, make sure that it is compatible with postgress, it is the only supported DB for now. 

Make sure that the app is able to read the username and the password, the location and name of the database from environement variabled and/or from a podman secret.  

Something in a single var like `MY_APP_DATABASE_URI`
```bash 
MY_APP_DATABASE_URI="postgresql+psycopg2://<POSTGRESS_USER>:<POSTGRESS_PW>@<POSTGRESS_HOST>/<DB_NAME>?client_encoding=utf8"
```
or in many variable like 
```
MY_APP_DB_USER=<USER>
MY_APP_DB_PW=<PASSWORD>
MY_APP_DB_HOST=<HOST:PORT>
MY_APP_DB_NAME=<DB NAME>
```


3. Data
The app should store its data in the container /data folder, that means that it has to be configurable with an option or a environement variable when the app is started. The /data folder will be mounted from the host inside the container and it will be possible to backup its content. 

## Some considerations for sysadmin deploying the app:

Onece the conainer ready, it needs to be deployed on the webapp vm. You can use this template to deploy it with systemd:
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
TimeoutStopSec=70
ExecStartPre=/bin/rm -f %t/%n.ctr-id
ExecStart=/usr/bin/podman run \
	--cidfile=%t/%n.ctr-id \
	--cgroups=no-conmon \
	--rm \
	--sdnotify=conmon \
	-d \
	--replace \
	--name <running conainer name> \
	--label io.containers.autoupdate=registry \
	--secret DATABASE_NAME_ANDPW,type=env \
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
`--network slirp4netns:allow_host_loopback=true` will exposed localhost on the 10.0.2.2 ip adress. This will let the running app inside the container acess postgress on that adress, on the default port 5432. 
`--label io.containers.autoupdate=registry` will make sure that once a container in available in the registry with the exact same tag, in this example the registry is quay.io/c3genomics/<my container> and the tag latest, it will be downloaded and started, this will let the developers update their app witout having to hask a sysadmin, they simply need to push a new container to the registry. Note that the check to run the automatic update is ran every 15 minutes.


To activate the systemd for the podman container, run:

```bash
cp <my_new_unit>  .config/systemd/user/.
systemctl --user enable --now <my_new_unit>
```

you can check that it is now running with the podman ps command
```
podman ps
22fabdd23301  quay.io/c3genomics/<my new webapp>:latest_release               2 seconds ago Up 2 seconds 0.0.0.0:8080->8080/tcp  <my new webapp>
2f28e1b3c644  quay.io/c3genomics/project_tracking:latest_release  -w 1        7 weeks ago   Up 7 weeks   0.0.0.0:8000->8000/tcp  traking-api
b0e789e48db0  quay.io/c3genomics/parpal:latest                                10 hours ago  Up 10 hours  0.0.0.0:8001->8000/tcp  PARPAL
613c454b4fec  docker.io/sosedoff/pgweb:0.15.0                                 7 hours ago   Up 7 hours   0.0.0.0:8081->8081/tcp  pg-web
```
