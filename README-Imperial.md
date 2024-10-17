# [Beta] Invenio Helm chart: Imperial Fair Data Repository instructions

The deployment takes about 2-3 minutes. We've added an init container to the install-init
job in order to wait for OpenSearch to become available. This in turn has required an
increase in the initial delay of the startup probes for the worker and worker-beat pods.

Kubectl has been upgraded to v. 1.31.

# Setup
In order to have a ReadWriteMany shared volume, a StorageClass has been
[created](https://learn.microsoft.com/en-us/azure/aks/azure-csi-files-storage-provision#create-a-storage-class) at `templates/azure-file-sc.yaml`.

The application routing add-on was [enabled](https://learn.microsoft.com/en-us/azure/aks/app-routing#enable-on-an-existing-cluster)
and the `ingressClassName` was given the correct value for Azure from that documentation.

[Set up a custom domain name and SSL certificate with the application routing add-on](https://learn.microsoft.com/en-us/azure/aks/app-routing-dns-ssl)


# Deployment commands
To install, cd into `charts/invenio` and run:

Note that you can add the argument `--debug` for all helm commands.

If the namespace invenio doesn't exist you need to add the argument `--create-namespace`.
You can do a dry run of a helm command by adding, `--dry-run`.

```bash
helm install -f values-overrides-imperial.yaml -n invenio fair-data-repository-dev .
```

If you want to apply a change of configuration you can upgrade like so:
```bash
helm upgrade -f values-overrides-imperial.yaml --debug data-repository-dev .
```

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