# [Beta] Invenio Helm Chart v. 0.2.1

This repository contains the helm chart to deploy an Invenio instance.

:warning: Please note that this configuration is not meant to be used in production.
This configuration should be adapted and hardened depending on your infrastructure and
constraints.

1. [Pre-requisites](#pre-requisites)
2. [Configuration](#configuration)
3. [Secret management](#secret-management)
4. [Deploy your instance](#deploy-your-instance)

## Pre-requisites

- [Helm, version 3.x](https://helm.sh/docs/intro/install/)

Depending on the underlying technology, pre-requisites and configuration might
change.

- [Kubernetes](README-Kubernetes.md)
- [OpenShift](README-OpenShift.md)

## Dependencies
This Helm chart uses Bitnami charts as [dependencies](https://helm.sh/docs/chart_best_practices/dependencies/)
for the following exact pinned versions:
* Opensearch 1.0.0 ([values.yaml](https://github.com/bitnami/charts/blob/opensearch/1.0.0/bitnami/opensearch/values.yaml))
* PostgreSQL 14.0.1 ([values.yaml](https://github.com/bitnami/charts/blob/postgresql/14.0.1/bitnami/postgresql/values.yaml))
* RabbitMQ 12.9.3 ([values.yaml](https://github.com/bitnami/charts/blob/rabbitmq/12.9.3/bitnami/rabbitmq/values.yaml))
* Redis 18.12.0 ([values.yaml](https://github.com/bitnami/charts/blob/redis/18.12.0/bitnami/redis/values.yaml))

Each one of them has a persistent volume claim for 8gb by default. Note that by default Redis will spin up 3 replicas.

## Configuration

:warning: Before installing you need to configure two things in your
`values.yaml` file.

- Host
- The web/worker docker images. If you need credentials you can see how to set
  them up in [Kubernetes](README-Kubernetes/#docker-credentials).

```yaml
host: yourhost.localhost

web:
  image: your/invenio-image

worker:
  image: your/invenio-image
```

## Secret management

It is recommended to configure the following variables. It can be done in the
`values-overrides.yaml` file.

```yaml

invenio:
    init: true  # initiates db, index, and admin roles
    demo_data: true  # for a demo set of records
    default_users: # for creating users on install
        "user@example.com": "password"
    secret_key: "my-very-safe-secret"

rabbitmq:
    auth:
        password: "mq_password"

postgresql:
    auth:
        password: "db_password"

```

It's however **strongly advised** to override them either through a value file
or through the `--set` flag, especially if running anything else than a
private test environment. If using OpenShift, you can use
[Secrets](README-OpenShift.md/#secret-management).

You can see an example of the `--set` option. Multiple values and/or `--set`
flags can be used in the same command.

```bash
DB_PASSWORD=$(openssl rand -hex 8)
helm install -f safe-values.yaml \
  --set rabbitmq.auth.password=$RABBITMQ_PASSWORD \
  --set postgresql.auth.password=$DB_PASSWORD \
  invenio ./invenio-k8s --namespace invenio
```

## Deploy your instance

To deploy your instance you have to options, directly from GitHub or from your local
clone.

### Adding a helm repository

``` console
$ helm repo add helm-invenio https://inveniosoftware.github.io/helm-invenio/
$ helm repo update
$ helm search invenio

NAME                   	CHART VERSION	APP VERSION	DESCRIPTION
helm-invenio/invenio	0.2.0        	1.16.0     	Open Source framework for large-scale digital repositories
helm-invenio/invenio	0.1.0        	1.16.0     	Open Source framework for large-scale digital repositories
```

Install the desired version

``` console
$ helm install invenio helm-invenio/invenio --version 0.2.0
```

### Cloning the GitHub repository

First, clone the GitHub repository and change directory:

```bash
git clone https://github.com/inveniosoftware/helm-invenio.git
cd helm-invenio/
```

If using kubernetes, you might need to add `--namespace invenio` to the
install command.

Then proceed to the installation

```bash
$ helm install [-f values-overrides.yaml] invenio ./invenio
# NAME: invenio
# LAST DEPLOYED: Mon Mar  9 16:25:15 2020
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
# NOTES:Invenio is ready to rock :rocket:
```

If for some reason you need to update parameters you can simply edit them in
the `values-overrides.yaml` file and use the `upgrade` command:

```bash
$ helm upgrade --atomic -f values-overrides.yaml invenio ./invenio
# Release "invenio" has been upgraded. Happy Helming!
# NAME: invenio
# LAST DEPLOYED: Tue Dec  7 15:29:08 2021
# NAMESPACE: default
# STATUS: deployed
# REVISION: 2
# TEST SUITE: None
# NOTES:
# Invenio is ready to rock 🚀
```
