---
title: "Bootstrap cheatsheet"
linkTitle: "Bootstrap"
description: "Showcase various configurations of Flux controllers at bootstrap time."
weight: 29
---

## This is customization of Flux

To customize the Flux controllers during bootstrap,
first you'll need to create a Git repository and clone it locally.

Create the file structure required by bootstrap with:

```sh
mkdir -p clusters/my-cluster/flux-system
touch clusters/my-cluster/flux-system/gotk-components.yaml \
    clusters/my-cluster/flux-system/gotk-sync.yaml \
    clusters/my-cluster/flux-system/kustomization.yaml
```

```sh
    musheeza 
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


### OpenShift compatibility

Allow Flux controllers to run as non-root:

```shell
#!/usr/bin/env bash
set -e

FLUX_NAMESPACE="flux-system"
FLUX_CONTROLLERS=(
"source-controller"
"kustomize-controller"
"helm-controller"
"notification-controller"
"image-reflector-controller"
"image-automation-controller"
)

for i in ${!FLUX_CONTROLLERS[@]}; do
  oc adm policy add-scc-to-user nonroot system:serviceaccount:${FLUX_NAMESPACE}:${FLUX_CONTROLLERS[$i]}
done
```

Set the user to nobody and delete the seccomp profile from the security context:

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
          spec:
            containers:
              - name: manager
                securityContext:
                  runAsUser: 65534
                  seccompProfile:
                    $patch: delete
    target:
      kind: Deployment
      labelSelector: app.kubernetes.io/part-of=flux
```

### IAM Roles for Service Accounts

To allow Flux access to an AWS service such as KMS or S3, after setting up IRSA,
you can annotate the controller service account with the role ARN:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: kustomize-controller
        annotations:
          eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<KMS-ROLE-NAME>
    target:
      kind: ServiceAccount
      name: kustomize-controller
  - patch: |
      apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: source-controller
        annotations:
          eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<S3-ROLE-NAME>
    target:
      kind: ServiceAccount
      name: source-controller
```

### Multi-tenancy lockdown

Lock down Flux on a multi-tenant cluster by disabling cross-namespace references and Kustomize remote bases, and
by setting a default service account:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --no-cross-namespace-refs=true
    target:
      kind: Deployment
      name: "(kustomize-controller|helm-controller|notification-controller|image-reflector-controller|image-automation-controller)"
  - patch: |
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --no-remote-bases=true
    target:
      kind: Deployment
      name: "kustomize-controller"
  - patch: |
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --default-service-account=default
    target:
      kind: Deployment
      name: "(kustomize-controller|helm-controller)"
  - patch: |
      - op: add
        path: /spec/serviceAccountName
        value: kustomize-controller
    target:
      kind: Kustomization
      name: "flux-system"
```

### Disable Kubernetes cluster role aggregations

By default, Flux [RBAC](/flux/security/#controller-permissions) grants Kubernetes builtin `view`, `edit` and `admin` roles
access to Flux custom resources. To disable the RBAC aggregation, you can remove the `flux-view` and `flux-edit`
cluster roles with:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      apiVersion: rbac.authorization.k8s.io/v1
      kind: ClusterRole
      metadata:
        name: flux
      $patch: delete
    target:
      kind: ClusterRole
      name: "(flux-view|flux-edit)-flux-system"
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
