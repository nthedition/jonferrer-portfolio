# jonferrer-portfolio

The infrastructure behind [jonrferrer.duckdns.org](https://jonrferrer.duckdns.org) - a
personal portfolio site running on a self-managed Kubernetes cluster, entirely within
Oracle Cloud's Always Free tier (2 OCPU / 12GB total, $0/month).

## What's here

- **Two-node k3s cluster** on Oracle Cloud Ampere A1 (ARM64) instances - the entire
  Always Free Ampere allotment, split across a control plane and a worker.
- **Traefik** (bundled with k3s) as ingress, terminating TLS itself via its built-in
  ACME client — real Let's Encrypt certificates, no cloud load balancer, no
  cert-manager.
- **Prometheus + Grafana**, resource-trimmed to fit the free-tier memory budget
  alongside the site, for real cluster monitoring.
- A portfolio site built and published as a container image by a separate **app repo**,
  [jonferrer-site-portfolio](https://github.com/nthedition/jonferrer-site-portfolio) —
  this repo only references its image tag, it holds no site content itself.

## Layout

Two Kustomize trees, reconciled by two separate Flux `Kustomization`s with
different permission levels — see `infrastructure/rbac.yaml`'s comment for
why they're split this way.

| Path | What it is |
|---|---|
| `apps/portfolio-site/` | Deployment/Service/Ingress for the site — the narrowly-scoped, no-`ClusterRole` tier |
| `infrastructure/` | Namespace, Traefik ACME config, RBAC definitions, Grafana secret (SOPS), Discord notifications — the broader-permission, cluster/bootstrap tier |
| `clusters/oci-k8s-study/` | Flux's own bootstrap output (`flux-system/`, untouched) plus two thin `Kustomization` pointers at the trees above |
| `RUNBOOK.md` | Step-by-step instructions for standing this up from scratch on fresh (or wiped) nodes, including the real debugging encountered along the way |

This is deliberately the **GitOps/cluster-config repo only** — app source and
its build live in
[jonferrer-site-portfolio](https://github.com/nthedition/jonferrer-site-portfolio),
kept separate so this repo's CI only ever validates cluster config, never
builds anything.

## Why this exists

This started as a Kubernetes study project and became the actual infrastructure for
a portfolio site - the [debug log on the site itself](https://jonrferrer.duckdns.org)
documents real incidents hit while building it (cross-node networking failures,
firewalld/iptables interactions, a stale system service squatting on a port, ACME
setup), not a sanitized "it just worked" writeup.
