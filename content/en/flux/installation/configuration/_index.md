---
title: Configuration
linkTitle: Configuration
description: "How to configure Flux"
weight: 30
---

## Customize Flux manifests

You can customize the Flux components before or after running bootstrap.

Assuming you want to customise the Flux controllers before they get deployed on the cluster,
first you'll need to create a Git repository and clone it locally.

Create the file structure required by bootstrap with:

```sh
mkdir -p clusters/my-cluster/flux-system
touch clusters/my-cluster/flux-system/gotk-components.yaml \
    clusters/my-cluster/flux-system/gotk-sync.yaml \
    clusters/my-cluster/flux-system/kustomization.yaml
```

Add patches to `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1
kind: Kustomization
resources: # manifests generated during bootstrap
  - gotk-components.yaml
  - gotk-sync.yaml

patches: # customize the manifests during bootstrap
  - target:
      kind: Deployment
      labelSelector: app.kubernetes.io/part-of=flux
    patch: |
      # strategic merge or JSON patch
```

Push the changes to main branch:

```sh
git add -A && git commit -m "init flux" && git push
```

And run the bootstrap for `clusters/my-cluster`:

```sh
flux bootstrap git \
  --url=ssh://git@<host>/<org>/<repository> \
  --branch=main \
  --path=clusters/my-cluster
```

To make further amendments, pull the changes locally,
edit the `kustomization.yaml` file, push the changes upstream
and rerun bootstrap or let Flux upgrade itself.

Checkout the [bootstrap cheatsheet](../cheatsheets/bootstrap) for various examples of how to customize Flux.

