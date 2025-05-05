# Multi-cluster Setup

## Production Cluster Installation

Create a local Kubernetes cluster called `flux-production` using KinD:

```sh
kind create cluster --name flux-production --wait 5m
```

## Production Cluster Configuration

Copy the `clusters/staging` directory to `clusters/production`:

```sh
cp -r clusters/staging clusters/production
```

Change the `FluxInstance` path in `clusters/production/flux-system/flux-instance.yaml` to `clusters/production`:

```sh
yq -i '.spec.sync.path = "clusters/production"' clusters/production/flux-system/flux-instance.yaml
```

Change the runtime information in `clusters/production/flux-system/flux-runtime-info.yaml` to `production`:

```sh
yq -i '.data.ENVIRONMENT = "production"' clusters/production/flux-system/flux-runtime-info.yaml
yq -i '.data.CLUSTER_NAME = "kind-flux-production"' clusters/production/flux-system/flux-runtime-info.yaml
yq -i '.data.CLUSTER_DOMAIN = "production.example.com"' clusters/production/flux-system/flux-runtime-info.yaml
```

Commit and push the changes to the `flux-fleet` repository:

```sh
git add -A
git commit -m "Add production cluster"
git push origin main
```

## Production Cluster Bootstrap

Install the Flux Operator:

```sh
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace \
  --wait
```

Create a Kubernetes secret with the GitHub PAT named `github-auth` in the `flux-system` namespace:

```sh
flux create secret git github-auth \
  --url=https://github.com/stefanprodan/flux-fleet \
  --username=flux \
  --password=$GITHUB_TOKEN
```

Apply the `flux-instance.yaml` configuration:

```sh
kubectl apply -f clusters/production/flux-system/flux-instance.yaml
```

Wait for the Flux Operator to reconcile the `FluxInstance` and start the Flux controllers:

```sh
kubectl -n flux-system get pods -w
```

Wait for the cluster addons and apps to be deployed:

```sh
watch flux get all -A
```
