---
title: 'Installation with Docker'
---

This guide will step you through setting up your own self-hosted version of Sentry in [Docker](https://www.docker.com/).

## Dependencies

-   [Docker version 1.10+](https://www.docker.com/getdocker)

## Building Container

Start by cloning or forking [getsentry/onpremise](https://github.com/getsentry/onpremise). This will act the base for your own custom Sentry.

Inside of this repository, we have a `sentry.conf.py` and `config.yml` ready for [_customizing_]({%- link _documentation/server/config.md -%}).

On top of that, we have a `requirements.txt` file for [_installing plugins_]({%- link _documentation/server/plugins.md -%}).

Now we need to build our custom image. We have two ways to do this, depending on your environment. If you have a custom registry you’re going to need to push to:

```bash
REPOSITORY=registry.example.com/sentry make build push
```

If not, you can just build locally:

```bash
make build
```

If you plan on running the dependent services in Docker as well, we support linking containers.

## Running Dependent Services

_Running the dependent services in Docker is entirely optional_, but you may, and we fully support linking containers to get up and running quickly.

### Redis

```bash
docker run \
  --detach \
  --name sentry-redis \
  redis:3.2-alpine
```

### PostgreSQL

```bash
docker run \
  --detach \
  --name sentry-postgres \
  --env POSTGRES_PASSWORD=secret \
  --env POSTGRES_USER=sentry \
  postgres:9.6
```

_Note that Postgres versions 9.5, 9.6 are the only tested and recommended versions._

### Outbound Email

```bash
docker run \
  --detach \
  --name sentry-smtp \
  tianon/exim4
```

## Running Sentry Services

{% capture __alert_content -%}
The image that is built, acts as the entrypoint for all running pieces for the Sentry application and the same image must be used for all containers.
{%- endcapture -%}
{%- include components/alert.html
  title="Note"
  content=__alert_content
%}

`${REPOSITORY}` corresponds to the name used when building your image in the previous step. If this wasn’t specified, the default is `sentry-onpremise`. To test that the image is working correctly, you can do:

```bash
docker run \
  --rm ${REPOSITORY} \
  --help
```

You should see the Sentry help output.

At this point, you’ll need to generate a `secret-key` value. You can do that with:

```bash
docker run \
  --rm ${REPOSITORY} \
  config generate-secret-key
```

This value can be put into your `config.yml`, or as an environment variable `SENTRY_SECRET_KEY`. If putting into `config.yml`, you must rebuild your image.

For all future Sentry command invocations, you must have all the necessary container links, mounted volumes, and the same environment variables. If different components are running with different configurations, Sentry will likely have unexpected behaviors.

The base for running commands will look something like:

```bash
docker run \
  --detach \
  --link sentry-redis:redis \
  --link sentry-postgres:postgres \
  --link sentry-smtp:smtp \
  --env SENTRY_SECRET_KEY=${SENTRY_SECRET_KEY} \
  ${REPOSITORY} \
  <command>
```

{% capture __alert_content -%}
Further documentation will not mention container links or environment variables for sake of brevity, but they are required for all instances if using linked containers, and the `${REPOSITORY}` will be referenced as `sentry-onpremise`.
{%- endcapture -%}
{%- include components/alert.html
  title="Note"
  content=__alert_content
%}

### Running Migrations

```bash
docker run --rm -it sentry-onpremise upgrade
```

During the upgrade, you will be prompted to create the initial user which will act as the superuser.

All schema changes and database upgrades are handled via the `upgrade` command, and this is the first thing you’ll want to run when upgrading to future versions of Sentry.

### Starting the Web Service

The web interface needs to expose port 9000 into the container. This can just be done with _–publish 9000:9000_:

```bash
docker run \
  --detach \
  --name sentry-web-01 \
  --publish 9000:9000 \
  sentry-onpremise \
  run web
```

You should now be able to test the web service by visiting `http://localhost:9000/`.

### Starting Background Workers

A large amount of Sentry’s work is managed via background workers:

```bash
docker run \
  --detach \
  --name sentry-worker-01 \
  sentry-onpremise \
  run worker
```

See [_Asynchronous Workers_]({%- link _documentation/server/queue.md -%}) for more details on configuring workers.

### Starting the Cron Process

Sentry also needs a cron process:

```bash
docker run \
  --detach \
  --name sentry-cron \
  sentry-onpremise \
  run cron
```

It’s recommended to only run one cron container at a time or you will see unnecessary extra tasks being pushed onto the queues, but the system will still behave as intended if multiple beat processes are run. This can be used to achieve high availability.

## What’s Next? {#what-s-next}

At this point, you should have a fully functional installation of Sentry. You may want to explore [_various plugins_]({%- link _documentation/server/plugins.md -%}) available.
