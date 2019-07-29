# Campo Docker

Campo docker deploy template.

Notice: this template only work for single host.

## Usage

### Install Docker

Get docker in https://www.docker.com/ .

### Deploy

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

Set SECRET_KEY_BASE, you can generate a secret key by `docker run getcampo/campo bin/rails secret`.

Start service:

```console
$ docker-compose up -d
```

Init db(only first deploy):

```console
$ docker-compose run web bin/rails db:setup
```

Visit http://yourserverip:3000 and you will see campo is running.

### Upgrade

**backup database before upgrade, see backup section**

Update source code:

```console
$ git pull
```

Update service:

```console
$ docker-compose up -d
$ docker-compose run web bin/rails db:migrate
```

Done.

### Backup

All data including database, attachments are store in `./data` directory, you can copy & paste directory to backup data.

For safety bacup postgres, run this command to backup:

```console
$ docker-compose run web pg_dump -h postgres -U postgres -c campo_production > campo_production.sql
```

And this command to restore:

```console
$ docker-compose run web psql -h postgres -U postgres -d campo_production < campo_production.sql
```
