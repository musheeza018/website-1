---
title: Multi-tenancy lockdown
linkTitle: Multi-tenancy lockdown
description: "How to configure Multi-tenancy lockdown in Flux"
weight: 40
---

### Multi-tenancy lockdown

Assuming you want to lock down Flux on multi-tenant clusters,
add the following patches to `clusters/my-cluster/flux-system/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  - patch: |
      - op: add
        path: /spec/template/spec/containers/0/args/0
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
        path: /spec/template/spec/containers/0/args/0
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

With the above configuration, Flux will:

- Deny cross-namespace access to Flux custom resources, thus ensuring that a tenant can't use another tenant's sources or subscribe to their events.
- Deny accesses to Kustomize remote bases, thus ensuring all resources refer to local files, meaning only the Flux Sources can affect the cluster-state.
- All `Kustomizations` and `HelmReleases` which don't have `spec.serviceAccountName` specified, will use the `default` account from the tenant's namespace.
  Tenants have to specify a service account in their Flux resources to be able to deploy workloads in their namespaces as the `default` account has no permissions.
- The flux-system `Kustomization` is set to reconcile under a service account with cluster-admin role,
  allowing platform admins to configure cluster-wide resources and provision the tenant's namespaces, service accounts and RBAC.

To apply these patches, push the changes to the main branch and run `flux bootstrap`.

### Multi-tenancy lockdown cluster

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

