# [Beta] Invenio Helm chart: Imperial Fair Data Repository instructions

**Currently for DEV / CI - using letsencrypt certificates on nginx ingress**

The deployment takes about 2-3 minutes. We've added an init container to the install-init
job in order to wait for OpenSearch to become available. This in turn has required an
increase in the initial delay of the startup probes for the worker and worker-beat pods.

# Setup

Tools required are `helm`, the Azure CLI tool [az](https://go.microsoft.com/fwlink/?linkid=872496) and [Kubectl](https://go.microsoft.com/fwlink/?linkid=2233742).

## Connect to the cluster
Hint: Commands are pre-populated when you view the instructions in the `connect` link in [Azure Portal](https://portal.azure.com)

```bash
az login
az account set --subscription <subscription_uuid>
az aks get-credentials --resource-group invenio-dev --name invenio-dev --overwrite-existing
```

Your account will need role `Azure Kubernetes Service RBAC Cluster Admin` for the above to suceed.

In the command above, `aks` refers to the Azure Kubernetes Service, it's editing your local `.kube/config` file to add the context for cluster access via `kubectl`.

## Enable Azure features

### Storage
In order to have a ReadWriteMany shared volume, a StorageClass has been
[created](https://learn.microsoft.com/en-us/azure/aks/azure-csi-files-storage-provision#create-a-storage-class) at `templates/azure-file-sc.yaml`.

To use this, the Storage feature needs to be enabled on the subscription, via:
```bash
az provider register --namespace Microsoft.Storage
```

### Approuting
The application routing add-on must be [enabled](https://learn.microsoft.com/en-us/azure/aks/app-routing#enable-on-an-existing-cluster) 
to correspond with the `ingressClassName` for Azure from that documentation, i.e. `class: "webapprouting.kubernetes.azure.com"`

```bash
az aks approuting enable --resource-group invenio-dev --name invenio-dev
```

### Azure DNS

**FIXME:** for the timebeing, we are _manually via GUI_ annotating the Public IP created in the above step with a DNS name, which gives us an Azure FQDN, e.g. `imperial-invenio-dev.uksouth.cloudapp.azure.com`
The same `hostname` must be used in `values-overrides-imperial.yml` to access the application.

## Install CertManager on the cluster

CertManager is used to automatically handle SSL certificates outwith the service itself. It also gives us a simple way to verify for LetsEncrypt certs.
For the full process followed here, see the documentation at https://cert-manager.io/docs/tutorials/getting-started-aks-letsencrypt/

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

This installation occurs on the same cluster but in a different namespace to the InvenioRDM application.

# Deployment commands
To install, cd into `charts/invenio`, first run

```bash
helm dependency build
```
To get the charts for opensearch, postgresql, rabbitmq, and redis.

Note that you can add the argument `--debug` for all helm commands.

If the namespace invenio doesn't exist you need to add the argument `--create-namespace`.
You can do a dry run of a helm command by adding, `--dry-run`.

Some of the secret values in `values-overrides-imperial.yaml` have been obscured with the text `REPLACE-ME` and not checked into the repo.
These need to be supplied at runtime (so that they can be templated via GitHub Secrets for CI). Unfortunately this makes the following setup
commands rather lengthy. Feel free to use environment variables to ease the process.

```bash
helm install -f values-overrides-imperial.yaml -n invenio fair-data-repository-dev . [--create-namespace] \
  --set invenio.secret_key=<your_key> \
  --set invenio.security_login_salt=<your_login_salt> \
  --set invenio.csrf_secret_salt=<another_secret_salt> \
  --set invenio.extra_config.ICL_OAUTH_CLIENT_ID=<id_provided> \
  --set invenio.extra_config.ICL_OAUTH_CLIENT_SECRET=<key_provided> \
  --set invenio.extra_config.ICL_OAUTH_WELL_KNOWN_URL=<url_provided> \
  --set rabbitmq.auth.password=<your_mq_password> \
  --set postgresql.auth.password=<your_pg_password>
```
`fair-data-repository-dev` is the installation name in `helm`.

If you want to apply a change of configuration you can upgrade like so:
```bash
helm upgrade -f values-overrides-imperial.yaml -n invenio fair-data-repository-dev . \
  --set invenio.secret_key=<your_key> \
  --set invenio.security_login_salt=<your_login_salt> \
  --set invenio.csrf_secret_salt=<another_secret_salt> \
  --set invenio.extra_config.ICL_OAUTH_CLIENT_ID=<id_provided> \
  --set invenio.extra_config.ICL_OAUTH_CLIENT_SECRET=<key_provided> \
  --set invenio.extra_config.ICL_OAUTH_WELL_KNOWN_URL=<url_provided> \
  --set rabbitmq.auth.password=<your_mq_password> \
  --set postgresql.auth.password=<your_pg_password>
```

You **MUST** provide the same secrets as before if you wish for the existing data in the instance to be accessible

If you want to uninstall you can run:
```bash
helm uninstall fair-data-repository-dev -n invenio
```

Uninstalling a helm installation does not remove the 11 persistent volumes (PVs) and claims (PVCs) created. To see the
pvcs run

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

To scale web or workers, you can run:
```bash
kubectl scale deployment --replicas=5 web -n invenio
```