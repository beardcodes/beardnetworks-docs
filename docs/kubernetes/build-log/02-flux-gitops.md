# K3s Build Log, Part 2: GitOps with Flux CD and GitHub

*November 2026 · K3s on Proxmox · Real homelab build log*

In [Part 1](01-proxmox-vm.md), I got a K3s server running inside a Proxmox VM and configured `kubectl` on my Windows desktop. That gave me a working Kubernetes cluster, but I still needed a way to manage it without manually running `kubectl apply` every time I wanted to change something.

This is where I introduced **GitOps**.

My goal was simple: make the GitHub repository the place where I define what should be running in Kubernetes, then let the cluster pull those definitions and reconcile itself. I wanted to learn that workflow before putting important services or databases on K3s.

The actual configuration lives in my private `beardcodes/k3s-homelab` repository. This article follows the repository's real structure and manifests. Commands that recreate the initial bootstrap are presented as reproduction steps, not as a verbatim export of my shell history.

## 1. Why I chose Flux CD

I already had plenty of services running in Docker across the homelab. Docker Compose had taught me the value of keeping service definitions in files instead of relying on settings made in a web dashboard. Kubernetes takes that idea further, but it doesn't automatically give you a Git-based deployment workflow.

I wanted a setup where I could:

- Edit Kubernetes manifests in VS Code on Windows.
- Commit the changes to GitHub.
- Have the cluster notice and apply those changes.
- See whether the desired state was actually reconciled.
- Revert a bad change through Git rather than remembering which commands I had run manually.

