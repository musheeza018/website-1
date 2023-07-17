---
title: Node affinity, tolerations and eviction
linkTitle: Node affinity, tolerations and eviction
description: "How to configure Node affinity, tolerations and eviction in Flux"
weight: 70
---

### Node affinity and tolerations

Pin the Flux controllers to specific nodes:

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
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                    - matchExpressions:
                        - key: role
                          operator: In
                          values:
                            - flux
            tolerations:
              - effect: NoSchedule
                key: role
                operator: Equal
                value: flux
    target:
      kind: Deployment
      labelSelector: app.kubernetes.io/part-of=flux
```

The above configuration pins Flux to nodes tainted and labeled with:

```sh
kubectl taint nodes <my-node> role=flux:NoSchedule
kubectl label nodes <my-node> role=flux
```

