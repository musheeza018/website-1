---
title: "Bootstrap cheatsheet"
linkTitle: "Bootstrap"
description: "Showcase various configurations of Flux controllers at bootstrap time."
weight: 29
---

## How to customize Flux

To customize the Flux controllers during bootstrap,
first you'll need to create a Git repository and clone it locally.

Create the file structure required by bootstrap with:

```sh
mkdir -p clusters/my-cluster/flux-system
touch clusters/my-cluster/flux-system/gotk-components.yaml \
    clusters/my-cluster/flux-system/gotk-sync.yaml \
    clusters/my-cluster/flux-system/kustomization.yaml
```

The Flux controller deployments, container command arguments, node affinity, etc can be customized using
[Kustomize strategic merge patches and JSON patches](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/patchMultipleObjects.md).

You can make changes to all controllers using a single patch or
target a specific controller:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: # manifests generated during bootstrap
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  # target all controllers
  - patch: | 
      # strategic merge or JSON patch
    target:
      kind: Deployment
      labelSelector: "app.kubernetes.io/part-of=flux"
  # target controllers by name
  - patch: |
      # strategic merge or JSON patch
    target:
      kind: Deployment
      name: "(kustomize-controller|helm-controller)"
  # add a command argument to a single controller
  - patch: |
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --concurrent=5
    target:
      kind: Deployment
      name: "image-reflector-controller"
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
edit the kustomization.yaml file, push the changes upstream
and rerun bootstrap.

## Customization examples

### Safe to evict

Allow the cluster autoscaler to evict the Flux controller pods:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: all
      spec:
        template:
          metadata:
            annotations:
              cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    target:
      kind: Deployment
      labelSelector: app.kubernetes.io/part-of=flux
```

### Allow Helm DNS lookups

By default, the helm-controller will not perform DNS lookups when rendering Helm
templates in clusters because of potential [security
implications](https://github.com/helm/helm/security/advisories/GHSA-pwcw-6f5g-gxf8).

To enable DNS lookups, you must add the `--feature-gates=AllowDNSLookups=true`
flag to the helm-controller Deployment.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      # Allow Helm DNS lookups
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --feature-gates=AllowDNSLookups=true
    target:
      kind: Deployment
      name: helm-controller
```




### Enable notifications for third party controllers

Enable notifications for 3rd party Flux controllers such as [tf-controller](https://github.com/weaveworks/tf-controller):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      - op: add
        path: /spec/versions/0/schema/openAPIV3Schema/properties/spec/properties/eventSources/items/properties/kind/enum/-
        value: Terraform
      - op: add
        path: /spec/versions/1/schema/openAPIV3Schema/properties/spec/properties/eventSources/items/properties/kind/enum/-
        value: Terraform
    target:
      kind: CustomResourceDefinition
      name:  alerts.notification.toolkit.fluxcd.io
  - patch: |
      - op: add
        path: /spec/versions/0/schema/openAPIV3Schema/properties/spec/properties/resources/items/properties/kind/enum/-
        value: Terraform
      - op: add
        path: /spec/versions/1/schema/openAPIV3Schema/properties/spec/properties/resources/items/properties/kind/enum/-
        value: Terraform
    target:
      kind: CustomResourceDefinition
      name:  receivers.notification.toolkit.fluxcd.io
  - patch: |
      - op: add
        path: /rules/-
        value:
          apiGroups: [ 'infra.contrib.fluxcd.io' ]
          resources: [ '*' ]
          verbs: [ '*' ]
    target:
      kind: ClusterRole
      name:  crd-controller-flux-system
```
