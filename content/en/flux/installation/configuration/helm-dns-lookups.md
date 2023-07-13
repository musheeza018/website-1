---
title: Allow Helm DNS lookups 
linkTitle: Allow Helm DNS lookups
description: "Allow Helm DNS lookups"
weight: 120
card:
  name: configuration
  weight: 10
---

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
