---
title: Git repository access via SOCKS5 SSH proxy
linkTitle: Git repository access via SOCKS5 SSH proxy
description: "Git repository access via SOCKS5 SSH proxy"
weight: 140
---


### Git repository access via SOCKS5 SSH proxy

If your cluster has Internet restrictions, requiring egress traffic to go
through a proxy, you must use a SOCKS5 SSH proxy to be able to reach GitHub
(or other external Git servers) via SSH.

To configure a SOCKS5 proxy set the environment variable `ALL_PROXY` to allow
both source-controller and image-automation-controller to connect through the
proxy.

```
ALL_PROXY=socks5://<proxy-address>:<port>
```

The following is an example of patching the Flux setup kustomization to add the
`ALL_PROXY` environment variable in source-controller and
image-automation-controller:

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
                env:
                  - name: "ALL_PROXY"
                    value: "socks5://proxy.example.com:1080"
    target:
      kind: Deployment
      labelSelector: app.kubernetes.io/part-of=flux
      name: "(source-controller|image-automation-controller)"
```
