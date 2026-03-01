# Sovereign Hybrid Infra Showcase
**DevOps • Cloud Engineering • DevSecOps • Platform Engineering**

**Owner:** @Dr-LaBulle  
**Last update:** 2026-03-01  
**French version:** [`README_FR.md`](README_FR.md)

A production-inspired **hybrid homelab** built as a long-term **skills showcase**:
- **Self-hosting & privacy-first services**
- **CGNAT bypass** using a Swiss public VPS gateway + **WireGuard**
- **Network segmentation** (UniFi VLANs) + security-by-design
- Progressive container journey: **Docker Compose → Docker Swarm → Kubernetes**
- **Infrastructure as Code** (Terraform), **Secrets Management** (Vault), **Automation** (Ansible)
- **Observability** (LGTM) + **SIEM** (Wazuh)
- **Linux hardening & operations** aligned with **RHCSA/RHCE** and **CKA/CKS**

> Security note: sensitive identifiers (real domain, hostnames, IP plan, public endpoints) are intentionally omitted.

---

## Ethos & Principles (Why “Sovereign”)

This project is not only technical; it also reflects engineering choices:

- **Sovereignty & privacy-first:** prefer self-hosting and controlled data flows.
- **Vendor-neutral by design:** prioritize open standards and portable tooling (WireGuard, Terraform/IaC, containers, Kubernetes).
- **Technology-agnostic mindset:** focus on concepts and patterns, not a single cloud vendor.
- **Portability as a constraint:** document migrations (Compose → Swarm → Kubernetes) and avoid lock-in.
- **Security-by-design:** segmentation, least privilege, auditability (Wazuh), hardened baselines.

---

