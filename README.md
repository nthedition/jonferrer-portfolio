# jonferrer-portfolio

The infrastructure behind [jonrferrer.duckdns.org](https://jonrferrer.duckdns.org) — a
personal portfolio site running on a self-managed Kubernetes cluster, entirely within
Oracle Cloud's Always Free tier (2 OCPU / 12GB total, $0/month).

## What's here

- **Two-node k3s cluster** on Oracle Cloud Ampere A1 (ARM64) instances — the entire
  Always Free Ampere allotment, split across a control plane and a worker.
- **Traefik** (bundled with k3s) as ingress, terminating TLS itself via its built-in
  ACME client — real Let's Encrypt certificates, no cloud load balancer, no
  cert-manager.
- **Prometheus + Grafana**, resource-trimmed to fit the free-tier memory budget
  alongside the site, for real cluster monitoring.
- A static portfolio site served straight from a `ConfigMap`-mounted `nginx` pod —
  no build step, no image to push for content changes.

## Layout

| Path | What it is |
|---|---|
| `site/index.html` | The actual portfolio site content |
| `manifests/` | Kubernetes manifests (namespace, ingress, Traefik ACME config, site Deployment/Service) |
| `monitoring/values.yaml` | Trimmed Helm values for `kube-prometheus-stack` |
| `RUNBOOK.md` | Step-by-step instructions for standing this up from scratch on fresh (or wiped) nodes, including the real debugging encountered along the way |

## Why this exists

This started as a Kubernetes study project and became the actual infrastructure for
a portfolio site — the [debug log on the site itself](https://jonrferrer.duckdns.org)
documents real incidents hit while building it (cross-node networking failures,
firewalld/iptables interactions, a stale system service squatting on a port, ACME
setup), not a sanitized "it just worked" writeup.
