# WireGuard — Tunnel site-to-site (VPS Gateway ⇄ Lab)

## Objet
Définir le contrat réseau du tunnel **WireGuard** entre :
- un **VPS public** (Suisse) servant d’ingress unique,
- un **homelab privé** (Portugal) derrière une connexion **5G** (CGNAT probable),

en limitant strictement le périm��tre exposé.

> Note “publication” : les valeurs d’adressage (CIDR privés) et le port WireGuard indiqués ci-dessous sont considérés **non sensibles** dans le cadre de ce showcase.  
> Ils peuvent être remplacés par des placeholders si nécessaire, tant que le contrat (scope, routage, flux autorisés) reste identique.

---

## Références
- Décision d’architecture : **ADR 0002** — terminaison WireGuard côté lab sur Linux/CT  
  `platform/architecture/adr/0002-wireguard-termination-lab-linux-container.md`

---

## Décisions (valeurs réelles)

### Réseaux
- DMZ (VLAN 30) : `192.168.30.0/24`
- Hôte tunnel côté lab (`wg-lab`) : `192.168.30.2` (IP fixe en DMZ)

### Transit WireGuard (point-à-point)
- Réseau transit : `10.100.0.0/30`
- VPS (wg0) : `10.100.0.1/30`
- `wg-lab` (wg0) : `10.100.0.2/30`

### Port public WireGuard (VPS)
- WireGuard : `51820/udp`

### Scope (MVP)
- **Option D=2 (strict)** : le VPS peut atteindre **uniquement** l’hôte `wg-lab` (`192.168.30.2`) via le tunnel.
- Aucun routage vers `192.168.30.0/24` (DMZ complète) n’est activé au MVP.

---

## Topologie et rôles

### VPS (Suisse)
- Endpoint WireGuard public (écoute sur `51820/udp`).
- Reverse proxy/TLS pour les services Internet (ex. Traefik) — séparé du tunnel.
- Pare-feu strict (ports minimum).

### Lab (Portugal)
- UDM Pro : segmentation inter-VLAN + firewall (deny-by-default).
- Proxmox : héberge un CT LXC Debian dédié `wg-lab`.
- `wg-lab` : endpoint WireGuard côté lab.

> Note : `wg-lab` est un **CT LXC** ; WireGuard utilise le **kernel de l’hôte Proxmox** (module WireGuard côté host).  
> L’objectif est de conserver une configuration WireGuard “Linux standard” (fichiers, systemd, logs) tout en restant léger.

---

## Mode de fonctionnement (important sur 5G)
- Le **VPS écoute**, le lab **initie** la connexion.
- `wg-lab` doit configurer `PersistentKeepalive = 25` vers le VPS pour maintenir le NAT 5G ouvert.

---

## Routage et `AllowedIPs` (contractuel)

### Principe
WireGuard route ce qui est listé dans `AllowedIPs` (c’est le “contrat” de routage).

### MVP (D=2 : un seul hôte)
- Côté VPS, pour le peer `wg-lab` : `AllowedIPs` doit inclure :
  - `10.100.0.2/32` (IP WireGuard du peer)
  - `192.168.30.2/32` (**uniquement** l’hôte `wg-lab` en DMZ)

- Côté `wg-lab`, pour le peer VPS : `AllowedIPs` doit inclure :
  - `10.100.0.1/32` (IP WireGuard du VPS)

> Note : on n’annonce **pas** `192.168.30.0/24` au stade MVP.

---

## Sécurité (contrat)

### Surface d’exposition
- Internet → VPS uniquement.
- Aucun port entrant direct vers le lab (5G/CGNAT + politique de sécurité).

### Pare-feu VPS (intention)
- Autoriser : `51820/udp`, `80/tcp`, `443/tcp`
- SSH : restreint (IP allowlist ou accès via tunnel), clés uniquement

### Pare-feu UDM Pro (intention)
- Inter-VLAN : deny-by-default
- Pas de route ni de règle donnant accès à MGMT/INFRA via le tunnel

---

## Tests d’acceptation (DoD)
- `wg show` sur VPS et `wg-lab` affiche un handshake récent.
- Depuis le VPS : ping/port-check vers `10.100.0.2` et `192.168.30.2`.
- Depuis le VPS : **impossible** d’atteindre une autre IP DMZ (ex. `192.168.30.10`) tant que le scope D=2 est actif.