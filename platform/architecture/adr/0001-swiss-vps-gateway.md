# ADR 0001 : Choix d'une passerelle VPS en Suisse (Infomaniak)

## État
**Accepté**

## Contexte
Le laboratoire principal est situé au Portugal derrière une connexion **5G** (routeur opérateur).  
Dans ce contexte, l’accès Internet est généralement fourni via **CGNAT** (Carrier-Grade NAT), ce qui empêche l’exposition directe de services sur Internet (absence d’IP publique routable en entrée, redirections de ports non fiables/impossibles).

Le projet étant axé sur la **souveraineté** et l’agnosticisme technologique, la **j uridiction** et la **nature du fournisseur** du point d’entrée public sont des critères importants.

## Décision
Utiliser un **VPS Lite chez Infomaniak (Suisse)** comme passerelle d’entrée unique (Gateway) exposée sur Internet.

Relier cette passerelle au laboratoire privé via un tunnel **WireGuard** persistant (site-to-site).  
Les services publics seront exposés via le VPS (reverse proxy / TLS), puis routés vers le lab via le tunnel.

## Critères de décision
- **IP publique** disponible sur la passerelle (nécessaire à l’exposition de services).
- **Budget** : ≤ **5 €/mois**.
- **Souveraineté** : fournisseur indépendant, réduction du recours aux GAFAM pour le “premier saut”.
- **Portabilité** : design reproductible et migrable (le VPS doit pouvoir être reprovisionné/migré).
- **Latence** : acceptable pour l’usage (axe Suisse ↔ Portugal).

## Options considérées
1. **Exposition directe depuis le lab (port forwarding)**  
   Rejetée : CGNAT fréquent sur les accès 5G, pas d’IP publique entrante stable.
2. **Tunnels managés (ex : Cloudflare Tunnel)**  
   Rejetés : dépendance à un tiers central, moins aligné avec l’objectif de souveraineté.
3. **VPS dans un autre pays de l’UE**  
   Non retenu : préférence pour la Suisse dans le cadre de la stratégie de souveraineté/juridiction (tout en restant migrable).
4. **Solution overlay type Tailscale / Headscale**  
   Possibles selon les cas, mais ne répondent pas directement au besoin “point d’entrée public unique + reverse proxy public”.

## Justification
1. **Juridiction (Suisse)** : choix cohérent avec une approche “privacy-first” et une préférence pour un cadre juridique perçu comme favorable à la protection des données.
2. **Bypass CGNAT (5G)** : le VPS dispose d’une IP publique ; le tunnel WireGuard initié depuis le lab permet un retour de trafic vers les services internes.
3. **Souveraineté** : Infomaniak est un acteur indépendant disposant de ses propres datacenters, évitant un fournisseur hyperscaler comme point d’entrée.
4. **Budget** : l’offre VPS Lite respecte la contrainte de coût (≈ 5 €/mois).
5. **Performance** : latence acceptable sur l’axe Suisse-Portugal pour les services ciblés.

## Conséquences
### Positives
- Exposition sécurisée des services via un point d’entrée unique (reverse proxy + TLS).
- IP publique fixe côté gateway, compatible avec DNS/TLS automatisés.
- Cohérence avec la vision “privacy-first / souveraineté”.
- Architecture migrable : possibilité de changer de fournisseur de VPS si nécessaire.

### Négatives / risques
- Coût mensuel récurrent (≈ 5 €/mois).
- Dépendance à la disponibilité du tunnel WireGuard (dégradation si rupture du tunnel).
- La passerelle devient un composant critique (cible d’attaque potentielle).

### Mesures de mitigation (préconisées)
- Durcissement de la passerelle (SSH par clés uniquement, mises à jour, pare-feu deny-by-default).
- Supervision et auto-reconnexion du tunnel WireGuard (systemd + healthchecks).
- Centralisation des logs et alerting (ex : Wazuh / observabilité).