# Runbook: fresh k3s + portfolio site + monitoring on existing OCI Always Free nodes

Follow this top to bottom. It reuses the two existing OCI instances as-is — **no
`oci compute instance terminate` or `launch` anywhere in this doc** — because
A1.Flex "out of host capacity" errors made recreating instances risky in the past.
We only wipe and reinstall the *software* (k3s), not the VMs.

## Assumptions going in

- Two running instances, both already within the 2 OCPU / 12GB Always Free budget:
  - `oci-k8s-control-plane` — `161.153.55.24`, 1 OCPU/6GB, user `opc`, key `~/.ssh/visiontemplate_vps.key`
  - `oci-k8s-worker-arm1` — `129.146.167.80`, 1 OCPU/6GB, user `opc`, key `~/.ssh/oci_k8s_study`
- Local kubectl context `oci-k8s-study` already exists in `~/.kube/config-personal`.
- OCI security list already allows inbound TCP 22/80/443/6443 from `0.0.0.0/0` and
  6443/8472/10250 between nodes (set up in phase 2b). If you changed that since,
  re-check `oci network security-list get` before step 3.

---

## Step 1 — Wipe k3s on both nodes

```bash
ssh -i ~/.ssh/oci_k8s_study opc@129.146.167.80 "sudo /usr/local/bin/k3s-agent-uninstall.sh"
ssh -i ~/.ssh/oci-control-plane.key opc@161.153.55.24 "sudo /usr/local/bin/k3s-uninstall.sh"
```
Worker first, then control-plane — no strict requirement, but avoids a worker
briefly trying to reach a control-plane that's already gone. Both uninstall
scripts are installed automatically by `get.k3s.io` and remove the binary,
systemd units, and `/etc/rancher`/`/var/lib/rancher` state.

## Step 2 — Reinstall fresh

**Control plane** (pin the node name explicitly from the start — avoids the
hostname-reset gotcha documented in phase 2b §9):
```bash
ssh -i ~/.ssh/oci-control-plane.key opc@161.153.55.24 \
  "curl -sfL https://get.k3s.io | sudo K3S_NODE_NAME=k8s-control-plane \
   INSTALL_K3S_EXEC='server --node-external-ip=161.153.55.24 --tls-san=161.153.55.24' sh -"

ssh -i ~/.ssh/oci-control-plane.key opc@161.153.55.24 "sudo cat /var/lib/rancher/k3s/server/node-token"
```
Save the printed token — the worker needs it next.

**Worker:**
```bash
ssh -i ~/.ssh/oci_k8s_study opc@129.146.167.80 \
  "curl -sfL https://get.k3s.io | sudo K3S_NODE_NAME=k8s-worker-arm1 \
   K3S_URL=https://10.0.0.247:6443 K3S_TOKEN=<token-from-above> \
   INSTALL_K3S_EXEC='agent --node-external-ip=129.146.167.80' sh -"
```
`K3S_URL` uses the control-plane's **private** IP (`10.0.0.247`) — agent traffic
stays inside the VCN, matching how it was set up before. If the control-plane's
private IP changed (it shouldn't, since we didn't recreate the VM), get the
current one with `ssh ... "hostname -I"` on the control-plane first.

## Step 3 — Verify and re-wire local kubectl

k3s reinstall generates a **new** cluster CA/certs, so the old kubeconfig entry
is now invalid — this isn't optional to skip.

Purge any stale `oci-k8s-study` entries first, and list the **freshly-fetched
file first** in `KUBECONFIG` — `kubectl config view --flatten` keeps the
*first*-seen cluster/context/user when names collide across files, so the new
file must win even if the purge misses something (a reinstall's new CA won't
match whatever `config-personal` already had, producing
`certificate signed by unknown authority` otherwise):
```bash
kubectl --kubeconfig=~/.kube/config-personal config delete-context oci-k8s-study 2>/dev/null
kubectl --kubeconfig=~/.kube/config-personal config delete-cluster oci-k8s-study 2>/dev/null
kubectl --kubeconfig=~/.kube/config-personal config unset users.oci-k8s-study 2>/dev/null
```

```bash
ssh -i ~/.ssh/oci-control-plane.key opc@161.153.55.24 "sudo cat /etc/rancher/k3s/k3s.yaml" > /tmp/oci-k3s.yaml
sed -i 's#127.0.0.1#161.153.55.24#; s/default/oci-k8s-study/g' /tmp/oci-k3s.yaml
KUBECONFIG=/tmp/oci-k3s.yaml:~/.kube/config-personal kubectl config view --flatten > /tmp/merged
mv /tmp/merged ~/.kube/config-personal
rm /tmp/oci-k3s.yaml

kubectl --context oci-k8s-study get nodes -o wide
```
Expect both nodes `Ready` within ~30-60s of the worker install finishing.

## Step 4 — Deploy the portfolio site

The site content lives at `site/index.html` — a single self-contained static
page (no build step). The root `kustomization.yaml` turns it into a
`ConfigMap` via `configMapGenerator`, which hashes the file's content into
the generated name (e.g. `portfolio-html-btmg9c6mc9`). Kustomize also rewrites
the Deployment's volume reference to match automatically, so **every real
content change produces a new ConfigMap name and forces a rolling pod
update** — no manual restart, no waiting on the kubelet's sync delay.

```bash
cd ~/projects/personal/kubernetes-study/portfolio-deployment
kubectl --context oci-k8s-study apply -k .
```

That one command applies everything — namespace, site ConfigMap+Deployment+
Service, ingress, and the Traefik ACME config. Editing `site/index.html` and
re-running it is the entire update workflow (once Flux is set up per Step 7,
this happens automatically on every push instead).

