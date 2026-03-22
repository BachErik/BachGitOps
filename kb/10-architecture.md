# Architecture

## High-level

- 3-node k3s cluster (1 heavy, 2 light — all server nodes with embedded etcd)
- GitOps (FluxCD v2) manages cluster state from a monorepo
- Longhorn provides persistent volumes and backups to Cloudflare R2
- CloudNativePG provides per-app Postgres clusters (backed up to R2)
- Traefik provides ingress (80/443) on all nodes via hostPort
- cert-manager issues TLS certificates (DNS-01 via Cloudflare API)
- Cloudflare provides DNS (DNS-only; A + AAAA for apps)
- ExternalDNS manages Cloudflare DNS records from Kubernetes resources
- Flannel + WireGuard encrypts inter-node traffic
- kube-prometheus-stack provides internal monitoring (Prometheus + Grafana + Alertmanager)
- OneUptime provides uptime monitoring + public status pages
- Keycloak provides authentication (Postgres via CNPG)

## Control plane topology

### Target

- 3x k3s server nodes (embedded etcd)
- Servers also run workloads (no dedicated workers)
- 1 heavy node (`node-role.bach-industries/heavy=true`): Postgres, OneUptime, Prometheus
- 2 light nodes (`node-role.bach-industries/light=true`): Discord bot, Stirling-PDF, website, etc.

### Availability implication

- One node can fail and cluster should continue operating.
- Two-node failure is not a "normal operation" scenario due to quorum.

## k3s API endpoint

- No floating IP available. Each node has its own public IPv4 + IPv6.
- DNS names only as TLS SANs — no raw IPs. This keeps certs portable across providers; move servers → update DNS records → done, no cert regeneration.
- DNS records (managed **outside** ExternalDNS to avoid circular dependency):
  - `k3s.example.com` → A/AAAA round-robin to all 3 nodes (primary API endpoint)
  - `k3s-1.example.com` → A/AAAA to node 1 (targeted access)
  - `k3s-2.example.com` → A/AAAA to node 2
  - `k3s-3.example.com` → A/AAAA to node 3
- All 4 DNS names are passed as `--tls-san` during install.
- kubeconfig points to `k3s.example.com:6443`.

## Ingress traffic pattern

- Cloudflare DNS publishes multiple A/AAAA targets (round robin) pointing at node public IPs.
- Traefik is deployed on the cluster via FluxCD and listens on each node (hostPort 80/443).
- Requests can land on any node and be routed to workloads.
- Note: DNS round robin is not a health-checked load balancer; failure handling depends on client retry + DNS caching/TTL.

## TLS strategy

- cert-manager with DNS-01 challenge via Cloudflare API token.
- ClusterIssuers: `letsencrypt-staging` and `letsencrypt-prod`.
- DNS-01 is preferred over HTTP-01 because:
  - Supports wildcard certificates.
  - Does not require port 80 to be reachable during issuance.
  - Works regardless of Traefik's state.

## OneUptime exposure pattern

- OneUptime is exposed through Traefik Ingress (Traefik remains the single edge entrypoint).
- OneUptime's bundled nginx gateway runs behind a ClusterIP service (no dedicated public IP / no LoadBalancer service).
- Hostnames:
  - oneuptime.bacherik.de → OneUptime
  - status.bacherik.de → OneUptime (status pages via hostname routing)
- TLS:
  - cert-manager issues certificates for oneuptime.bacherik.de and status.bacherik.de via DNS-01.

## IPv6 approach (phase 1)

- Edge dual-stack only:
  - publish A + AAAA records for app hostnames
  - Traefik accepts IPv4/IPv6 on nodes and routes to pods (pods can remain IPv4-only initially)

## Disabled k3s components

- `traefik` — deployed via FluxCD for full version/config control
- `servicelb` — using hostPort instead (no external LB available)
- `local-storage` — Longhorn replaces it
- `helm-controller` — FluxCD manages all Helm releases; avoids conflicts
- `cloud-controller` — bare metal, no cloud provider

## Version management

- **No auto-apply.** All version bumps go through PRs.
- Renovate Bot watches the monorepo for Helm chart versions, container image tags, and k3s Plan version — opens PRs with `automerge: false`.
- system-upgrade-controller (Rancher) handles k3s node upgrades via a `Plan` CR with a pinned version.
- Renovate Dependency Dashboard (persistent GitHub Issue) provides an overview of all pending updates.
- Flux notification-controller sends reconciliation alerts to Discord.

## GitOps layout (monorepo, 3-tier dependency chain)

- `clusters/prod/` holds Flux Kustomization resources that orchestrate the tiers:
  - `helm-repositories/` — shared HelmRepository sources
  - `controllers/` — Tier 1: operators & CRD providers (cert-manager, Traefik, ExternalDNS, Longhorn, CNPG, system-upgrade-controller, kube-prometheus-stack)
  - `configs/` — Tier 2: CRs that depend on controllers (ClusterIssuers, Traefik middlewares, Longhorn backup targets, StorageClasses)
  - `upgrades/` — k3s version Plan CR (depends on controllers)
  - `notifications/` — Flux alert provider + alerts to Discord (depends on controllers)
  - `apps/` — Tier 3: workloads (Keycloak, OneUptime, Stirling-PDF, website, Discord bot, monitoring dashboards)
- Dependency order: `helm-repositories` → `controllers` → `configs` → `apps`
- `kb/` contains human documentation + ADRs.
- `renovate.json` at repo root configures Renovate Bot.
