# homelab

Production-grade Kubernetes homelab running on a bare-metal machine. Built to host my own home setup and practice the same tools used in real infrastructure teams — GitOps, immutable OS, long-term observability, and infrastructure-as-code from the ground up.

> The full repository containing the complete cluster setup, configurations, and runbooks is private for security reasons (credentials, internal network topology, node details). This repo serves as an overview of the architecture and design decisions.

---

## Stack

| Layer | Tool | Why |
|---|---|---|
| Hypervisor | [Proxmox VE](https://www.proxmox.com/) | Mature bare-metal virtualisation, good community |
| OS / Kubernetes | [Talos Linux](https://www.talos.dev/) | Immutable, API-driven, no SSH, minimal attack surface |
| Infra provisioning | [OpenTofu](https://opentofu.org/) | IaC for VMs — reproducible cluster from scratch |
| Config management | [Ansible](https://www.ansible.com/) | VM post-provisioning — Tailscale install and subnet router configuration |
| GitOps | [ArgoCD](https://argo-cd.readthedocs.io/) | Declarative, Git-driven deployments with drift detection |
| Ingress | [Envoy Gateway](https://gateway.envoyproxy.io/) | Kubernetes Gateway API — the successor to Ingress |
| Load balancer | [MetalLB](https://metallb.universe.tf/) | Assigns real LAN IPs to LoadBalancer services (L2 mode) |
| LAN DNS | [CoreDNS](https://coredns.io/) | Runs in-cluster, resolves custom hostnames across the LAN |
| Storage | [local-path-provisioner](https://github.com/rancher/local-path-provisioner) | Simple node-local PVs, no external dependencies |
| Object storage | [MinIO](https://min.io/) | S3-compatible store for Thanos long-term metrics |
| Metrics | [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) + [Thanos](https://thanos.io/) | Short-term in Prometheus, long-term in MinIO via Thanos |
| Logs | [Loki](https://grafana.com/oss/loki/) + [Alloy](https://grafana.com/oss/alloy/) | Log aggregation and collection |
| Dashboards | [Grafana](https://grafana.com/) | Unified view for metrics and logs |
| Alerting | Grafana Unified Alerting | Alert rules as code, routed to Telegram |
| Remote access | [Tailscale](https://tailscale.com/) | WireGuard-based VPN, subnet router on a dedicated Debian VM |

---

## Cluster

3 VMs on a single bare-metal host (Proxmox):

| Role | CPU | RAM | Disk |
|---|---|---|---|
| Control plane × 1 | 2 vCPU | 5 GB | 10 GB |
| Worker × 2 | 2 vCPU | 3 GB | 60 GB |

Workers are memory-constrained (3 GB each), so workloads are pinned via `nodeSelector` to keep memory balanced across both nodes:

- **Data worker** — PVC-bound workloads that can't move once scheduled: Prometheus, MinIO, Loki
- **Stateless worker** — everything that has no storage dependency: ArgoCD, Grafana, Envoy Gateway, CoreDNS, kube-state-metrics

DaemonSets (Alloy, MetalLB speaker, node-exporter) run on all nodes.

---

## Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │                Bare-metal host               │
                    │                   Proxmox VE                 │
                    │                                             │
                    │   ┌─────────────┐   ┌─────────────────┐   │
                    │   │Control plane│   │    Workers ×2   │   │
                    │   │ Talos Linux │   │  Talos Linux    │   │
                    │   └──────┬──────┘   └────────┬────────┘   │
                    └──────────┼──────────────────-┼─────────────┘
                               │    Kubernetes      │
                    ┌──────────▼────────────────────▼──────────┐
                    │                                          │
                    │   ┌──────────┐     ┌─────────────────┐  │
                    │   │  ArgoCD  │────▶│ ApplicationSets │  │
                    │   └──────────┘     └────────┬────────┘  │
                    │                             │ syncs      │
                    │              ┌──────────────▼──────────┐ │
                    │              │        Addons           │ │
                    │   Ingress    │  MetalLB · CoreDNS      │ │
                    │   ┌──────┐   │  Envoy Gateway          │ │
  Browser ─────────┼──▶│      │   │                         │ │
                    │   │Envoy │   │  Observability          │ │
                    │   │  GW  │   │  Alloy · Prometheus     │ │
                    │   └──┬───┘   │  Thanos · MinIO         │ │
                    │      │       │  Loki · Grafana         │ │
                    │      │       │    │ Alerting            │ │
                    │      │       │    └▶ Telegram            │ │
                    │      ▼       └─────────────────────────┘ │
                    │   Services                               │
                    └──────────────────────────────────────────┘
```

---

## GitOps design

No manual `kubectl apply` for workloads. Everything is a Helm chart wrapper committed to Git. ArgoCD watches the repo and reconciles the cluster to match.

**ApplicationSets** auto-discover addons by scanning for directories containing a `config.json`:

```
gitops/addons/grafana/config.json   →   ArgoCD creates Application "grafana"
gitops/addons/loki/config.json      →   ArgoCD creates Application "loki"
```

Adding a new addon is three files and a push. Removing it is renaming `config.json` to `config.json.disabled`.

Each addon follows the same structure:

```
addons/my-addon/
├── config.json    # app name + target namespace
├── Chart.yaml     # Helm wrapper with upstream chart as dependency
└── values.yaml    # all configuration
```

---

## Observability

Full metrics + logs pipeline built from open-source components:

```
Collection:    Alloy (DaemonSet) ──────────────┐
                                               │
               ┌── metrics ──▶ Prometheus ─────┤
               │               (2d hot, 10Gi)  │
               │                    │          │
               │               Thanos Sidecar  │
               │                    │ blocks   │
               │                    ▼          │
               │               MinIO (20Gi) ◀──┘
               │                    │
               │               Thanos Store
               │                 Gateway
               │                    │
               └── logs ──▶ Loki    │
                                    │
                            Thanos Query
                                    │
                            Thanos QueryFrontend
                                    │
                               Grafana ◀── Loki
                                    │
                            Unified Alerting
                            (rules as code)
                                    │
                                    │
                                Telegram
                                 (bot)
```

**Metrics retention:**
- Prometheus keeps 2 days of hot data locally
- Thanos sidecar uploads 2-hour TSDB blocks to MinIO every 2 hours
- Thanos Compactor enforces a 7-day raw retention, 30-day 5m downsampled, 90-day 1h downsampled
- Grafana points to Thanos QueryFrontend — queries transparently span both hot and historical data

**Alerting:**
Grafana Unified Alerting handles routing directly — no Alertmanager. For a single-cluster homelab, Grafana's built-in alerting covers everything needed: it already has the datasource, dashboards, and notification channels in one place, so there's no real reason to run Alertmanager as a separate component. That said, if the setup grows in complexity (multiple clusters, more advanced routing or silencing needs), switching to Alertmanager is a straightforward path.

Alert rules are defined inline in `grafana/values.yaml` and loaded at pod startup as mounted volumes — no sidecar or reload API involved. 11 rules across 4 groups:

| Group | Rules |
|---|---|
| Nodes | Node down, Disk > 80%, Memory > 85%, PVC > 80% |
| Workloads | Pod crash looping, OOMKill (fires only on recent restarts), Pod not ready > 5m |
| ArgoCD | App unhealthy, App out of sync > 15m |
| Observability | Prometheus target down, Thanos not uploading > 3h |

Rules that detect absence of a metric (node down, target down, Thanos stale) use `noDataState: Alerting`. Threshold rules (disk, memory, crash loop, etc.) use `noDataState: OK` — no data means the condition isn't met. Notifications go to a Telegram bot (email was dropped — redundant once Telegram was reliable). Credentials are injected at runtime from a Kubernetes secret — nothing sensitive in Git.

**Why Thanos instead of just Prometheus?**
Prometheus is intentionally short-lived with local storage. Thanos decouples storage from the scraping layer — metrics survive node failures and aren't limited by local disk. MinIO provides the S3-compatible backend without external cloud dependencies.

---

## Networking

Traffic flows through a single entry point:

```
Browser → LAN DNS (CoreDNS) → resolves *.homelab.internal
        → MetalLB VIP → Envoy Gateway → HTTPRoute → Service → Pod
```

CoreDNS runs inside the cluster and handles DNS for all internal hostnames. MetalLB assigns real LAN IPs to the `LoadBalancer` services (Envoy Gateway, CoreDNS) using L2 advertisement — no BGP required.

**Why Envoy Gateway over a standard Ingress controller?**
Envoy Gateway implements the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/), which is the intended successor to the Ingress resource. `HTTPRoute` gives per-route control, cross-namespace routing without ReferenceGrants (via `allowedRoutes: from: All`), and a cleaner separation between the infrastructure owner (Gateway) and the app developer (HTTPRoute).

---

## Interesting challenges

**Talos has no shell.** Debugging node-level issues (disk pressure, PVC directories) requires running privileged pods in the cluster or using `talosctl` for read-only inspection. This forces proper GitOps discipline — you can't make one-off changes on the node.

**local-path PVCs have no size enforcement — and worse, they can deadlock a node.** A 20 Gi PVC is a label, not a hard cap. Thanos blocks were accumulating in MinIO at ~2.5 GB/day with no retention policy, filling a 30 GB disk in ~6 days. Adding Compactor retention and doubling the scrape interval (30s → 60s) slowed the growth, but didn't fix the underlying risk: a `local-path` PV pins its pod to the node it was first scheduled on. When the disk filled again later, MinIO was evicted for disk pressure and couldn't reschedule anywhere else — and with MinIO down, the Compactor couldn't reach the bucket to enforce retention or free space, so the node could never recover on its own. The actual fix was doubling the worker disks (30 GB → 60 GB) for real headroom; retention policy alone only buys time against the growth rate, it doesn't prevent the deadlock.

**ArgoCD + Helm hooks don't always mix.** kube-prometheus-stack annotates its CRDs and admission webhook resources as `pre-install` Helm hooks. ArgoCD processes these as hooks and waits for them to reach a terminal state — but `ServiceAccount` and `ClusterRole` resources have no terminal state, so ArgoCD hangs indefinitely. Fixed by disabling the admission webhook setup entirely (not needed for a homelab) and excluding `batch/Job` resources from ArgoCD tracking.

**Workload placement on memory-constrained nodes.** With 3 GB RAM per worker, a single out-of-place heavy pod tips the balance. Everything stateless is pinned to one worker via `nodeSelector`, and everything PVC-bound lands on the other by necessity. ArgoCD's application-controller (349 Mi actual) was the swing factor — moving it to the data worker brought both nodes to ~55% utilisation.

---

## Remote access

Tailscale runs on a dedicated Debian 12 VM on Proxmox, acting as a subnet router that advertises the `192.168.1.0/24` LAN to the Tailscale mesh. When connected from any device, the full homelab network is reachable — including internal services and `*.london.internal` hostnames.

The VM is provisioned via a generic, distro-agnostic OpenTofu module (`infrastructure/proxmox/modules/vm/`) and lives in its own independent Terraform root (`infrastructure/proxmox/vms/tailscale/`), completely decoupled from the Kubernetes cluster. This means it survives a full cluster rebuild and can be managed independently.

After Terraform creates the VM, an Ansible playbook (`infrastructure/ansible/playbooks/tailscale.yml`) handles post-provisioning: enabling IPv4 forwarding, adding the Tailscale apt repo, installing the daemon, and running `tailscale up` with the auth key and subnet advertisement.

Tailscale was chosen over self-hosted VPN solutions for its simplicity — no server to maintain, WireGuard under the hood, free tier covers homelab scale.

## What's next

- SOPS for secrets management (credentials are currently plaintext in values)
- More RAM on the host to allow worker memory expansion
