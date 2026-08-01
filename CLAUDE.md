# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This repo holds only Argo CD `Application` and `AppProject` custom resources — it defines *what* gets deployed and *how it's synced*, not the actual Kubernetes manifests. There is no build, lint, or test tooling (no CI config, no Makefile, no package manager) because there's nothing to compile: every file is a static YAML CR that Argo CD applies directly to the cluster. "Testing" a change means validating the YAML is well-formed and matches the patterns below.

The real workload manifests (Helm values, Kustomize overlays, Vault secret refs) live in a **separate sibling repo**: `https://github.com/Franjly/homelab-pi-k3s.git` (note: no `argocd-` prefix). Almost every `Application.spec.source` in this repo points there. This repo (`argocd-homelab-pi-k3s`) is only used as a source when an app-of-apps parent needs to enumerate child `Application` objects (see below) — those are the only sources that point back at `https://github.com/Franjly/argocd-homelab-pi-k3s.git`.

Rendering of the sibling repo's content happens through custom Argo CD Config Management Plugins referenced via `spec.source.plugin.name`:
- `vault-kustomized-helm` — Kustomize + Helm + Vault secret injection (the default for almost everything)
- `kustomized-helm` — same without Vault (rare; kept as an alternate manifest, e.g. `system/longhorn-system/application/longhorn-system-no-vault.yaml`)
- `vault-kustomized` — Kustomize + Vault, no Helm chart (e.g. `apps/external-endpoints`)

These plugins themselves are not defined in this repo — treat `plugin.name` values as opaque references to cluster-level plugin config.

## Repository layout and conventions

Top-level groups by concern, each containing one subdirectory per component:
- `apps/` — user-facing services (arr stack, calibre, dashboards, home-assistant, myspeed, octoprint, pi-hole, pod-info, romm, external-endpoints)
- `platform/` — core platform services (argocd itself, authelia, gitea, tekton, trivy-system, vault)
- `system/` — cluster infrastructure (calico, cert-manager, cloudflare, ingress-nginx, istio-system, k8tz, kyverno, longhorn-system, metallb-system, monitoring-system, nfs-provisioner, velero)

Every component directory follows:
```
<group>/<component>/
  application/*.yaml   # one or more Application CRs
  project/<component>.yaml   # exactly one AppProject CR
```

### AppProject conventions
- `metadata.name` matches the component directory name, and every `Application` in that component sets `spec.project` to the same name.
- `clusterResourceWhitelist` / `namespaceResourceWhitelist` are wide open (`group: '*', kind: '*'`) — projects exist for repo/destination scoping, not RBAC restriction.
- `sourceRepos` always includes `homelab-pi-k3s.git`. If the component uses the app-of-apps pattern (below), it also lists `argocd-homelab-pi-k3s.git`.

### Two Application patterns
1. **Single-app component** (e.g. `apps/home-assistant`, `apps/myspeed`): one `Application` file whose `source.path` points into the `homelab-pi-k3s` repo and uses a `plugin.name`.
2. **App-of-apps component** (e.g. `apps/arr`, `apps/dashboards`, `system/monitoring-system`, `system/istio-system`, `system/kyverno`, `system/nfs-provisioner`, `platform/trivy-system`): a parent `Application` whose `source.path` is `<group>/<component>/application/apps` in **this** repo (no plugin — Argo CD reads the raw child `Application` YAMLs directly). Each file under `application/apps/` is itself a child `Application` that deploys via a plugin from `homelab-pi-k3s`, using the same `project` as its parent.

When ordering matters within an app-of-apps group, children use `metadata.annotations["argocd.argoproj.io/sync-wave"]` (see `system/istio-system/application/apps/*.yaml`: istio-base=1, istiod=2, istio-ingress=3, istio-egress=4).

### Disabling a component without deleting it
Comment out the entire YAML document with `#` rather than removing the file (see `apps/dashboards/application/apps/homer.yaml`). This keeps the manifest around for reference/re-enabling while Argo CD stops managing it.

### Other patterns worth matching
- `syncPolicy.automated: {}` (auto-sync, no prune/self-heal) plus `syncOptions: [CreateNamespace=true]` is the default for almost every Application. A few explicitly opt into `prune: true` / `selfHeal: true` and `Replace=true` / `ServerSideApply=true` (e.g. `system/kyverno/application/apps/kyverno.yaml`) — only diverge from the default when there's a specific reason (e.g. CRDs that need force-replace).
- `ignoreDifferences` is used sparingly to silence known controller-managed drift, e.g. CRD `selectableFields` churn (`system/cert-manager/application/cert-manager.yaml`) or Kyverno-managed `ClusterRole.rules` (`system/kyverno/application/apps/kyverno.yaml`).

## Making changes

When adding a new component, copy the closest existing analog (single-app vs app-of-apps) rather than building from scratch — the two patterns above cover essentially everything in this repo. Keep `metadata.name`, the AppProject name, and `spec.project` in sync with the directory name.
