---
title: "Installation"
description: "Flux install, bootstrap, upgrade and uninstall documentation."
weight: 30
---

Flux facilitates the uninterrupted deployment of both infrastructure and user-facing applications by employing version control at each stage. This approach guarantees that the deployment process remains reproducible, auditable, and reversible.

Flux consists of several distinct components that work together to create a smooth and manageable data flow within an application.

## Prerequisites

You will need a Kubernetes cluster that matches one of the following versions:

| Kubernetes version | Minimum required |
|--------------------|------------------|
| `v1.24`            | `>= 1.24.0`      |
| `v1.25`            | `>= 1.25.0`      |
| `v1.26`            | `>= 1.26.0`      |
| `v1.27` and later  | `>= 1.27.1`      |

{{% alert color="info" title="Kubernetes EOL" %}}
Note that Flux may work on older versions of Kubernetes e.g. 1.19,
but we don't recommend running [EOL versions](https://endoflife.date/kubernetes)
in production nor do we offer support for these versions.
{{% /alert %}}

## Install the Flux CLI

The Flux CLI is available as a binary executable for all major platforms,
the binaries can be downloaded from GitHub
[releases page](https://github.com/fluxcd/flux2/releases).

{{< tabpane text=true >}}
{{% tab header="Homebrew" %}}

With [Homebrew](https://brew.sh) for macOS and Linux:

```sh
brew install fluxcd/tap/flux
```

{{% /tab %}}
{{% tab header="bash" %}}

With [Bash](https://www.gnu.org/software/bash/) for macOS and Linux:

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

{{% /tab %}}
{{% tab header="yay" %}}

With [yay](https://github.com/Jguer/yay) (or another [AUR helper](https://wiki.archlinux.org/title/AUR_helpers)) for Arch Linux:

```sh
yay -S flux-bin
```

{{% /tab %}}
{{% tab header="nix" %}}

With [nix-env](https://nixos.org/manual/nix/unstable/command-ref/nix-env.html) for NixOS:

```sh
nix-env -i fluxcd
```

{{% /tab %}}
{{% tab header="Chocolatey" %}}

With [Chocolatey](https://chocolatey.org/) for Windows:

```powershell
choco install flux
```

{{% /tab %}}
{{< /tabpane >}}

To configure your shell to load `flux` [bash completions](./cmd/flux_completion_bash.md) add to your profile:

```sh
. <(flux completion bash)
```

[`zsh`](./cmd/flux_completion_zsh.md), [`fish`](./cmd/flux_completion_fish.md),
and [`powershell`](./cmd/flux_completion_powershell.md)
are also supported with their own sub-commands.

A container image with `kubectl` and `flux` is available on DockerHub and GitHub:

* `docker.io/fluxcd/flux-cli:<version>`
* `ghcr.io/fluxcd/flux-cli:<version>`

## Bootstrap with Flux CLI

Using the `flux bootstrap` command you can install Flux on a
Kubernetes cluster and configure it to manage itself from a Git
repository.

### Example to use bootstrap Git with private SSH

Generate an SSH key pair: In the command prompt you will have to generate an SSH key pair:
`ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`

Provide a file path to save the key pair when prompted.

Add the public key to your Git provider:

You have to copy the contents of the public key file which is located at `~/.ssh/id_rsa.pub`.

Go to your Git provider's website and navigate to your account settings. You will find the SSH key settings and add a new SSH key. 
Copy the public key into the field and save settings.

Configure SSH agent:

Start the SSH agent by running the command: 
`eval "$(ssh-agent -s)"`

Add your private key to the SSH agent: 
`ssh-add ~/.ssh/id_rsa` (replace id_rsa with your private key file name if different).

Clone the Bootstrap repository:

Navigate to the directory where you want to clone the repository.
`git clone git@github.com:twbs/bootstrap.git`

If the Flux components are present on the cluster, the bootstrap
command will perform an upgrade if needed. The bootstrap is
idempotent, it's safe to run the command as many times as you want.

The Flux component images are published to DockerHub and GitHub Container Registry
as [multi-arch container images](https://docs.docker.com/docker-for-mac/multi-arch/)
with support for Linux `amd64`, `arm64` and `armv7` (e.g. 32bit Raspberry Pi)
architectures.

If your Git provider is Generic Git Server, AWS CodeCommit, Azure DevOps, Bitbucket Server, GitHub or GitLab please
follow the specific bootstrap procedure:

* [Generic Git Server](./bootstrap/generic-git-server.md)
* [AWS CodeCommit](./bootstrap/aws-codecommit.md#flux-installation-for-aws-codecommit)
* [Azure DevOps](./bootstrap/azure.md#flux-installation-for-azure-devops)
* [Bitbucket Server and Data Center](./bootstrap/bitbucket-server-and-data-center)
* [GitHub.com and GitHub Enterprise](./bootstrap/github-and-github-enterprise)
* [GitLab.com and GitLab Enterprise](./bootstrap/gitlab-and-gitlab-enterprise)

## Bootstrap with Terraform

The bootstrap procedure can be implemented with Terraform using the Flux provider published on
[registry.terraform.io](https://registry.terraform.io/providers/fluxcd/flux).
The provider offers a Terraform resource called
[flux_bootstrap_git](https://registry.terraform.io/providers/fluxcd/flux/latest/docs/resources/bootstrap_git)
that can be used to bootstrap Flux in the same way the Flux CLI does it.

Example of Git HTTPS bootstrap:

```hcl
provider "flux" {
  kubernetes = {
    config_path = "~/.kube/config"
  }
  git = {
    url  = var.gitlab_url
    http = {
      username = var.gitlab_user
      password = var.gitlab_token
    }
  }
}

resource "flux_bootstrap_git" "this" {
  path                   = "clusters/my-cluster"
  network_policy         = true
  kustomization_override = file("${path.module}/kustomization.yaml")
}
```

For more details on how to use the Terraform provider
please see the [Flux docs on registry.terraform.io](https://registry.terraform.io/providers/fluxcd/flux/latest/docs).

## Dev install

For testing purposes you can install Flux without storing its manifests in a Git repository:

```sh
flux install
```

Or using kubectl:

```sh
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml
```

Then you can register Git repositories and reconcile them on your cluster:

```sh
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --tag-semver=">=4.0.0" \
  --interval=1m

flux create kustomization podinfo-default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --validation=client \
  --interval=10m \
  --health-check="Deployment/podinfo.default" \
  --health-check-timeout=2m \
  --target-namespace=default
```

You can register Helm repositories and create Helm releases:

```sh
flux create source helm bitnami \
  --interval=1h \
  --url=https://charts.bitnami.com/bitnami

flux create helmrelease nginx \
  --interval=1h \
  --release-name=nginx-ingress-controller \
  --target-namespace=kube-system \
  --source=HelmRepository/bitnami \
  --chart=nginx-ingress-controller \
  --chart-version="5.x.x"
```

