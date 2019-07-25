# Campo Docker

Campo docker deploy template.

Notice: this template only work for single host.

## Usage

### Install Docker

Get docker in https://www.docker.com/ .

### Create docker swarm

```
$ docker swarm init
```

### Create data directory

```
$ mkdir -p data/uploads
$ mkdir -p data/redis
$ mkdir -p data/storage
```

### Edit `.env`

Copy `.env.example` to `.env`, and edit it for your need. Your should set `SECRET_KEY_BASE` here.

### Start service

```
$ docker stack deploy -c docker-compose.yml campo
$ docker run --network=campo_default --env-file=.env getcampo/campo:latest bin/setup
```

Visit http://yourserver.ip:3000 and you will see campo is running.

### Update

When you update image version or change envs, run this commands restart services:

```
$ docker run --network=campo_default --env-file=.env getcampo/campo:latest bin/update
$ docker stack deploy -c docker-compose.yml campo
```
