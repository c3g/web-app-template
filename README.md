# \<web-app-template\>

First find a clear name for your app, and replace the header of this section!

Your app will run into a [podman container](https://podman.io/docs), it will make its deployement easyer in our infratructure.

## Some considerations

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