I picked [Flux CD](https://fluxcd.io/) because it runs inside the cluster, supports Kustomize and Helm, and works well with Git repositories. In a pull-based GitOps model, my laptop isn't responsible for continuously deploying applications. The controllers running in K3s perform reconciliation.

That doesn't mean GitOps replaces backups, monitoring, or review. It means **Git becomes the source of truth for the configuration I intentionally put under Flux management**.

## 2. Prepare the Windows workstation

I was using VS Code and PowerShell on Windows, with the K3s kubeconfig stored at:

```text
$HOME\.kube\k3s-node1.yaml
```

The first thing was to make sure my terminal was pointing at the right cluster:

```powershell
$env:KUBECONFIG = "$HOME\.kube\k3s-node1.yaml"

kubectl get nodes -o wide
kubectl get pods -A
```

I then installed the Flux CLI on Windows. The exact installation method can vary, so I recommend following the [official Flux CLI installation guide](https://fluxcd.io/flux/installation/) rather than copying a version-specific installer from a blog post.

With `flux` available in the terminal:

```powershell
flux --version
flux check --pre
```

My repository's generated manifests identify **Flux v2.10.0**, with four controllers:

- `source-controller` — fetches Git and other supported sources.
- `kustomize-controller` — applies Kubernetes manifests.
- `helm-controller` — manages Helm releases.
- `notification-controller` — handles events and notifications.

The preflight check matters because it's much easier to fix API or permission issues before trying to bootstrap anything.

## 3. Bootstrap Flux against GitHub

I used a dedicated repository named `k3s-homelab` in my `beardcodes` GitHub account.

The equivalent Flux bootstrap command for this layout is:

```powershell
flux bootstrap github --owner=beardcodes --repository=k3s-homelab --branch=main --path=clusters/homelab --personal
```

!!! warning "Reproduction command"

    This is a reproduction command, not a command to rerun against an already working cluster. Bootstrap requires a GitHub personal access token with appropriate repository permissions. Flux prompts for it if it isn't provided through its supported configuration.

I used GitHub access over SSH for the cluster's Git source, not an HTTPS URL containing a personal access token. The current `gotk-sync.yaml` shows:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: ssh://git@github.com/beardcodes/k3s-homelab
```

Flux also created a Kustomization that watches `./clusters/homelab`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/homelab
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

That distinction is important: the **GitRepository** describes *where* Flux gets the configuration, while the **Flux Kustomization** describes *which path* to reconcile into the cluster.

The SSH private key is stored in a Kubernetes Secret inside `flux-system`; it is **not** committed to GitHub. My local kubeconfig is also kept outside the repository.

## 4. Understand the repository structure

Once bootstrap was working, the real benefit became clearer. I could separate cluster entry points, infrastructure, and applications rather than throwing everything into one directory.

Here is a shortened view of the layout I ended up with:

```text
k3s-homelab/
├── clusters/
│   └── homelab/
│       ├── flux-system/
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── apps.yaml
│       ├── gateway-api.yaml
│       ├── traefik.yaml
│       ├── cert-manager.yaml
│       └── tls.yaml
├── infrastructure/
│   ├── gateway-api/
│   ├── traefik/
│   ├── cert-manager/
│   └── tls/
└── apps/
    ├── kustomization.yaml
    ├── podinfo/
    ├── uptime-kuma/
    ├── monitoring/
    └── homepage/
```

The `flux-system` files are generated by Flux and are marked **DO NOT EDIT**. I leave those alone and add my own configuration around them.

One detail that confused me initially is that the word **Kustomization** refers to two related but different things:

- A **Flux `Kustomization` resource** tells a controller which repository directory to reconcile and how often.
- A **Kustomize `kustomization.yaml` file** lists the local Kubernetes resources that belong together.

My `clusters/homelab/apps.yaml` is the first kind:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

And `apps/kustomization.yaml` is the second kind:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - podinfo
  - uptime-kuma
  - monitoring
  - homepage
```

That one `apps` Flux object manages the application directory. The subdirectories each have their own manifests and, where useful, their own Kustomize files.

!!! warning "A caution about `prune: true`"

    It allows Flux to remove previously managed resources that disappear from the declared configuration. That's useful for keeping Git and the cluster aligned, but it's also a reason to review deletions carefully, especially once workloads have persistent data.

## 5. Deploy a test app: Podinfo

Rather than starting with a database or a critical household service, I used **Podinfo**, a small sample web application made for Kubernetes demonstrations.

In my repository, `apps/podinfo/podinfo.yaml` defines a namespace, an OCI chart source and a Helm release:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: podinfo
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: podinfo
  namespace: podinfo
spec:
  interval: 1h
  url: oci://ghcr.io/stefanprodan/charts/podinfo
  ref:
    semver: ">=6.0.0 <7.0.0"
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  interval: 10m
  chartRef:
    kind: OCIRepository
    name: podinfo
  values:
    replicaCount: 1
```

Here's what those pieces do:

1. **Namespace** isolates Podinfo's namespaced resources from other applications.
2. **OCIRepository** points Flux at the Helm chart stored in GitHub Container Registry.
3. **HelmRelease** tells the Helm controller to install the chart with one replica.

The version selector allows compatible 6.x chart versions rather than pinning a single patch release. That's convenient in a lab, but it also means a later reconciliation may pick up a newer matching chart version. For more controlled environments I'd consider pinning a version or digest and updating it deliberately.

The application's `apps/podinfo/kustomization.yaml` is small:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - podinfo.yaml
  - httproute.yaml
```

The `httproute.yaml` wasn't enough by itself to give Podinfo a working public or LAN URL. That route depends on the **Gateway API CRDs and a Traefik Gateway**, which I configured as the next stage of the homelab. The Podinfo Helm release is the GitOps test here; routing is the subject of Part 3.

## 6. The deployment workflow I now use

Once everything was in Git, I stopped treating ad-hoc `kubectl apply` commands as my primary deployment method.

My normal workflow on Windows is:

```powershell
# Review local changes
git status
git diff --check

# Check that Kustomize can render the application tree
kubectl kustomize apps | Out-Null

# Commit and push the intended configuration
git add apps/
git commit -m "feat: update homelab apps"
git push origin main
```

Flux's Git source checks `main` every minute, and the `apps` Kustomization has a ten-minute reconciliation interval. When I want to request an immediate sync, I run:

```powershell
flux reconcile kustomization apps --with-source
```

Then I check the actual state instead of assuming the command succeeded:

```powershell
flux get sources git -A
flux get kustomizations -A
flux get helmreleases -A
kubectl get pods -n podinfo
```

This separation of **desired state** (Git) from **observed state** (Kubernetes) is one of the biggest things I learned.

## 7. A lesson from a real Flux timeout

Later in this build, when I updated Homepage, Flux fetched a new Git revision successfully but the CLI waited five minutes for the `apps` Kustomization and returned:

```text
context deadline exceeded
```

That message didn't prove GitHub was broken, and it didn't automatically mean the application was running. It meant **the client stopped waiting before reconciliation completed**.

In my case, inspecting the actual deployment manifest revealed an accidental typo in a Kubernetes volume name: `homepage-logs` had acquired two trailing backticks. That wasn't a Flux bug. The YAML tooling could render the file, but Kubernetes couldn't accept the invalid volume name.

The useful troubleshooting sequence became:

```powershell
flux get kustomizations -A
flux get helmreleases -A
kubectl get events -n homepage --sort-by=.lastTimestamp
kubectl describe deployment homepage -n homepage
```

Then fix the manifest in Git, push it and reconcile again.

This is what I want from GitOps: not a guarantee that I can't make mistakes, but a clear record of *what I intended to deploy* and a way to correct that configuration without relying on terminal history.

## 8. What I learned

There are a few things I'd repeat if I built this from zero again.

**Keep the first application disposable.** Podinfo was a much better first test than a password manager or database.

**Learn the repository structure early.** Splitting `clusters/`, `infrastructure/` and `apps/` saved me confusion as the homelab grew.

**Protect credentials from the start.** The Git deploy key, Kubernetes kubeconfig and later API tokens belong in secrets or secure local storage, not committed as plaintext manifests.

**Validation has layers.** `git diff --check` catches whitespace problems, `kubectl kustomize` checks rendering, and Flux plus the Kubernetes API tell you whether the resource was actually accepted and running.

**A successful Git push is not a successful deployment.** I now check Flux status and Kubernetes pods after meaningful changes.

By the end of this stage, I had a cluster reading its configuration from GitHub and a small application managed through Flux's Helm controller. More importantly, I had a workflow I could use for everything that followed.

*Next in Part 3: I replace the idea of "just expose another port" with Traefik and the Kubernetes Gateway API, then make Podinfo reachable through an HTTPRoute.*

---

**Official references:** [Flux GitHub bootstrap](https://fluxcd.io/flux/installation/bootstrap/github/) · [Flux Helm releases](https://fluxcd.io/flux/guides/helmreleases/) · [Kustomize](https://kustomize.io/)