**Quick test without a domain**, using Traefik's host-port directly:
```bash
curl -H "Host: portfolio.example.com" http://161.153.55.24
```
Should return the site content. Once you point real DNS at a node's IP, drop
the `-H "Host: ..."` override.

If you outgrow a single static HTML file (framework, build step, etc.), swap
the `nginx:1.27-alpine` image + `ConfigMap` approach in
`manifests/portfolio-placeholder.yaml` for an actual built image pushed
somewhere pullable — GHCR is free for public images.

## Step 5 — Install Prometheus + Grafana (trimmed)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f monitoring/values.yaml \
  --set grafana.adminPassword="$(openssl rand -base64 18)"
```
Don't hardcode a real password into `values.yaml` and commit it — the
`--set` above generates a random one at install time. Retrieve it after:
```bash
kubectl --context oci-k8s-study -n monitoring get secret monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

`monitoring/values.yaml` disables Alertmanager, drops Prometheus retention to
6h with no PVC, and caps every component's requests/limits — sized to fit
alongside the portfolio site inside the 12GB total budget, not the chart's
defaults (which alone can exceed 1GB+ per component).

**Access Grafana** (no ingress wired up for it yet, deliberately — port-forward
keeps it off the public internet):
```bash
kubectl --context oci-k8s-study -n monitoring port-forward svc/monitoring-grafana 3000:80
# browse http://localhost:3000, log in admin / <password from above>
```
The `kube-prometheus-stack` chart ships default dashboards (Kubernetes /
Compute Resources / Cluster, Node Exporter / Nodes) already wired to the
Prometheus datasource — nothing further to configure to see live CPU/memory
per node.

## Step 6 — Sanity-check the resource budget

```bash
kubectl --context oci-k8s-study top nodes
kubectl --context oci-k8s-study top pods -A
```
Everything here (k3s system pods + portfolio site + trimmed monitoring stack)
should sit well under 12GB combined memory and leave real headroom. If `top`
shows a node above ~80% memory, that's the signal to trim `monitoring/values.yaml`
further before adding anything else — not to resize/add nodes (would exceed
the free budget).

## Note on the idle-reclamation threshold

Oracle reclaims an Always Free A1 instance if, over a 7-day window, **all**
of CPU (95th percentile), network, and memory utilization stay under 20%
(see the [Always Free resources docs](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)).
Prometheus's own scrape loop (every 30s by default) plus Grafana
plus the portfolio site's baseline traffic should keep memory utilization
comfortably over that line on its own — no separate synthetic load generator
needed. If `top nodes` after a week shows otherwise, that's worth a follow-up
before it becomes a reclamation risk, not something to guess about now.

## Step 7 — TLS via Traefik's ACME (Let's Encrypt)

`manifests/traefik-acme-config.yaml` is a `HelmChartConfig` for the `traefik`
release k3s manages automatically — applying it triggers k3s's built-in
helm-controller to redeploy Traefik with ACME enabled (HTTP-01 challenge) and
a small persistent volume (`local-path` storage class, k3s's built-in
provisioner) for `acme.json`. Persistence matters: without it, the cert is
lost on every pod restart, and re-requesting one too often risks hitting
Let's Encrypt's rate limits.

```bash
kubectl --context oci-k8s-study apply -f manifests/traefik-acme-config.yaml
```

No separate `helm upgrade` needed for this one (unlike monitoring) — k3s's
helm-controller does it automatically when the `HelmChartConfig` changes.

`manifests/portfolio-ingress.yaml` requests the cert via annotations
(`traefik.ingress.kubernetes.io/router.tls.certresolver: letsencrypt`) rather
than a `cert-manager` Certificate resource — simpler for a single ingress,
no extra controller to install. No `tls.secretName` is set on purpose: with a
certresolver annotation, Traefik keeps the issued cert in its own
`acme.json`, not a Kubernetes Secret.

```bash
kubectl --context oci-k8s-study apply -f manifests/portfolio-ingress.yaml
curl https://<your-domain>
```

**Gotcha that cost real time getting here**: DuckDNS's actual service is at
**`duckdns.org`** — `duckdns.com` is an unrelated, unaffiliated domain that
happens to resolve *any* subdomain (real or made up) to its own generic
parking IPs. If `dig` shows a hostname flapping between a couple of fixed
IPs regardless of what you set on the DuckDNS dashboard, check you're not
accidentally testing `.com`.

## Step 8 — Flux (GitOps)

Once this repo is pushed to GitHub, Flux reconciles the cluster against it
automatically — a `git push` to `main` becomes the deploy step, no manual
`kubectl apply` needed going forward.

```bash
export GITHUB_TOKEN="$(gh auth token)"
flux bootstrap github \
  --context=oci-k8s-study \
  --owner=<your-github-username> \
  --repository=jonferrer-portfolio \
  --branch=main \
  --path=clusters/oci-k8s-study \
  --personal
```

This installs Flux's controllers into the `flux-system` namespace and commits
its own bootstrap manifests back into `clusters/oci-k8s-study/flux-system/`
in the repo. It does **not** yet know to deploy the site/monitoring — that
needs an explicit `GitRepository` + `Kustomization` pointing at the repo
root, added separately (see `clusters/oci-k8s-study/apps.yaml`).

Flux picked over Argo CD here deliberately: it's modular (source-controller +
kustomize-controller only, no bundled UI/Redis/Dex), which matters on a
12GB-total cluster already running Prometheus/Grafana.

## Not covered here (deliberately, follow-up work)

- **CI/CD** for the actual portfolio site content (build → push image →
  `kubectl set image` or a GitOps tool) — depends on what the real site is
  built with, not decided yet.
