# [Beta] Invenio Helm chart: Imperial Fair Data Repository instructions

**Currently for DEV / CI - using letsencrypt certificates on nginx ingress**

The deployment takes about 2-3 minutes. We've added an init container to the
install-init job in order to wait for OpenSearch to become available. This in
turn has required an increase in the initial delay of the startup probes for
the worker and worker-beat pods. The standard deployment has 24 pods.


# Components

| Component      | Description                        | Pods/containers                                       |
|----------------|------------------------------------|-------------------------------------------------------| 
| InvenioRDM     | Python/Flask app for research data | 6 pods; each pod has 2 containers: web and nginx      |
| Init job       | One off init job                   | 1 pod                                                 |
| PostgreSQL     | Main relational database           | * should be 1 pod but will be external                |
| Elasticsearch  | Search indexing backend            | 8 pods; 2 master, 2 data, 2 ingest and 2 coordinating |
| Redis          | Cache and Celery broker            | 4 pods; 1 master and 3 replicas                       |                       
| Celery workers | Background task execution          | 3 pods, 2 worker and 1 beat                           |
| RabbitMQ       | Message queue for Celery tasks     | 1 pod                                                 |
| Kubernetes     | Orchestration and scaling platform |
| Helm           | Deployment and configuration tool  |
| Nginx          | HTTP server and reverse proxy      | * this is part of the InvenioRDM deployment           |

# Setup

