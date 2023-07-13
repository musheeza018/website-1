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
### Persistent storage for Flux internal artifacts

Flux maintains a local cache of artifacts acquired from external sources.
By default, the cache is stored in an `EmptyDir` volume, which means that after a restart,
Flux has to restore the local cache by fetching the content of all Git
repositories, Buckets, Helm charts and OCI artifacts. To avoid losing the cached artifacts,
you can configure source-controller with a persistent volume.

Create a Kubernetes PVC definition named `gotk-pvc.yaml` and place it in your `flux-system` directory:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gotk-pvc
  namespace: flux-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 2Gi
```

Add the PVC file to the `kustomization.yaml` resources and patch the source-controller volumes:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
  - gotk-pvc.yaml
patches:
  - patch: |
      - op: add
        path: '/spec/template/spec/volumes/-'
        value:
          name: persistent-data
          persistentVolumeClaim:
            claimName: gotk-pvc
      - op: replace
        path: '/spec/template/spec/containers/0/volumeMounts/0'
        value:
          name: persistent-data
          mountPath: /data
    target:
      kind: Deployment
      name: source-controller
```

### Enable Helm drift detection

At present, Helm releases are not by default checked for drift compared to
cluster-state. To enable experimental drift detection, you must add the
`--feature-gates=DetectDrift=true` flag to the helm-controller Deployment.

Enabling it will cause the controller to check for drift on all Helm releases
using a dry-run Server Side Apply, triggering an upgrade if a change is detected.
For detailed information about this feature, [refer to the
documentation](/flux/components/helm/helmreleases/#drift-detection).

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      # Enable drift detection feature
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --feature-gates=DetectDrift=true
      # Enable debug logging for diff output (optional)
      - op: replace
        path: /spec/template/spec/containers/0/args/2
        value: --log-level=debug
    target:
      kind: Deployment
      name: helm-controller
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
