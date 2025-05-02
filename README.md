# Flux Operator Workshop

[Cloud Native Days Romania](https://cloudnativedays.ro/) - **Applied GitOps with Flux Operator** Workshop

## Prerequisites

For this workshop, you will need a GitHub account and the ability
to create clusters locally using Kubernetes in Docker (KinD).

### Tools

Install the following command line tools:

- [flux](https://fluxcd.io/docs/installation/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [helm](https://helm.sh/docs/intro/install/)
- [yq](https://mikefarah.gitbook.io/yq/)

If you are using macOS, you can install the tools using [Homebrew](https://brew.sh/)
and the provided [Brewfile](Brewfile):

```shell
brew bundle --file=Brewfile
```

Make sure the tools are up to date and available in your `PATH`.

Run `flux version --client` and check that the output version is `v2.5.1`.

### Kubernetes Cluster

You can use any Kubernetes cluster for this workshop (version 1.30 or newer),
but we recommend using Kubernetes KinD and Docker Desktop.

To create a local Kubernetes cluster using KinD, run the following command:

```shell
kind create cluster --name flux-dev --wait 5m
```

To test that the cluster is compatible with Flux, run the following command:

```shell
flux check --pre
```

### GitHub PAT

Navigate to [https://github.com/settings/tokens/new](https://github.com/settings/tokens/new)
and create a GitHub personal access token (classic) with the following scopes:

- `repo` (for pulling Git repositories and commit status updates)
- `read:packages` (for pulling OCI artifacts from GitHub Container Registry)

Export the token as an environment variable:

```shell
export GITHUB_TOKEN=<your-token>
```

To make the PAT available to all shell sessions, you can add the export command to your shell profile.

### GitHub Repository

Navigate to [https://github.com/new](https://github.com/new) and create a public
GitHub repository called `flux-fleet` under your personal GitHub account,
and make sure to initialize it with a `README.md` file.

Clone the repository to your local machine:

```shell
git clone https//github.com/<your-username>/flux-fleet.git
cd flux-fleet
```

Open the repository in your favorite code editor and check that you can push changes to it by modifying the `README.md` file.
We will use this repository to store the GitOps configuration for the workshop.
