---
title: Enable Helm repositories caching 
linkTitle: Enable Helm repositories caching
description: "How to enable Helm repositories caching in Flux"
weight: 90
---

### Enable Helm repositories caching

For large Helm repository index files, you can enable
caching to reduce the memory footprint of source-controller:

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
        value: --helm-cache-max-size=10
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --helm-cache-ttl=60m
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --helm-cache-purge-interval=5m
    target:
      kind: Deployment
      name: source-controller
```

When `helm-cache-max-size` is reached, an error is logged and the index is instead
read from file. Cache hits are exposed via the `gotk_cache_events_total` Prometheus
metrics. Use this data to fine-tune the configuration flags.

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

