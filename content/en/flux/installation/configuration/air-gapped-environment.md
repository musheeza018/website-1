---
title: Air Gapped Environment
linkTitle: Air Gapped Environment
description: "How to configure Air Gapped Environment in Flux"
weight: 20
---

### Air-gapped Environments

To bootstrap Flux on air-gapped environments without access to github.com and ghcr.io, first you'll need 
to download the `flux` binary, and the container images from a computer with access to internet.

List all container images:

```sh
$ flux install --export | grep ghcr.io

image: ghcr.io/fluxcd/helm-controller:v2.0.0-rc.4
image: ghcr.io/fluxcd/kustomize-controller:v2.0.0-rc.4
image: ghcr.io/fluxcd/notification-controller:v2.0.0-rc.4
image: ghcr.io/fluxcd/source-controller:v2.0.0-rc.4
```

Pull the images locally and push them to your container registry:

```sh
docker pull ghcr.io/fluxcd/source-controller:v2.0.0-rc.4
docker tag ghcr.io/fluxcd/source-controller:v2.0.0-rc.4 registry.internal/fluxcd/source-controller:v2.0.0-rc.4
docker push registry.internal/fluxcd/source-controller:v2.0.0-rc.4
```

Copy `flux` binary to a computer with access to your air-gapped cluster,
and create the pull secret in the `flux-system` namespace:

```sh
kubectl create ns flux-system

kubectl -n flux-system create secret generic regcred \
    --from-file=.dockerconfigjson=/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson
```

Finally, bootstrap Flux using the images from your private registry:

```sh
flux bootstrap <GIT-PROVIDER> \
  --registry=registry.internal/fluxcd \
  --image-pull-secret=regcred \
  --hostname=my-git-server.internal
```

Note that when running `flux bootstrap` without specifying a `--version`,
the CLI will use the manifests embedded in its binary instead of downloading
them from GitHub. You can determine which version you'll be installing,
with `flux --version`.

