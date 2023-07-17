---
title: Controller vertical scaling (renamed from Increase the number of workers)
linkTitle: Controller vertical scaling (renamed from Increase the number of workers)
description: "How to configure controller vertical scaling (renamed from Increase the number of workers) in Flux"
weight: 60
---

### Increase the number of workers

If Flux is managing hundreds of applications, you may want to increase the number of reconciliations
that can be performed in parallel and bump the resources limits:

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
        value: --concurrent=20
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --kube-api-qps=500
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --kube-api-burst=1000
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --requeue-dependency=5s
    target:
      kind: Deployment
      name: "(kustomize-controller|helm-controller|source-controller)"
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
                resources:
                  limits:
                    cpu: 2000m
                    memory: 2Gi
    target:
      kind: Deployment
      name: "(kustomize-controller|helm-controller|source-controller)"
```

{{% alert color="info" title="Horizontal scaling" %}}
When vertical scaling is not an option, you can use sharding to horizontally scale
the Flux controllers. For more details please see the [sharding guide](/flux/installation/configuration/horizontal-scaling).
{{% /alert %}}

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

