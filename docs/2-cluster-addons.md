# Cluster Addons Delivery

To manage the cluster add-ons, we'll define a set of Kustomize overlays for each
environment (staging and production).

A cluster addon is defined in a directory with the following structure:

```sh
cluster-addon/
├── controllers # CRD definitions and Kubernetes controllers
│   ├── base # common definitions (Namespaces, RBAC, HelmRepositories, HelmReleases)
│   ├── production # production specific HelmRelease values
│   └── staging # staging specific HelmRelease values
└── configs # Custom Resources of controllers
    ├── base # common definitions
    ├── production # production specific values
    └── staging # staging specific values
```

The CRDs and their controllers are reconciled before the custom resources to ensure that the
controllers are ready to process the custom resources.

## Cert-Manager Configuration

To demonstrate how to configure a cluster add-on, we'll use cert-manager as an example.

Create a directory for the cluster add-ons in the `flux-fleet` repository root:

```sh
mkdir -p cluster-addons/cert-manager
```

Generate the cert-manager configuration with:

```sh
flux pull artifact oci://ghcr.io/controlplaneio-fluxcd/d2-infra/cert-manager \
--output ./cluster-addons/cert-manager
```

## Cluster Addons Tenant Configuration

Create a directory for the tenants in the `flux-fleet` repository root:

```sh
mkdir -p tenants
```

Create a `ResourceSet` in `tenants/cluster-addons.yaml` with the following content:

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSet
metadata:
  name: cluster-addons
  namespace: flux-system
  annotations:
    fluxcd.controlplane.io/reconcileEvery: "5m"
spec:
  inputs:
    - tenant: "cert-manager"
      environment: "${ENVIRONMENT}"
  resources:
    - apiVersion: v1
      kind: Namespace
      metadata:
        name: << inputs.tenant >>
    - apiVersion: v1
      kind: ConfigMap
      metadata:
        name: flux-runtime-info
        namespace: << inputs.tenant >>
        annotations:
          fluxcd.controlplane.io/copyFrom: "flux-system/flux-runtime-info"
    - apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: flux
        namespace: << inputs.tenant >>
    - apiVersion: rbac.authorization.k8s.io/v1
      kind: ClusterRoleBinding
      metadata:
        name: flux-infra-<< inputs.tenant >>
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin
      subjects:
        - kind: ServiceAccount
          name: flux
          namespace: << inputs.tenant >>
    - apiVersion: kustomize.toolkit.fluxcd.io/v1
      kind: Kustomization
      metadata:
        name: << inputs.tenant >>-controllers
        namespace: flux-system
      spec:
        targetNamespace: << inputs.tenant >>
        serviceAccountName: flux-operator
        interval: 30m
        retryInterval: 5m
        wait: true
        timeout: 5m
        sourceRef:
          kind: GitRepository
          name: flux-system
        path: "./cluster-addons/<< inputs.tenant >>/controllers/<< inputs.environment >>"
        prune: true
        postBuild:
          substituteFrom:
            - kind: ConfigMap
              name: flux-runtime-info
    - apiVersion: kustomize.toolkit.fluxcd.io/v1
      kind: Kustomization
      metadata:
        name: << inputs.tenant >>-configs
        namespace: flux-system
      spec:
        dependsOn:
          - name: << inputs.tenant >>-controllers
        targetNamespace: << inputs.tenant >>
        serviceAccountName: flux-operator
        interval: 30m
        retryInterval: 5m
        wait: true
        timeout: 5m
        sourceRef:
          kind: GitRepository
          name: flux-system
        path: "./cluster-addons/<< inputs.tenant >>/configs/<< inputs.environment >>"
        prune: true
        postBuild:
          substituteFrom:
            - kind: ConfigMap
              name: flux-runtime-info
```

Create a Flux Kustomization in `clusters/staging/tenants.yaml` with the following content:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: tenants
  namespace: flux-system
spec:
  serviceAccountName: flux-operator
  interval: 12h
  retryInterval: 3m
  path: ./tenants
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: flux-runtime-info
```

Commit and push the changes to the `flux-fleet` repository:

```sh
git add -A
git commit -m "Add custer-addons"
git push origin main
```

Wait for Flux to detect the change in Git or ask it to apply changes immediately:

```sh
flux reconcile kustomization flux-system --with-source
```

Verify that the `ResourceSet` is applied and the cluster addons are deployed:

```sh
watch flux get all -A
```
