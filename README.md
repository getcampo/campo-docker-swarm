# Campo Docker

This repo help you deploy Campo on docker swram.

If your website have a small number of users, or only have one server, you should read [Single host deployment](). It's simple but do not scale.

If your website have large number of users, and have knowledge about cloud service, please read [Cluster deployment](). It's complex but can be scale as you need.

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
$ git clone https://github.com/getcampo/campo-docker.git campo
$ cd campo
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

Now we need to setup databse. Because docker stack do not privide simple way to run command in container, so first we need to find container ID by this command:

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
