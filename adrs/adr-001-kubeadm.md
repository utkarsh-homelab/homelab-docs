# ADR-001: Use kubeadm over Managed Kubernetes Distributions

## Status

Accepted

## Context

The Kubernetes cluster needs to be bootstrapped and managed. Options included:
- **kubeadm** (manual, upstream tooling)
- **k3s** / **k3d** (lightweight, single-binary)
- **RKE2** (Rancher's hardened distribution)
- **Talos Linux** (immutable OS, API-driven)
- **Managed services** (EKS, GKE, AKS — not applicable to on-prem homelab)

## Decision

Use **kubeadm** with Ansible automation for cluster lifecycle management.

## Rationale

- **Deepest learning value** — kubeadm exposes every component (etcd, apiserver, scheduler, controller-manager, kubelet, kube-proxy). Troubleshooting a kubeadm cluster requires understanding the control plane internals.
- **Closest to production** — Most on-prem enterprise clusters run kubeadm or a wrapper around it (Rancher, OpenShift, etc.).
- **Vendor independence** — No dependency on a specific distribution. The cluster is standard upstream Kubernetes.
- **Ansible automation** — All `kubeadm init` / `kubeadm join` commands can be wrapped in Ansible playbooks stored in git. Rebuilding the cluster is a single playbook run.
- **Upgrade discipline** — `kubeadm upgrade plan` forces you to follow the correct upgrade order (control planes first, then workers, one node at a time).

## Consequences

- More manual work than k3s or Talos for initial setup
- No built-in etcd backup (must be configured separately via `etcdctl snapshot save`)
- OS patches and kubelet upgrades must be managed separately (we use Ansible for this)
- Higher maintenance burden than a managed distribution

## Alternatives Considered

| Option | Why Not |
|--------|---------|
| k3s | Simpler but too abstracted. Doesn't expose etcd or control plane components. |
| Talos Linux | Excellent but adds another layer (immutable OS API). Less portable learning. |
| RKE2 | Good but tied to Rancher ecosystem. Adds complexity of Rancher management. |