## Table of Contents
- [1. What this repo is](#1-what-this-repo-is)
- [2. Goals & principles](#2-goals--principles)
- [3. High-level architecture](#3-high-level-architecture)
- [4. Hardware & platforms](#4-hardware--platforms)
- [5. Network topology (VLANs & trust zones)](#5-network-topology-vlans--trust-zones)
- [6. Public exposure & connectivity (CGNAT bypass)](#6-public-exposure--connectivity-cgnat-bypass)
- [7. Compute & orchestration: Compose → Swarm → Kubernetes](#7-compute--orchestration-compose--swarm--kubernetes)
- [8. Core platform services](#8-core-platform-services)
- [9. Observability (LGTM)](#9-observability-lgtm)
- [10. Security monitoring (Wazuh SIEM)](#10-security-monitoring-wazuh-siem)
- [11. Backups & DR: ZFS + PBS + app backups](#11-backups--dr-zfs--pbs--app-backups)
- [12. Infrastructure as Code (Terraform)](#12-infrastructure-as-code-terraform)
- [13. Secrets management (HashiCorp Vault)](#13-secrets-management-hashicorp-vault)
- [14. Automation (Ansible) & standardization](#14-automation-ansible--standardization)
- [15. Hardening track (RHCSA/RHCE) + K8s security (CKS)](#15-hardening-track-rhcsarhce--k8s-security-cks)
- [16. Repository tree (repo map)](#16-repository-tree-repo-map)
- [17. Tracking tables (milestones & deliverables)](#17-tracking-tables-milestones--deliverables)
- [18. Security notes](#18-security-notes)
- [License](#license)

---

## 1. What this repo is

This is a **showcase repository** that progressively becomes:
- a **documentation hub** (architecture, network, decisions, runbooks)
- an **automation hub** (Terraform/Ansible, deployment manifests)
- a **skills portfolio** aligned with a multi-year roadmap (DCA, Terraform, Vault, CKA, RHCSA/RHCE, CKS)

It aims to be readable by:
- recruiters / peers (high-level view + trade-offs)
- myself in 6 months (runbooks + reproducibility)

---

## 2. Goals & principles

### Technical goals
- Stable **hybrid gateway design** (public VPS + private lab)
- Repeatable infra provisioning (IaC)
- Secure-by-default segmentation (VLANs + firewalling)
- Reliable backups (ZFS snapshots + PBS + app-level backups)
- Observability + security visibility

### Operating principles
- **Deny-by-default** between trust zones
- **Minimal public attack surface** (single public entrypoint: the gateway VPS)
- **Docs-first operations** (runbooks) + infra defined as code
- Backups are only real when **restore is tested**

---

## 3. High-level architecture

### Public Gateway (Switzerland)
- **VPS provider:** Infomaniak (VPS Lite)
- **OS:** Debian (kept minimal, “gateway appliance” mindset)
- **Availability:** 24/7
- Roles: public entrypoint, WireGuard endpoint, reverse proxy, security controls

### Private Homelab (Portugal)
- **Platform:** Proxmox VE
- **Storage:** ZFS RAID-Z2
- **Network:** UniFi UDM + managed switch + VLAN segmentation
- **Workloads:** Docker Compose (first), then Swarm, then Kubernetes

---

## 4. Hardware & platforms

### On-Prem (Portugal)
- **Server:** Dell PowerEdge T430 (2× CPU, 128GB RAM, 12× disks)
- **Virtualization:** Proxmox VE
- **Storage:** ZFS RAID-Z2 pool (~10TB usable depending on layout)

### Public (Switzerland)
- Debian VPS with a public IP (gateway)

---

## 5. Network topology (VLANs & trust zones)

| VLAN | Zone   | Purpose | Typical assets | Exposure |
|------|--------|---------|----------------|----------|
| 10   | MGMT   | Admin plane | Proxmox UI, iDRAC, UniFi controllers | Private only |
| 20   | SERVERS| Compute plane | Compose/Swarm/K8s nodes, workers | Private only |
| 30   | DMZ    | Published services | Reverse-proxied apps | Via VPS only |
| 40   | INFRA  | Sensitive services | Vault, PBS, Wazuh, critical data | Private only |

**Baseline**
- Inter-VLAN traffic: **deny-by-default**
- MGMT reachable only from admin access (VPN/bastion)
- DMZ has no direct access to MGMT; minimal access to INFRA if needed

---

## 6. Public exposure & connectivity (CGNAT bypass)

**Problem:** Starlink CGNAT blocks inbound connections.  
**Solution:** a Swiss VPS is the **only** public ingress.

### Default traffic path
1. Internet → VPS public IP
2. **Traefik** routes HTTPS
3. Backends reached through **WireGuard** tunnel to the homelab

### DNS & TLS (ClouDNS)
- Domain is managed with **ClouDNS** (kept generic in this repo)
- TLS via Let’s Encrypt (prefer **DNS-01** challenge)
- Public DNS only exposes intended services; internal endpoints remain private

---

## 7. Compute & orchestration: Compose → Swarm → Kubernetes

### Phase 0 — Docker Compose (single-node baseline)
Why start with Compose:
- Faster iteration and easier debugging
- Validate storage, volumes, reverse-proxy labels, TLS, monitoring
- Produce clean “MVP stacks” before Swarm complexity

Deliverables:
- Consistent `.env` usage (no secrets committed)
- Clear volumes naming + persistence documentation
- Runbooks: deploy/upgrade/rollback

### Phase 1 — Docker Swarm (next step)
Goals:
- service scheduling, rolling updates
- overlay networks
- Swarm configs/secrets
- multi-node operational patterns

Deliverables:
- Swarm bootstrap runbook
- migration notes (Compose → Swarm)
- published DMZ services through Traefik + WireGuard

### Phase 2 — Kubernetes (planned)
Goals (CKA/CKS-aligned):
- ingress, storage, networking
- RBAC, policies, security context
- upgrades, node maintenance
- security hardening and audit

Migration approach:
- keep apps portable (stateless where possible)
- explicit persistence + backup/restore per app
- document each service migration path

---

## 8. Core platform services

| Category | Component | Purpose |
|---------|-----------|---------|
| Ingress | Traefik | TLS termination, routing, middleware, rate limits |
| Tunnel | WireGuard | Site-to-site secure connectivity |
| Observability | Grafana + Prometheus + Loki + Promtail | Metrics + logs + dashboards |
| SIEM | Wazuh | Security monitoring and auditing |
| Secrets (planned) | HashiCorp Vault | Central secrets management |
| IaC | Terraform | Reproducible provisioning |
| Automation (planned) | Ansible | OS baseline & hardening, drift control |
| Backups | ZFS + PBS + app backups | Layered backup strategy |

---

## 9. Observability (LGTM)

Goal: **DevOps observability** (availability, performance, capacity).

- **Metrics:** Prometheus (node-exporter, cAdvisor, service exporters)
- **Logs:** Loki + Promtail
- **Dashboards:** Grafana

Deliverables:
- Standard dashboards (hosts, containers, reverse proxy)
- Baseline alerting (resources, service health, tunnel health)

---

## 10. Security monitoring (Wazuh SIEM)

Goal: **security telemetry** and audit trail.

- Wazuh manager/dashboard in **INFRA**
- Agents on VPS + Proxmox + nodes

Deliverables:
- SSH/auth anomaly detection on the VPS
- FIM on sensitive config paths
- Alerts routed to controlled channels (email/webhook)

---

## 11. Backups & DR: ZFS + PBS + app backups

Strategy: layered backups aligned with restore needs.

1. **ZFS snapshots** (fast rollback)
2. **Proxmox Backup Server (PBS)** (VM/CT backups, incremental + dedup)
3. **Application-level backups**
   - DB dumps (e.g., Postgres for Matrix/Synapse)
   - config exports (Traefik, Compose/Swarm/K8s manifests, Vault policies)
   - periodic restore tests (documented runbooks)

> Backups are only real once restore is tested.

---

## 12. Infrastructure as Code (Terraform)

Terraform is used to make the infra reproducible and reviewable.

Planned scope:
- VPS provisioning (baseline packages/services, firewall baseline)
- DNS records (where appropriate and sanitized)
- Proxmox resources (optional, depending on provider usage)

Deliverables:
- `terraform fmt/validate` passing
- clear module boundaries (vps / dns / proxmox)
- environment separation (lab/prod-like)

---

## 13. Secrets management (HashiCorp Vault)

Vault is planned as the central secrets system for:
- app secrets (tokens, DB creds)
- automation/CI credentials
- dynamic secrets where possible

Deliverables:
- init/unseal procedure + key handling documented
- policies + auth method (e.g., AppRole)
- audit logs integrated with logging/SIEM

---

## 14. Automation (Ansible) & standardization

Ansible will enforce:
- OS baseline packages and configuration
- SSH hardening
- firewall baseline
- CIS-inspired checks where relevant
- idempotent operations and drift control

Deliverables:
- roles: `baseline`, `hardening`, `docker`, `kubernetes-node`, `monitoring-agent`
- inventories per VLAN/zone
- evidence: “2nd run = 0 changes”

---

## 15. Hardening track (RHCSA/RHCE) + K8s security (CKS)

This lab is a practice field for:
- **RHCSA:** users/groups, permissions, systemd, storage, networking, firewall, SELinux basics
- **RHCE:** Ansible automation and standardization
- **CKS:** cluster hardening, RBAC, network policies, supply-chain controls (later)

Deliverables:
- hardening checklists per host role
- before/after evidence (configs, audit output, Wazuh alerts)

---

## 16. Repository tree (repo map)

> Target structure (evolves over time). Some folders may be empty initially.

```text
.
├── README.md
├── README_FR.md
├── platform/                         # "Full infrastructure" view (architecture/runbooks)
│   ├── architecture/
│   │   ├── diagrams/                 # diagrams (draw.io, png, svg)
│   │   ├── adr/                      # Architecture Decision Records (ADRs)
│   │   └── overview.md
│   ├── network/
│   │   ├── vlans.md
│   │   ├── firewall-policy.md
│   │   └── wireguard.md
│   ├── runbooks/
│   │   ├── bootstrap.md
│   │   ├── incident-response.md
│   │   └── restore-tests.md
│   ├── backups/
│   │   ├── zfs.md
│   │   ├── pbs.md
│   │   └── app-backups.md
│   └── security/
│       ├── threat-model.md
│       ├── wazuh.md
│       └── hardening-baseline.md
├── tech/                             # "By technology" view (artifacts + docs)
│   ├── docker-compose/
│   │   ├── stacks/
│   │   └── docs/
│   ├── docker-swarm/
│   │   ├── stacks/
│   │   └── docs/
│   ├── kubernetes/
│   │   ├── manifests/
│   │   ├── helm/
│   │   └── docs/
│   ├── terraform/
│   │   ├── modules/
│   │   ├── envs/
│   │   └── docs/
│   ├── vault/
│   │   ├── policies/
│   │   ├── auth-methods/
│   │   └── docs/
│   ├── ansible/
│   │   ├── inventories/
│   │   ├── roles/
│   │   └── playbooks/
│   └── observability/
│       ├── grafana/
│       ├── prometheus/
│       └── loki/
├── certs/                            # Certification tracks + labs + notes
│   ├── DCA/
│   ├── Terraform-Associate/
│   ├── Vault-Associate/
│   ├── CKA/
│   ├── RHCSA/
│   ├── RHCE/
│   └── CKS/
└── .github/
    └── workflows/                    # CI (lint, validate, docs checks, etc.)
```

---

## 17. Tracking tables (milestones & deliverables)

### 17.1 Certification roadmap (2026–2028)
| Timeframe | Track | Target | Status | Evidence |
|----------|-------|--------|--------|----------|
| Q1 2026 | Docker | DCA | ⬜ | `certs/DCA/` |
| Q2 2026 | IaC | Terraform Associate | ⬜ | `certs/Terraform-Associate/` |
| Q3 2026 | Secrets | Vault Associate | ⬜ | `certs/Vault-Associate/` |
| Q4 2026–Q1 2027 | Kubernetes | CKA | ⬜ | `certs/CKA/` |
| 2027–2028 | Linux | RHCSA / RHCE | ⬜ | `certs/RHCSA/` + `certs/RHCE/` |
| Q2–Q3 2028 | K8s Security | CKS | ⬜ | `certs/CKS/` |

Legend: ⬜ planned • 🟨 in progress • ✅ done

### 17.2 Lab deliverables tracker
| Domain | Deliverable | Target date | Status | Notes |
|--------|------------|------------|--------|------|
| Network | VLANs (10/20/30/40) implemented | 2026-04 | ⬜ | UniFi rules documented |
| Network | WireGuard site-to-site stable | 2026-04 | ⬜ | auto-reconnect |
| Ingress | Traefik on VPS with TLS | 2026-04 | ⬜ | ACME DNS-01 |
| Containers | **Compose baseline (single-node)** | 2026-03/04 | ⬜ | MVP stacks |
| Containers | **Swarm bootstrap + migrate stacks** | 2026-04/05 | ⬜ | overlay + secrets |
| Observability | LGTM baseline dashboards + alerts | 2026-04 | ⬜ | baseline rules |
| SIEM | Wazuh manager + agents | 2026-05 | ⬜ | tuning rules |
| Backups | PBS deployed + first restore test | 2026-05 | ⬜ | restore report |
| Secrets | Vault MVP | 2026-07 | ⬜ | policies + audit |
| Automation | Ansible baseline role | 2026-08 | ⬜ | idempotent |
| Kubernetes | First K8s cluster (lab) | 2026-12 | ⬜ | CKA practice |

---

## 18. Security notes
- No sensitive identifiers in this repo (no real hostnames/IPs/secrets)
- Admin plane stays private (VPN/tunnel-first)
- Deny-by-default between VLANs
- Backups are encrypted where appropriate
- SIEM (Wazuh) used to build security visibility early

---

## License
TBD (MIT / Apache-2.0). Add a `LICENSE` file when ready.