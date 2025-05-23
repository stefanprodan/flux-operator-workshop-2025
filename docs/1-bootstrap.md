# Flux Operator Bootstrap

The bootstrap procedure is a one-time operation that installs the
[Flux Operator](https://github.com/controlplaneio-fluxcd/flux-operator) on the cluster,
configures the Flux controllers and the delivery of platform components and applications.

After bootstrap, changes to the Flux configuration and version upgrades are done by
modifying the `FluxInstance` manifest and letting Flux reconcile the changes,
there is no need to run bootstrap again nor connect to the cluster.

## Cluster Installation

Create a local Kubernetes cluster called `flux-staging` using KinD:

```sh
kind create cluster --name flux-staging --wait 5m
```

Verify that the cluster is compatible with Flux:

```sh
flux check --pre
```

## Flux Operator Installation

Install Flux Operator with the Helm CLI in the `flux-system` namespace:

```sh
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace \
  --wait
```

## Flux Instance Configuration

Clone the `flux-fleet` repository locally:

```sh
git clone https//github.com/<your-username>/flux-fleet.git
cd flux-fleet
```

Create the directory structure for the staging cluster:

```sh
mkdir -p clusters/staging/flux-system
```

Create a `FluxInstance` in `clusters/staging/flux-system/flux-instance.yaml` with the following content:

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: FluxInstance
metadata:
  name: flux
  namespace: flux-system
spec:
  distribution:
    version: "2.x"
    registry: "ghcr.io/fluxcd"
    artifact: "oci://ghcr.io/controlplaneio-fluxcd/flux-operator-manifests:latest"
  components:
    - source-controller
    - kustomize-controller
    - helm-controller
    - notification-controller
  cluster:
    type: kubernetes
    multitenant: true
    tenantDefaultServiceAccount: flux
    networkPolicy: true
    domain: "cluster.local"
  sync:
    kind: GitRepository
    url: "https://github.com/stefanprodan/flux-fleet"
    ref: "refs/heads/main"
    path: "clusters/staging"
    pullSecret: "github-auth"
  kustomize:
    patches:
      - target:
          kind: Deployment
          name: "(kustomize-controller|helm-controller)"
        patch: |
          - op: add
            path: /spec/template/spec/containers/0/args/-
            value: --concurrent=10
          - op: add
            path: /spec/template/spec/containers/0/args/-
            value: --requeue-dependency=5s
```

Make sure to replace the `url` field with your own GitHub repository URL.

Commit and push the changes to the `flux-fleet` repository:

```sh
git add -A
git commit -m "Add Flux Instance"
git push origin main
```

## Flux Instance Installation

Create a Kubernetes secret with the GitHub PAT named `github-auth` in the `flux-system` namespace:

```sh
flux create secret git github-auth \
  --url=https://github.com/stefanprodan/flux-fleet \
  --username=flux \
  --password=$GITHUB_TOKEN
```

Make sure the `GITHUB_TOKEN` environment variable is set to your GitHub PAT.

Apply the `FluxInstance` manifest to the cluster:

```sh
kubectl apply -f clusters/staging/flux-system/flux-instance.yaml
```

Wait for the Flux Operator to reconcile the `FluxInstance` and start the Flux controllers:

```sh
kubectl -n flux-system get pods -w
```

Verify the Flux installation by inspecting the `FluxReport`:

```sh
kubectl -n flux-system get fluxreport flux -o yaml
```

> [!NOTE]
> From now on, we can manage the Kubernetes cluster solely through GitOps.
> Any changes to the cluster should be done by creating, modifying or deleting Kubernetes
> manifests in the Git repository.

## Cluster Info Configuration

To identify the cluster where Flux is running, we are going to create a `ConfigMap`
with the details about the cluster such as the environment, cluster name and domain.

Create a `ConfigMap` in `clusters/staging/flux-system/flux-runtime-info.yaml` with the following content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flux-runtime-info
  namespace: flux-system
  labels:
    toolkit.fluxcd.io/runtime: "true"
data:
  ENVIRONMENT: staging
  CLUSTER_NAME: kind-flux-staging
  CLUSTER_DOMAIN: staging.example.com
```

Commit and push the changes to the `flux-fleet` repository:

```sh
git add -A
git commit -m "Add Flux runtime info"
git push origin main
```

## Automated Upgrades Configuration

Create a `ResourceSet` in `clusters/staging/flux-system/flux-operator.yaml` with the following content:

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSet
metadata:
  name: flux-operator
  namespace: flux-system
spec:
  dependsOn:
    - apiVersion: apiextensions.k8s.io/v1
      kind: CustomResourceDefinition
      name: helmreleases.helm.toolkit.fluxcd.io
  resources:
    - apiVersion: source.toolkit.fluxcd.io/v1beta2
      kind: OCIRepository
      metadata:
        name: flux-operator
        namespace: flux-system
      spec:
        interval: 30m
        url: oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator
        ref:
          semver: '*'
        verify:
          provider: cosign
          matchOIDCIdentity:
          - issuer: ^https://token\.actions\.githubusercontent\.com$
            subject: ^https://github\.com/controlplaneio-fluxcd/charts/.*$
    - apiVersion: helm.toolkit.fluxcd.io/v2
      kind: HelmRelease
      metadata:
        name: flux-operator
        namespace: flux-system
      spec:
        interval: 30m
        releaseName: flux-operator
        serviceAccountName: flux-operator
        chartRef:
          kind: OCIRepository
          name: flux-operator
        values:
          multitenancy:
            enabled: true
            defaultServiceAccount: flux-operator
          reporting:
            interval: 45s
```

Commit and push the changes to the `flux-fleet` repository:

```sh
git add -A
git commit -m "Add Flux Operator ResourceSet"
git push origin main
```

Wait for Flux to detect the change in Git or ask it to apply changes immediately:

```sh
flux reconcile kustomization flux-system --with-source
```

Verify that the `ResourceSet` is applied and the `HelmRelease` is created:

```sh
flux get helmreleases
```

> [!NOTE]
> From now on, the Flux Operator will be automatically upgraded, 
> which in turn will upgrade the Flux controllers to the latest version.

## Commit Status Configuration

To know when a change in Git has been reconciled on the cluster, Flux can be configured
to update the status of the commit in GitHub.

Create a Flux `Alert` and `Provider` in `clusters/staging/flux-system/flux-notifications.yaml` with the following content:

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSet
metadata:
  name: flux-notifications
  namespace: flux-system
spec:
  dependsOn:
    - apiVersion: apiextensions.k8s.io/v1
      kind: CustomResourceDefinition
      name: alerts.notification.toolkit.fluxcd.io
  resources:
    - apiVersion: notification.toolkit.fluxcd.io/v1beta3
      kind: Provider
      metadata:
        name: github-status
        namespace: flux-system
      spec:
        type: github
        address: https://github.com/stefanprodan/flux-fleet
        secretRef:
          name: github-auth     
    - apiVersion: notification.toolkit.fluxcd.io/v1beta3
      kind: Alert
      metadata:
        name: github-status
        namespace: flux-system
      spec:
        providerRef:
          name: github-status
        eventSources:
          - kind: Kustomization
            name: flux-system
```

Commit and push the changes to the `flux-fleet` repository:

```sh
git add -A
git commit -m "Add Flux Notifications"
git push origin main
```

Wait for Flux to detect the change in Git or ask it to apply changes immediately:

```sh
flux reconcile kustomization flux-system --with-source
```

Now if we go to the GitHub repository, we should see a green check mark next to
the last commit with the message "kustomization/flux-system reconciliation succeeded".

> [!NOTE]
> From now on, Flux will keep updating the commit status in GitHub every time
> the Kustomization is reconciled, any errors will be reported as a red cross.
>
> Flux also supports sending notifications to other providers such as Slack, Discord, Teams, etc.
> When using messaging providers, the alert posted to the channel will contain
> details about which resources were created, deleted or modified and any errors that occurred.
