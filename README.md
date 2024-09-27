# \<web-app-template\>

First find a clear name for your app, and replace the header of this section!

Your app will run into a [podman container](https://podman.io/docs), it will make its deployment easier in our infrastructure.

## Some considerations for app developers

We user the Containerfile instead of the Dockerfile naming convention to follow the 
[Open Container Initiative (OCI)](https://opencontainers.org/) conventions. But if you already know how to work with dockerfiles, it is more or less the same thing.

1. Install the app in the /app folder 

Install your apps in the /app folder, it will make it easier to find our way around if debugging is needed.

2. DB 

If you need a database for the app, make sure that it is compatible with postgres, it is the only supported DB for now. 

Make sure that the app is able to read the username and the password, the location and name of the database from environment variables or from a podman secret.  

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
The app should store its data in the container /data folder, which means that it has to be configurable with an option or an environment variable when the app is started. The /data folder will be mounted from the host inside the container and it will be possible to backup its content. 

## Some considerations for sysadmins deploying the app:
### Deploying the container 
Once the container ready, it needs to be deployed on the webapp VM. You can use this template to deploy it with systemd:
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
	--name <running container name> \
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
`--network slirp4netns:allow_host_loopback=true` will expose localhost on the 10.0.2.2 IP address. This will let the running app inside the container access postgres on that address, on the default port 5432. 
`--label io.containers.autoupdate=registry` will make sure that once a container in available in the registry with the exact same tag, in this example the registry is quay.io/c3genomics/<my container> and the tag latest, it will be downloaded and started, this will let the developers update their app without having to ask a sysadmin, they simply need to push a new container to the registry. Note that the check to run the automatic update is running every 15 minutes.


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
### Reserve space for your new app

You have to create a new volume in the C3G-prod project and mount it on the webapp VM and expose 
that volume to the container so it can store data. Make sure to mount that folder in the 
/data path of the container. That is where we have told the developers they will find their data. 


### Creating a new user and database in postgres 
Every app should have its used in the database so isolation is ensured between apps.
Connect to the posgess server
```
sudo -u postgres psql
```
Ceate the DB and the user for the app to store its data.
```
CREATE DATABASE <MY_APP_DB_NAME>;
CREATE USER <MY_APP_DB_USER> WITH ENCRYPTED PASSWORD '<MY_APP_DB_PW>';
ALTER DATABASE <MY_APP_DB_NAME> OWNER TO <MY_APP_DB_USER>;
GRANT ALL PRIVILEGES ON DATABASE <MY_APP_DB_NAME> TO <MY_APP_DB_USER>;
```

### Adding a route to the nginx server

On the web proxy, the configurations are all in `/etc/nginx/conf.d`. You can create a server directly use the * certificate valid for all the `*.c3g-app.sd4h.ca` addresses installed on the system, for example, for <my new app>:

```
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

Will work out of the box without any extra DNS or certificate configuration. Also `172.16.8.75` is where the webapp are running right now, we will get a proper internal DNS setup at some point, I promise. 


