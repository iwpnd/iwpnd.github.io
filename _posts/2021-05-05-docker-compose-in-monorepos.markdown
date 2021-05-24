---
layout: post
title: docker-compose in monorepos
tags: [til, monorepo, docker, docker compose]
categories: docker docker-compose monorepo
date: 2021-05-05 13:37:00 +0200
toc: true
---

<p align="center">
<img src="/img/2021-05-05-docker-compose/docker-compose-monorepos.png" alt="docker-compose-monorepos">
</p>

## Monorepo

As I was joining TIER Mobility not only did I have to learn TypeScript quickly, but I also had
to learn how to work with a monorepo - something that first felt weird, but I got to love quickly.
The idea is that instead of having a package for every one of your services with its pre-commit hooks,
lint setups, prettier configs, ci/cd pipelines, and whatnot, every service is instead a package
in a single repository that is being orchestrated with yarn and Lerna. They share the same configurations,
the same dependabot that keeps them up to date and you can even reference them from within each other.
Do you have a `logger` that is used in all of your services? Have it as a package in the monorepo and
import it in your other services.

## Local development environment

One way or another you will want to have your development environment mirror your deployment environment.
For that purpose, and because it comes shipped alongside Docker, you will probably use `docker compose` CLI
and `up` your stack like that.
In our monorepo, we started with a single `docker-compose.yaml` at the top level of our monorepo. We
have our database, our Redis, and maybe something like RabbitMQ, that we tend to use across our stack
consistently. All good and fine.

## Until..

Our monorepo grew, and so did our stack.

There we had a `docker-compose.yml` that up RabbitMQ, Tile38, Redis, some databases, and Kafka - a stack that
brings even the biggest machine to hold (if you run [lofi.cafe](https://lofi.cafe)) next to it.
So instead of having a single `docker-compose.yml`, I created a file for each service within its package
to only start what the service needed.
Yarns task runner is pretty useful to set up each respective `package.json` with a `up`/`down` script.

```
{
  "name": "my-service",
  "scripts": {
    "lint": "eslint --ext .ts src",
    "test": "jest --testMatch '**/*.test.ts'",
    "up": "docker compose up",
    "down": "docker compose down"
  }
}
```

Now if you work on a service, you can use `yarn workspace my-service up` to start the development environment with the dependencies you need for your service, and not those of your monorepo.

## There is one more thing

This is already pretty helpful. However, something like Redis is more often used than not.
That means that

```
  redis:
    image: redis:6.2.3
    ports:
      - 6379:6379
```

occurs in almost every `docker-compose.yml` across the entire monorepo. That means duplication, and you will
have to make sure that the version is updated.
So instead of duplicating a Redis service in every `docker-compose` file, **today I learned** that
[compose configurations can be shared across file and projects](https://docs.docker.com/compose/extends/).

```
// monorepo/docker-compose.yml
version: "3"

services:
  redis:
    image: redis:5.0.6
    ports:
      - 6379:6379

  (...) // other shared dependencies
```

```
// monorepo/packages/my-service/docker-compose.yml
version: "3"

services:
  redis:
    extends:
      file: ../../docker-compose.yml
      service: redis

  (...) // other service dependencies
```

In this setup, Redis is added to the top-level `docker-compose` file and is extended in the services
`docker-compose` file.
If we now up the service with `yarn workspace my-service up` it will check for Redis in the top-level
up it, and continue to add the dependencies as they are defined in the services `docker-compose`.

Do this sooner than later, and your monorepo is on course to grow indefinitely.
