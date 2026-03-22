# Platform Context

## Goals

- Run multiple long-lived side projects on Kubernetes with a stable baseline.
- Prefer GitOps workflows (Git is source of truth; cluster reconciles from Git).
- Keep operational overhead low and recovery procedures clear.
- Improve availability within the limits of 3 nodes.
- No auto-apply of upgrades; all version bumps go through PRs.

## Constraints

- Provider: Strato vServers (VC). No provider floating IP / ClusterIP feature.
- Node count: 3 → realistic availability target is "tolerate 1 node failure".
- One public IPv4 + IPv6 per server; no shared/floating IP.

## Current stack

- Kubernetes distribution: k3s (version-pinned via system-upgrade-controller Plan CR)
- Target topology: 3x server nodes (embedded etcd quorum), 1 heavy + 2 light
- OS: Debian 13 (Trixie)
- Storage: Longhorn (PVC provider)
- Backups: Longhorn backups to Cloudflare R2 (S3-compatible)
- Database: CloudNativePG (per-app Postgres clusters, backed up to R2)
- Auth: Keycloak (Postgres via CNPG)
- GitOps: FluxCD v2
- Ingress: Traefik (managed via FluxCD; k3s packaged Traefik disabled)
- Certificates: cert-manager (DNS-01 challenge via Cloudflare)
- DNS: Cloudflare DNS (DNS-only; no Tunnel); ExternalDNS manages app records
- Network: Flannel + WireGuard (encrypted inter-node traffic)
- Secret management: SOPS + age (FluxCD native decryption)
- Monitoring: kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
- Uptime / status page: OneUptime
- Updates: system-upgrade-controller (k3s version pinning); Renovate Bot (PRs for Helm charts, container images, k3s version — automerge disabled)
- Notifications: Flux notification-controller → Discord webhook
- CI: GitHub Actions
- Registry: GHCR when we build our own images

## k3s install flags

- `--disable traefik,servicelb,local-storage`
- `--disable-cloud-controller`
- `--disable-helm-controller` (FluxCD manages all Helm releases)
- `--flannel-backend wireguard-native`
- `--kube-apiserver-arg default-not-ready-toleration-seconds=15`
- `--kube-apiserver-arg default-unreachable-toleration-seconds=15`
- metrics-server: kept (k3s built-in, needed for `kubectl top` + HPA)

## Node roles

- Heavy node: labeled `node-role.bach-industries/heavy=true` — runs resource-intensive workloads (Postgres, OneUptime, Prometheus)
- Light nodes (×2): labeled `node-role.bach-industries/light=true` — run lighter services (Discord bot, Stirling-PDF, etc.)
- All 3 are server nodes (etcd members) for HA

## Working agreements

- Small, reversible changes.
- No auto-apply: Renovate opens PRs, humans merge. FluxCD only reconciles what's on `main`.
- Critical components get:
  - ADR (why)
  - Runbook (how to fix)
  - Backup/restore drill instructions