Tools required are `helm`, the Azure CLI tool [az](https://go.microsoft.com/fwlink/?linkid=872496)
and [Kubectl](https://go.microsoft.com/fwlink/?linkid=2233742).

## Connect to the cluster

Hint: Commands are pre-populated when you view the instructions in the `connect`
link in [Azure Portal](https://portal.azure.com)

```bash
az login
az account set --subscription <subscription_uuid>
az aks get-credentials --resource-group <resource_group> --name invenio-dev --overwrite-existing
```

Your account will need role `Azure Kubernetes Service RBAC Cluster Admin` for
the above to succeed.

In the command above, `aks` refers to the Azure Kubernetes Service, it's editing
your local `.kube/config` file to add the context for cluster access via `kubectl`.

The choice of `resource group` seems to have implications for the storage account
and other resources because resource groups have geographical locations. The current 
resource groups location is `uksouth` also available is `ukwest`. To list resource
groups run:

```bash
az group list
```

## Enable Azure features

### Storage

In order to have a ReadWriteMany shared volume, a StorageClass has been
[created](https://learn.microsoft.com/en-us/azure/aks/azure-csi-files-storage-provision#create-a-storage-class) at
`templates/azure-file-sc.yaml`.

To use this, the Storage feature needs to be enabled on the subscription, via:

```bash
az provider register --namespace Microsoft.Storage
```

[Azure Storage redundancy SKUs (Stock Keeping Unit)](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
offer different levels of availability and backup strategies: 
* Standard_LRS: Standard locally redundant storage (LRS)
* Standard_GRS: Standard geo-redundant storage (GRS)
* Standard_ZRS: Standard zone redundant storage (ZRS)
* Standard_RAGRS: Standard read-access geo-redundant storage (RA-GRS)
* Premium_LRS: Premium locally redundant storage (LRS)
* Premium_ZRS: Premium zone redundant storage (ZRS)

The storage class unit `Standard_LRS` is used in the `azure-file-sc.yaml` file,
but this can be changed to suit your needs.

Storage accounts are linked to resource groups, to create a new storage account
see [this guide](https://learn.microsoft.com/en-gb/azure/storage/common/storage-account-create).
To list storage accounts run:

```bash
az storage account list
```

The various pods will access a shared volume via the `azure-file-sc` storage class
called `shared volume` (see persistence.name in values.yaml) in the `invenio` 
namespace. To view information about the persistent volume claim (PVC) run:

```bash
kubectl describe pvc -n invenio shared-volume
```

### Approuting

The application routing add-on must be [enabled](https://learn.microsoft.com/en-us/azure/aks/app-routing#enable-on-an-existing-cluster)
to correspond with the `ingressClassName` for Azure from that documentation, i.e.
`class: "webapprouting.kubernetes.azure.com"`

```bash
az aks approuting enable --resource-group invenio-dev --name invenio-dev
```

### Azure DNS

**FIXME:** for the time being, we are _manually via GUI_ annotating the Public IP 
created in the above step with a DNS name, which gives us an Azure FQDN, e.g.
`icl-invenio-dev.uksouth.cloudapp.azure.com` The same `hostname` must be used in 
`values-overrides-imperial.yml` to access the application.

## Install CertManager on the cluster

CertManager is used to automatically handle SSL certificates outwith the service 
itself. It also gives us a simple way to verify for LetsEncrypt certs. For the 
full process followed here, see the documentation at 
https://cert-manager.io/docs/tutorials/getting-started-aks-letsencrypt/

The necessary steps on a fresh cluster are:

## [Install cert-manager](https://cert-manager.io/docs/tutorials/getting-started-aks-letsencrypt/#install-cert-manager)

```bash
helm repo add jetstack https://charts.jetstack.io --force-update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.1 \
  --set crds.enabled=true
```

This installation occurs on the same cluster but in a different namespace to the 
InvenioRDM application.

### [Teardown Cert-Manager](https://cert-manager.io/v1.2-docs/installation/uninstall/kubernetes/)

```shell
helm --namespace cert-manager delete cert-manager
kubectl delete namespace cert-manager
```

# InvenioRDM Deployment commands

With Azure features enabled, you can now deploy the InvenioRDM application.

To install, cd into `charts/invenio`, first run

```bash
helm dependency build
```

To get the charts for `opensearch`, `postgresql`, `rabbitmq`, and `redis`.

Note that you can add the argument `--debug` to all `helm` commands for a 
bit more verbosity.

If the namespace invenio doesn't exist you need to add the argument 
`--create-namespace`. You can do a dry run of a `helm` command by adding,
`--dry-run`.

Some of the secret values in `values-overrides-imperial.yaml` have been 
obscured with the text `REPLACE-ME` and not checked into the repo.

These need to be supplied at runtime (so that they can be templated via 
GitHub Secrets for CI). Unfortunately this makes the following commands 
rather lengthy. Feel free to use environment variables to ease the process
(see below).

Note: These commands are for demonstration purposes only, and do not reflect the
number of arguments that should be passed in. See the Github action workflows
in the [repository](https://github.com/ImperialCollegeLondon/fair-data-repository/tree/develop/.github/workflows).

```bash
helm install -f values-overrides-imperial.yaml -n invenio fair-data-repository-dev . [--create-namespace] \
  --set invenio.secret_key=<your_key> \
  --set invenio.security_login_salt=<your_login_salt> \
  --set invenio.csrf_secret_salt=<another_secret_salt> \
  --set invenio.extraConfig.ICL_OAUTH_CLIENT_ID=<id_provided> \
  --set invenio.extraConfig.ICL_OAUTH_CLIENT_SECRET=<key_provided> \
  --set invenio.extraConfig.ICL_OAUTH_WELL_KNOWN_URL=<url_provided> \
  --set rabbitmq.auth.password=<your_mq_password> \
  --set postgresql.auth.password=<your_pg_password>
```

`fair-data-repository-dev` is the deployment name in `helm`, which you'll use
for subsequent management commands.

If you want to apply a change of configuration you can upgrade like so:

```bash
helm upgrade -f values-overrides-imperial.yaml -n invenio fair-data-repository-dev . \
  --set invenio.secret_key=<your_key> \
  --set invenio.security_login_salt=<your_login_salt> \
  --set invenio.csrf_secret_salt=<another_secret_salt> \
  --set invenio.extraConfig.ICL_OAUTH_CLIENT_ID=<id_provided> \
  --set invenio.extraConfig.ICL_OAUTH_CLIENT_SECRET=<key_provided> \
  --set invenio.extraConfig.ICL_OAUTH_WELL_KNOWN_URL=<url_provided> \
  --set rabbitmq.auth.password=<your_mq_password> \
  --set postgresql.auth.password=<your_pg_password>
```

You **MUST** provide the same secrets as before if you wish for the existing
data in the instance to be accessible.

## Secrets generation and env templating

Useful for generating these: `uuidgen`, `pwgen -N 1` for UUIDs and a 
single simple password, respectively.

```shell
# A secret local shell script to create our app secrets
# icl_secrets.sh
export ICL_INVENIO_SECRET_KEY=`uuidgen`
export ICL_INVENIO_SECURITY_LOGIN_SALT=`uuidgen`
export ICL_INVENIO_CSRF_SECRET_SALT=`uuidgen`
export ICL_INVENIO_OAUTH_CLIENT_ID=<provided>
export ICL_INVENIO_OAUTH_CLIENT_SECRET=<provided>
export ICL_INVENIO_OAUTH_WELL_KNOWN_URL=<provided>
export ICL_INVENIO_RABBITMQ_PW=`pwgen -N 1 27`
export ICL_INVENIO_POSTGRES_PW=`pwgen -N 1 17`
```

Then using the install command:

```shell
helm install -f values-overrides-imperial.yaml -n invenio fair-data-repository-dev . --create-namespace \
  --set invenio.secret_key=$ICL_INVENIO_SECRET_KEY \
  --set invenio.security_login_salt=$ICL_INVENIO_SECURITY_LOGIN_SALT \
  --set invenio.csrf_secret_salt=$ICL_INVENIO_CSRF_SECRET_SALT \
  --set invenio.extraConfig.ICL_OAUTH_CLIENT_ID=$ICL_INVENIO_OAUTH_CLIENT_ID \
  --set invenio.extraConfig.ICL_OAUTH_CLIENT_SECRET=$ICL_INVENIO_OAUTH_CLIENT_SECRET \
  --set invenio.extraConfig.ICL_OAUTH_WELL_KNOWN_URL=$ICL_INVENIO_OAUTH_WELL_KNOWN_URL \
  --set rabbitmq.auth.password=$ICL_INVENIO_RABBITMQ_PW \
  --set postgresql.auth.password=$ICL_INVENIO_POSTGRES_PW
```

Or the upgrade command:

```shell
helm upgrade -f values-overrides-imperial.yaml -n invenio fair-data-repository-dev . \
  --set invenio.secret_key=$ICL_INVENIO_SECRET_KEY \
  --set invenio.security_login_salt=$ICL_INVENIO_SECURITY_LOGIN_SALT \
  --set invenio.csrf_secret_salt=$ICL_INVENIO_CSRF_SECRET_SALT \
  --set invenio.extraConfig.ICL_OAUTH_CLIENT_ID=$ICL_INVENIO_OAUTH_CLIENT_ID \
  --set invenio.extraConfig.ICL_OAUTH_CLIENT_SECRET=$ICL_INVENIO_OAUTH_CLIENT_SECRET \
  --set invenio.extraConfig.ICL_OAUTH_WELL_KNOWN_URL=$ICL_INVENIO_OAUTH_WELL_KNOWN_URL \
  --set rabbitmq.auth.password=$ICL_INVENIO_RABBITMQ_PW \
  --set postgresql.auth.password=$ICL_INVENIO_POSTGRES_PW
```

## Checking on installation progress

```bash
kubectl get pods --namespace invenio
```

Check the `STATUS` and `READY` columns - if they are stuck in a `Pending` 
state your cluster may not have enough resources.

To track the installation in real-time, it's helpful to use the `watch` 
command:

```bash
$ watch -n 10 kubectl get pods -n invenio

Every 10.0s: kubectl get pods -n invenio
(ctrl-c to exit)
```

Note that the _bitnami_ charts for OpenSearch and Redis are using their 
default configs, which includes replicas. These could be pared down at 
the expense of some redundancy.

The `install-init` container is temporary, i.e. it runs its job and then 
shuts down.

## Inspect a pod

To inspect a pod, you can run:

```bash
kubectl describe -n invenio web-57c8476cf8-2kvvt 
```

This command is compatible with `docker inspect` and will tell you the 
status of the pod. This is especially useful if something has gone wrong
with the pod, and it is stuck in a `CrashLoopBackOff` cycle. `CrashLoopBackOff`
means that the pod is crashing and restarting.

## Observe the web logs

To watch the logs from e.g. all web containers (running the Invenio app):

```bash
kubectl logs -f -l app=web -n invenio --max-log-requests=6
```

## Teardown

If you want to uninstall you can run:

```bash
helm uninstall fair-data-repository-dev -n invenio
```

Verify they're all destroyed with `kubectl get pods -n invenio`

If you're redeploying and the `install-init` pod keeps reappearing, that 
means it's being recreated by the job. You can remove that explicitly 
via:

```bash
kubectl delete job install-init -n invenio
```

Uninstalling a helm installation does not remove the 11 persistent 
volumes (PVs) and claims (PVCs) created. To see the pvcs run:

```bash
kubectl get pv -n invenio
```

and

```bash
kubectl get pvc -n invenio
```

You can delete all pvcs with this command:

```bash
kubectl delete pvc -n invenio --all
```

## Scale

To scale web or workers, you can run:

```bash
kubectl scale deployment --replicas=5 web -n invenio
```

# Administration - Run commands in pods

Observe which pods we have running

```bash
kubectl get pods -n invenio
```

Choose a `web` pod (running the application) and execute a shell

```bash
kubectl -n invenio exec --stdin --tty web-57c8476cf8-2kvvt -- /bin/bash
```

The `invenio` CLI command is available in the `web` pods, so from
the default entrypoint you can manage the application.

```bash
[invenio@web-57c8476cf8-2kvvt src]$ invenio --help
Usage: invenio [OPTIONS] COMMAND [ARGS]...

  Command Line Interface for Invenio.

Options:
  -e, --env-file FILE   Load environment variables from this file. python-
                        dotenv must be installed.
  -A, --app IMPORT      The Flask application or factory function to load, in
                        the form 'module:name'. Module can be a dotted import
                        or file path. Name is not required if it is 'app',
                        'application', 'create_app', or 'make_app', and can be
                        'name(args)' to pass arguments.
  --debug / --no-debug  Set debug mode.
  --version             Show the Flask version.
  --help                Show this message and exit.

Commands:
  access          Account commands.
  alembic         Perform database migrations.
  collect         Collect static files.
  communities     Invenio communities commands.
  db              Database commands.
  domains         Domain commands.
  files           File management commands.
  identity-cache  Invenio identity cache commands.
  index           Manage search indices.
  instance        Instance commands.
  limiter         Flask-Limiter maintenance & utility commmands
  pid             PID-Store management commands.
  queues          Manage events queue.
  rdm             Invenio app rdm commands.
  rdm-records     InvenioRDM records commands.
  roles           Role commands.
  routes          Show the routes for the app.
  run             Run a development server.
  shell           Runs a shell in the app context.
  stats           Statistics commands.
  tokens          OAuth2 server token commands.
  users           User commands.
  vocabularies    Vocabularies command.
  webpack         Webpack commands
```

## Search indices

InvenioRDM uses the [Bitnami OpenSearch charts](https://artifacthub.io/packages/helm/bitnami/opensearch).
The indices are created during the init job. They are not stored in 
the shared volume, but in separate persistent volumes. There are 
persistent volumes for two master nodes and two data nodes, each 
having 8gb.

In order to rebuild indices you can run:

```bash
kubectl -n invenio exec <name of a web or worker pod> -- invenio rdm-records rebuild-index
```

This is a short-hand command for rebuilding the indices. You can 
also run the `invenio index` commands for manual index management.

See more on [InvenioRDM docs on how to back up the search indices](https://inveniordm.docs.cern.ch/develop/howtos/backup_search_indices/).


## Dev - run data import script

This is just the demo data we have included in our instance repo. 
Documentation can be found at
https://github.com/ImperialCollegeLondon/fair-data-repository/tree/develop/test_data

```bash
cd test_data
python download_test_data.py
python create_test_data_records.py
```