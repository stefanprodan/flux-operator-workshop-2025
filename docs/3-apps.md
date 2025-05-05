# Applications Delivery

To manage the delivery of applications, we'll group the applications by namespace
and define a set of Kustomize overlays for each environment (staging and production).

The apps are defined in a directory with the following structure:

```sh
./apps/
└── namespace
    ├── base
    │   ├── kustomization.yaml
    │   └── release1.yaml
    │   └── release2.yaml
    ├── production
    │   ├── kustomization.yaml
    │   ├── release1-values.yaml
    │   └── release2-values.yaml
    └── staging
        ├── kustomization.yaml
        ├── release1-values.yaml
        └── release2-values.yaml
```

## Podinfo Configuration

To demonstrate how to configure a cluster add-on, we'll use podinfo as an example.

Create a directory for the frontend and backend apps in the `flux-fleet` repository root:

```sh
mkdir -p apps/frontend
mkdir -p apps/backend
```

Generate the podinfo configuration with:

```sh
flux pull artifact oci://ghcr.io/controlplaneio-fluxcd/d2-apps/frontend \
--output ./apps/frontend

yq -i 'del(.spec.values.serviceMonitor)' ./apps/frontend/base/podinfo.yaml

cp -r ./apps/frontend/ ./apps/backend/
```

## Apps Tenant Configuration

Create a `ResourceSet` in `tenants/apps.yaml` with the following content:

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSet
metadata:
  name: apps
  namespace: flux-system
  annotations:
    fluxcd.controlplane.io/reconcileEvery: "5m"
spec:
  inputs:
    - tenant: "frontend"
      environment: "${ENVIRONMENT}"
    - tenant: "backend"
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
      kind: RoleBinding
      metadata:
        name: flux
        namespace: << inputs.tenant >>
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
        name: << inputs.tenant >>-apps
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
        path: "./apps/<< inputs.tenant >>/<< inputs.environment >>"
        prune: true
        postBuild:
          substituteFrom:
            - kind: ConfigMap
              name: flux-runtime-info
```

Commit and push the changes to the `flux-fleet` repository:

```sh
git add -A
git commit -m "Add apps configuration"
git push origin main
```

Wait for Flux to detect the change in Git or ask it to apply changes immediately:

```sh
flux reconcile kustomization flux-system --with-source
```

Verify that the `ResourceSet` is applied and the apps are deployed:

```sh
watch flux get all -A
```

## Further Reading

Self-service environments:

- [Ephemeral Environments for GitHub Pull Requests](https://fluxcd.control-plane.io/operator/resourcesets/github-pull-requests/)
- [Ephemeral Environments for GitLab Merge Requests](https://fluxcd.control-plane.io/operator/resourcesets/gitlab-merge-requests/)