---
title: Access to the Garden Cluster for Extensions
---

# Access to the Garden Cluster for Extensions

Extensions that are installed on seed clusters via a `ControllerInstallation` can simply read the kubeconfig file specified by the `GARDEN_KUBECONFIG` environment variable to create a garden cluster client.
With this, they use a short-lived token (valid for `12h`) associated with a dedicated `ServiceAccount` in the `seed-<seed-name>` namespace to securely access the garden cluster.
The used `ServiceAccounts` are granted permissions in the garden cluster similar to gardenlet clients.

## Background

Historically, `gardenlet` has been the only component running in the seed cluster that has access to both the seed cluster and the garden cluster.
Accordingly, extensions running on the seed cluster didn't have access to the garden cluster.

Starting from Gardener [`v1.74.0`](https://github.com/gardener/gardener/releases/v1.74.0), there is a new mechanism for components running on seed clusters to get access to the garden cluster.
For this, `gardenlet` runs an instance of the [`TokenRequestor`](../concepts/gardenlet.md#tokenrequestor-controller) for requesting tokens that can be used to communicate with the garden cluster.

## Manually Requesting a Token for the Garden Cluster

Seed components that need to communicate with the garden cluster can request a token in the garden cluster by creating a garden access secret.
This secret has to be labelled with `resources.gardener.cloud/purpose=token-requestor` and `resources.gardener.cloud/class=garden`, e.g.:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: garden-access-example
  namespace: example
  labels:
    resources.gardener.cloud/purpose: token-requestor
    resources.gardener.cloud/class: garden
  annotations:
    serviceaccount.resources.gardener.cloud/name: example
type: Opaque
```

This will instruct gardenlet to create a new `ServiceAccount` named `example` in its own `seed-<seed-name>` namespace in the garden cluster, request a token for it, and populate the token in the secret's data under the `token` key.

## Using Gardenlet-Managed Garden Access

By default, extensions are equipped with secure access to the garden cluster using a dedicated `ServiceAccount` without requiring any additional action.
They can simply read the file specified by the `GARDEN_KUBECONFIG` and construct a garden client with it.

When installing a [`ControllerInstallation`](controllerregistration.md), gardenlet creates two secrets in the installation's namespace: a generic garden kubeconfig (`generic-garden-kubeconfig-<hash>`) and a garden access secret (`garden-access-extension`).
Additionally, it injects `volume`, `volumeMounts`, and two environment variables into all (init) containers in all objects in the `apps` and `batch` API groups:

- `GARDEN_KUBECONFIG`: points to the path where the generic garden kubeconfig is mounted.
- `SEED_NAME`: set to the name of the `Seed` where the extension is installed. 
  This is useful for restricting watches in the garden cluster to relevant objects.

If an object already contains the `GARDEN_KUBECONFIG` environment variable, it is not overwritten and injection of `volume` and `volumeMounts` is skipped.

For example, a `Deployment` deployed via a `ControllerInstallation` will be mutated as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gardener-extension-provider-local
  annotations:
    reference.resources.gardener.cloud/secret-795f7ca6: garden-access-extension
    reference.resources.gardener.cloud/secret-d5f5a834: generic-garden-kubeconfig-81fb3a88
spec:
  template:
    metadata:
      annotations:
        reference.resources.gardener.cloud/secret-795f7ca6: garden-access-extension
        reference.resources.gardener.cloud/secret-d5f5a834: generic-garden-kubeconfig-81fb3a88
    spec:
      containers:
      - name: gardener-extension-provider-local
        env:
        - name: GARDEN_KUBECONFIG
          value: /var/run/secrets/gardener.cloud/garden/generic-kubeconfig/kubeconfig
        - name: SEED_NAME
          value: local
        volumeMounts:
        - mountPath: /var/run/secrets/gardener.cloud/garden/generic-kubeconfig
          name: garden-kubeconfig
          readOnly: true
      volumes:
      - name: garden-kubeconfig
        projected:
          defaultMode: 420
          sources:
          - secret:
              items:
              - key: kubeconfig
                path: kubeconfig
              name: generic-garden-kubeconfig-81fb3a88
              optional: false
          - secret:
              items:
              - key: token
                path: token
              name: garden-access-extension
              optional: false
```

The generic garden kubeconfig will look like this:

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: LS0t...
    server: https://garden.local.gardener.cloud:6443
  name: garden
contexts:
- context:
    cluster: garden
    user: extension
  name: garden
current-context: garden
users:
- name: extension
  user:
    tokenFile: /var/run/secrets/gardener.cloud/garden/generic-kubeconfig/token
```

## Permissions in the Garden Cluster

Both the [`SeedAuthorizer` and the `SeedRestriction` plugin](../deployment/gardenlet_api_access.md) handle extensions clients and generally grant the same permissions in the garden cluster to them as to gardenlet clients.
With this, extensions are restricted to work with objects in the garden cluster that are related to seed they are running one just like gardenlet.
Note that if the plugins are not enabled, extension clients are only granted read access to global resources like `CloudProfiles` (this is granted to all authenticated users).
There are a few exceptions to the granted permissions as documented [here](../deployment/gardenlet_api_access.md#rule-exceptions-for-extension-clients).

If an extension needs access to additional resources in the garden cluster (e.g., extension-specific custom resources), permissions need to be granted via the usual RBAC means.
Note that this is done outside of Gardener and might require an additional controller that manages RBAC for extension clients in the garden cluster.

## Renewing All Garden Access Secrets

Operators can trigger an automatic renewal of all garden access secrets in a given `Seed` and their requested `ServiceAccount` tokens, e.g., when rotating the garden cluster's `ServiceAccount` signing key.
For this, the `Seed` has to be annotated with `gardener.cloud/operation=renew-garden-access-secrets`.
