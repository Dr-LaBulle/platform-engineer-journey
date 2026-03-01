# Sovereign Hybrid Infra Showcase — Lab Hybride (Dépôt Vitrine)
**DevOps • Cloud Engineering • DevSecOps • Platform Engineering**

**Owner :** @Dr-LaBulle  
**Dernière mise à jour :** 2026-03-01  
**Version anglaise :** [`README.md`](README.md)

Ce dépôt est une **vitrine** qui documente et automatise un lab hybride réaliste, pensé pour :
- l’**auto-hébergement** et la **vie privée**
- le **contournement CGNAT** via un VPS public + **WireGuard**
- la **segmentation réseau** (VLANs UniFi) + sécurité-by-design
- un parcours conteneurs progressif : **Docker Compose → Docker Swarm → Kubernetes**
- l’**IaC** (Terraform), la **gestion des secrets** (Vault), l’**automatisation** (Ansible)
- l’**observabilité** (LGTM) + un **SIEM** (Wazuh)
- le **hardening Linux** aligné **RHCSA/RHCE** et la sécurité K8s **CKA/CKS**

> Note sécurité : les identifiants sensibles (nom de domaine réel, hostnames, plan IP, endpoints publics) sont volontairement omis.

---

## Éthique & principes (Pourquoi “Sovereign”)

Ce projet est technique, mais il porte aussi des choix d’ingénierie :

- **Souveraineté & privacy-first :** privilégier l’auto-hébergement et des flux de données maîtrisés.
- **Agnostique / vendor-neutral :** prioriser standards ouverts et outils portables (WireGuard, IaC, conteneurs, Kubernetes).
- **Technos agnostiques :** focus sur les concepts et patterns, pas sur un cloud vendor unique.
- **Portabilité comme contrainte :** documenter les migrations (Compose → Swarm → Kubernetes) et limiter le lock-in.
- **Security-by-design :** segmentation, moindre privilège, auditabilité (Wazuh), baselines durcies.

---

