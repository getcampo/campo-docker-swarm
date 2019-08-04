# Campo Docker

This repo help you deploy Campo on docker swram.

If your website have a small number of users, or only have one server, you should read [Single host deployment](#single-host-deployment). It's simple but do not scale.

If your website have large number of users, and have knowledge about cloud service, please read [Cluster deployment](#cluster-deployment). It's complex but can be scale as you need.

## Prepare

Install docker on server, visit https://docs.docker.com/install/ for more infomation.

## Single host deployment

### Deploy

Run this command to create a single node swarm:

```console
# docker swarm init
```

Do not add swram nodes because this deployment depend on local volume.

Clone this repo:

```console
$ git clone https://github.com/getcampo/campo-docker-swarm.git
$ cd campo-docker-swarm
```

Edit env file:

```console
$ cp .env.example .env
$ vi .env
```

Set SECRET_KEY_BASE, you can generate secret key by `docker run getcampo/campo bin/rails secret`.

Create docker stack:

```console
# docker stack deploy -c docker-compose.yml campo
```

Now we need to setup databse. Because docker stack do not provide simple way to run command in container, so first we need to find container ID by this command:

```console
# docker container ls
```

Find campo_web container name in NAME column, it will looks like `campo_web.1.xyz`, then run this command to setup db:

```console
# docker exec campo_web.1.xyz bin/rails db:setup
```

Open browser, visit http://yourserverip:3000 and you will see campo is running.

### Upgrade

**backup database before upgrade, see backup section**

Update source code:

```console
$ git pull
```

Update service:

```console
# docker stack deploy -c docker-compose.yml campo
# docker exec campo_web.1.xyz bin/rails db:migrate
```

Docker will restart campo to newer version.

### Backup

All data including database, attachments are store in `./data` directory, you can copy & paste directory to backup data.

For safety backup postgres, run this command to backup:

```console
# docker exec campo_web.1.xyz pg_dump -h postgres -U postgres -c campo_production > campo_production.sql
```

And this command to restore:

```console
# docker exec campo_web.1.xyz psql -h postgres -U postgres -d campo_production < campo_production.sql
```

### SSL(Optional)

This deployment integrated certbot to apply Let's Encrypt certificate for free. Before continue, make sure your website is online and have a domain.

Edit './data/nginx/default.conf', uncomment this lines:

```
  # Replace example.com with your domain
  server_name example.com;
  listen 443 ssl;
  ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
```

Replace `example.com` to your domain.

Find certbot container ID:

```console
# docker container ls
```

Because Let's Encrypt has apply rate limit, recommend test apply in staging mode to make sure website is ready.  

Run this command to test apply:

```console
# docker exec campo_certbot.1.xyz certbot certonly --webroot -w /var/www/certbot -d example.com --email youremail@example.com --agree-tos -n --dry-run
```

If this command return success, apply real certificate:

```console
# docker exec campo_certbot.1.xyz certbot certonly --webroot -w /var/www/certbot -d example.com --email youremail@example.com --agree-tos -n
```

Also replace `example.com` to your domain.

If success, restart nginx service:

```console
# docker service update campo_nginx --force
```

Visit your website with `https://`, SSL should take effect.

## Cluster deployment

In cluster deployment, docker swarm do less thing, only run campo web and worker containers, and give the rest to cloud provider, that includes load balance, postgres, redis and storage.

Your can select a Cloud provider providing the above service, such as AWS, Google Cloud, or setup these services yourself(not in this tutorial).

### Prepare cloud service

Prepare following resources:

- Several linux server as you need.
- PostgresSQL
- Redis
- Storage(Only support AWS S3 and Google Cloud Storage)

### Setup docker swarm

Run this command on one server:

```console
# docker swarm init
```

This server will become manager server, and it will show how to add nodes to this swarm. Run this command on other server:

```console
# docker swarm join --token xxx host
```

Replace token and host with your environment. More infomation about docker swarm, read https://docs.docker.com/engine/swarm/ .

Subsequent docker command will be run on manager server.

### Deploy web and worker service

Create file `docker-compose.yml` with this content:

```yml
version: '3.3'

services:
  web:
    image: getcampo/campo:latest
    command: bin/rails server -b 0.0.0.0
    volumes:
      - ./data/uploads:/app/public/uploads
    env_file: .env
    port:
      - 3000:3000
  worker:
    image: getcampo/campo:latest
    command: bundle exec sidekiq
    env_file: .env
```

Create file `.env` with this content:

```
# Use for session encrypt, generate your key by
# docker run getcampo/campo bin/rails secret
SECRET_KEY_BASE=

# PostgreSQL connection
DATABASE_URL=postgres://username:password@host/database

# Redis connection
REDIS_URL=redis://username:password@host:6379/0

# Storage service avaliable: file, aws, gcloud
STORAGE_SERVICE=file

# Storage AWS S3
# STORAGE_AWS_ACCESS_KEY_ID=
# STORAGE_AWS_SECRET_ACCESS_KEY=
# STORAGE_AWS_REGION=
# STORAGE_AWS_BUCKET=

# Storage Google Cloud Storage
# STORAGE_GCLOUD_PROJECT=
# STORAGE_GCLOUD_BUCKET=
# STORAGE_GCLOUD_KEYFILE_CONTENT=
```

> TODO: ENV document

Now start services:

```console
# docker stack deploy -c docker-compose.yml campo
```

Run this command:

```console
# docker container ls
```

Find campo_web container name in NAME column, it will looks like `campo_web.1.xyz`, then run this command to setup db:

```console
# docker exec campo_web.1.xyz bin/rails db:setup
```

Now campo service is running, we need to expose to internet by load balance.

Setup a load balance in your cloud provider, config http and https(optional) frontend, and http backend to servers 3000 port.

Vist load balance's public IP or domain, you will see campo in online.

### Upgrade

**backup database before upgrade**

When Campo release new version, change image tag in `docker-compose.yml`, then run this command to update service:

```console
# docker stack deploy -c docker-compose.yml campo
```

And this command to migarte database:

```console
# docker exec campo_web.1.xyz bin/rails db:migrate
```