## Sommaire
- [1. À quoi sert ce dépôt](#1-à-quoi-sert-ce-dépôt)
- [2. Objectifs & principes](#2-objectifs--principes)
- [3. Architecture globale](#3-architecture-globale)
- [4. Matériel & plateformes](#4-matériel--plateformes)
- [5. Topologie réseau (VLANs & zones de confiance)](#5-topologie-réseau-vlans--zones-de-confiance)
- [6. Connectivité & exposition (contournement CGNAT)](#6-connectivité--exposition-contournement-cgnat)
- [7. Compute & orchestration : Compose → Swarm → Kubernetes](#7-compute--orchestration--compose--swarm--kubernetes)
- [8. Services socle (platform)](#8-services-socle-platform)
- [9. Observabilité (LGTM)](#9-observabilité-lgtm)
- [10. Supervision sécurité (SIEM Wazuh)](#10-supervision-sécurité-siem-wazuh)
- [11. Backups & DR : ZFS + PBS + backups applicatifs](#11-backups--dr--zfs--pbs--backups-applicatifs)
- [12. Infrastructure as Code (Terraform)](#12-infrastructure-as-code-terraform)
- [13. Gestion des secrets (HashiCorp Vault)](#13-gestion-des-secrets-hashicorp-vault)
- [14. Automatisation (Ansible) & standardisation](#14-automatisation-ansible--standardisation)
- [15. Parcours hardening (RHCSA/RHCE) & sécurité K8s (CKS)](#15-parcours-hardening-rhcsarhce--sécurité-k8s-cks)
- [16. Schéma du dépôt (repository tree)](#16-schéma-du-dépôt-repository-tree)
- [17. Tableaux de suivi (certifs & livrables)](#17-tableaux-de-suivi-certifs--livrables)
- [18. Notes sécurité](#18-notes-sécurité)
- [Licence](#licence)

---

## 1. À quoi sert ce dépôt

Ce dépôt est une **vitrine** qui devient progressivement :
- un **hub de documentation** (archi, réseau, décisions, runbooks)
- un **hub d’automatisation** (Terraform/Ansible, manifests de déploiement)
- un **portfolio** aligné avec la roadmap (DCA, Terraform, Vault, CKA, RHCSA/RHCE, CKS)

Lisible pour :
- recruteurs / pairs (vue d’ensemble + arbitrages)
- moi-même dans 6 mois (runbooks + reproductibilité)

---

## 2. Objectifs & principes

### Objectifs techniques
- Design hybride stable (VPS public + lab privé)
- Provisioning reproductible (IaC)
- Segmentation et cloisonnement (VLANs + firewall)
- Backups fiables (ZFS + PBS + applicatif)
- Observabilité + visibilité sécurité (SIEM)

### Principes d’exploitation
- **Deny-by-default** entre zones
- **Surface d’attaque minimale** (un seul point d’entrée public : le VPS)
- Discipline “infra as code” + décisions documentées
- Une sauvegarde n’existe vraiment que si la restauration est testée

---

## 3. Architecture globale

### Passerelle publique (Suisse)
- **VPS :** Infomaniak “VPS Lite”
- **OS :** Debian (minimal, approche “appliance”)
- **Disponibilité :** 24/7
- Rôles : point d’entrée public, endpoint WireGuard, reverse proxy, contrôles sécurité

### Homelab privé (Portugal)
- **Plateforme :** Proxmox VE
- **Stockage :** ZFS RAID-Z2
- **Réseau :** UniFi UDM + switch managé + VLANs
- **Workloads :** Docker Compose (d’abord), puis Swarm, puis Kubernetes

---

## 4. Matériel & plateformes

### On-Prem (Portugal)
- **Serveur :** Dell PowerEdge T430 (2× CPU, 128 Go RAM, 12× disques)
- **Virtualisation :** Proxmox VE
- **Stockage :** ZFS RAID-Z2 (~10 To selon layout)

### Public (Suisse)
- VPS Debian avec IP publique (rôle passerelle)

---

## 5. Topologie réseau (VLANs & zones de confiance)

| VLAN | Zone    | Usage | Exemples d’assets | Exposition |
|------|---------|-------|-------------------|------------|
| 10   | MGMT    | Plan admin | Proxmox UI, iDRAC, contrôleurs UniFi | Privé uniquement |
| 20   | SERVERS | Plan compute | Nœuds Compose/Swarm/K8s, workers | Privé uniquement |
| 30   | DMZ     | Services publiés | Apps reverse-proxy | Via VPS uniquement |
| 40   | INFRA   | Services sensibles | Vault, PBS, Wazuh, données critiques | Privé uniquement |

**Baseline**
- Inter-VLAN : **deny-by-default**
- MGMT accessible via VPN/bastion uniquement
- DMZ pas d’accès MGMT ; accès INFRA minimal et explicite si nécessaire

---

## 6. Connectivité & exposition (contournement CGNAT)

**Problème :** CGNAT Starlink empêche l’inbound direct.  
**Solution :** le VPS public est l’unique point d’entrée.

### Chemin par défaut
1. Internet → IP publique du VPS
2. **Traefik** route le HTTPS
3. Backends joints via **WireGuard** vers le homelab

### DNS & TLS (ClouDNS)
- Domaine géré chez **ClouDNS** (détails génériques dans ce repo)
- TLS via Let’s Encrypt (préférence **DNS-01**)
- DNS public limité aux services exposés ; endpoints internes restent privés

---

## 7. Compute & orchestration : Compose → Swarm → Kubernetes

### Phase 0 — Docker Compose (baseline single-node)
Pourquoi commencer par Compose :
- itération plus rapide, debug plus simple
- validation des volumes/réseaux/labels Traefik/TLS/monitoring
- production d’un MVP propre avant Swarm

Livrables :
- `.env` cohérent (sans secrets commités)
- volumes nommés et persistance documentée
- runbooks deploy/upgrade/rollback

### Phase 1 — Docker Swarm (prochaine étape)
Objectifs :
- scheduling, rolling updates
- réseaux overlay
- configs/secrets Swarm
- patterns multi-nœuds

Livrables :
- runbook bootstrap Swarm
- notes de migration (Compose → Swarm)
- services DMZ publiés via Traefik + WireGuard

### Phase 2 — Kubernetes (prévu)
Objectifs (CKA/CKS) :
- ingress, storage, networking
- RBAC, policies, security context
- upgrades, maintenance nodes
- hardening + audit

Approche :
- apps portables (stateless si possible)
- persistance + backup/restore par application
- documentation migration service par service

---

## 8. Services socle (platform)

| Catégorie | Composant | Rôle |
|----------|-----------|------|
| Ingress | Traefik | TLS, routage, middleware, rate limits |
| Tunnel | WireGuard | Connectivité site-à-site sécurisée |
| Observabilité | Grafana + Prometheus + Loki + Promtail | Métriques + logs + dashboards |
| SIEM | Wazuh | Supervision sécurité & audit |
| Secrets (prévu) | HashiCorp Vault | Secrets centralisés |
| IaC | Terraform | Provisioning reproductible |
| Automation (prévu) | Ansible | Baseline OS + hardening + anti-drift |
| Backups | ZFS + PBS + applicatif | Backups en couches |

---

## 9. Observabilité (LGTM)

Objectif : visibilité ops (dispo, perf, capacité).
- **Métriques :** Prometheus (node-exporter, cAdvisor, exporters)
- **Logs :** Loki + Promtail
- **Dashboards :** Grafana

Livrables :
- dashboards standardisés
- alerting de base (ressources, santé services, santé tunnel)

---

## 10. Supervision sécurité (SIEM Wazuh)

Objectif : télémétrie sécurité + audit.
- Wazuh manager/dashboard en **INFRA**
- agents sur VPS + Proxmox + nœuds

Livrables :
- détection brute-force SSH / anomalies d’auth
- FIM sur chemins sensibles
- alerting contrôlé (email/webhook)

---

## 11. Backups & DR : ZFS + PBS + backups applicatifs

1. **Snapshots ZFS** (rollback rapide)
2. **Proxmox Backup Server (PBS)** (VM/CT incrémental + dédup)
3. **Backups applicatifs**
   - dumps DB (ex : Postgres)
   - export configs (Traefik, manifests Compose/Swarm/K8s, policies Vault)
   - tests réguliers de restauration + runbooks

---

## 12. Infrastructure as Code (Terraform)

Terraform rend l’infra reproductible et reviewable :
- provisioning VPS
- DNS (si pertinent / sanitisé)
- Proxmox (optionnel)

Livrables :
- modules documentés
- `terraform fmt/validate` OK
- séparation environnements

---

## 13. Gestion des secrets (HashiCorp Vault)

Vault comme brique centrale :
- secrets apps, creds automation/CI
- policies + audit logs

Livrables :
- procédure init/unseal (documentée)
- policies + auth method (AppRole, etc.)
- audit logs exploitables (obs + SIEM)

---

## 14. Automatisation (Ansible) & standardisation

Ansible pour baseline + hardening + anti-drift.
Livrables :
- rôles (`baseline`, `hardening`, `docker`, `kubernetes-node`, `monitoring-agent`)
- inventaires par zone (VLAN)
- idempotence prouvée

---

## 15. Parcours hardening (RHCSA/RHCE) & sécurité K8s (CKS)

- RHCSA : fondamentaux Linux (users, systemd, storage, réseau, firewall, SELinux)
- RHCE : industrialisation via Ansible
- CKS : hardening cluster, RBAC, network policies, supply-chain (plus tard)

Livrables :
- checklists hardening par rôle
- preuves avant/après (configs, audits, alertes Wazuh)

---

## 16. Schéma du dépôt (repository tree)

> Structure cible (évolutive). Certains dossiers peuvent être vides au début.

```text
.
├── README.md
├── README_FR.md
├── platform/                         # Vue "infra globale" (architecture/runbooks)
│   ├── architecture/
│   │   ├── diagrams/                 # schémas (draw.io, png, svg)
│   │   ├── adr/                      # décisions d'architecture (ADR)
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
├── tech/                             # Vue "par techno" (artefacts + docs)
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
├── certs/                            # Parcours certifs + labs + notes
│   ├── DCA/
│   ├── Terraform-Associate/
│   ├── Vault-Associate/
│   ├── CKA/
│   ├── RHCSA/
│   ├── RHCE/
│   └── CKS/
└── .github/
    └── workflows/                    # CI (lint, validate, checks docs, etc.)
```

---

## 17. Tableaux de suivi (certifs & livrables)

### 17.1 Roadmap certifications (2026–2028)
| Période | Axe | Objectif | Statut | Preuve |
|--------|-----|----------|--------|--------|
| T1 2026 | Docker | DCA | ⬜ | `certs/DCA/` |
| T2 2026 | IaC | Terraform Associate | ⬜ | `certs/Terraform-Associate/` |
| T3 2026 | Secrets | Vault Associate | ⬜ | `certs/Vault-Associate/` |
| T4 2026–T1 2027 | Kubernetes | CKA | ⬜ | `certs/CKA/` |
| 2027–2028 | Linux | RHCSA / RHCE | ⬜ | `certs/RHCSA/` + `certs/RHCE/` |
| T2–T3 2028 | K8s Security | CKS | ⬜ | `certs/CKS/` |

Légende : ⬜ prévu • 🟨 en cours • ✅ fait

### 17.2 Suivi livrables du lab
| Domaine | Livrable | Date cible | Statut | Notes |
|--------|----------|-----------|--------|------|
| Réseau | VLANs (10/20/30/40) | 2026-04 | ⬜ | règles UniFi doc |
| Réseau | WireGuard site-à-site stable | 2026-04 | ⬜ | auto-reconnect |
| Ingress | Traefik sur VPS + TLS | 2026-04 | ⬜ | ACME DNS-01 |
| Conteneurs | **Compose baseline (single-node)** | 2026-03/04 | ⬜ | MVP stacks |
| Conteneurs | **Swarm bootstrap + migrate stacks** | 2026-04/05 | ⬜ | overlay + secrets |
| Obs | LGTM dashboards baseline | 2026-04 | ⬜ | alerting base |
| SIEM | Wazuh manager + agents | 2026-05 | ⬜ | tuning règles |
| Backups | PBS + 1er restore test | 2026-05 | ⬜ | rapport restore |
| Secrets | Vault MVP | 2026-07 | ⬜ | policies + audit |
| Automation | Ansible baseline role | 2026-08 | ⬜ | idempotent |
| K8s | 1er cluster K8s (lab) | 2026-12 | ⬜ | pratique CKA |

---

## 18. Notes sécurité
- Pas d’identifiants sensibles dans ce repo (pas de hostnames/IP/secrets réels)
- Plan admin privé (VPN/tunnel)
- Deny-by-default entre VLANs
- Backups chiffrés quand nécessaire
- SIEM (Wazuh) pour construire la visibilité sécurité tôt

---

## Licence
À définir (MIT / Apache-2.0). Ajouter un fichier `LICENSE` quand prêt.