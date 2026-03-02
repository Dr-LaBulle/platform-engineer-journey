# ADR 0003 : Traefik (edge) — certificats TLS via ACME DNS-01 (ClouDNS)

## État
**Accepté** (2026-03-02)

## Contexte
L’infrastructure expose des services applicatifs sur Internet via une passerelle publique (VPS en Suisse). Les workloads applicatifs (par exemple Matrix/Synapse) sont exécutés dans le lab et sont joints depuis le VPS via un tunnel WireGuard.

L’émission et le renouvellement des certificats TLS doivent être automatisés, avec les objectifs suivants :
- Centraliser la terminaison TLS sur le VPS (reverse proxy “edge”).
- Réduire la dépendance à l’ouverture du port `80/tcp` pour les challenges ACME.
- Permettre l’émission future de certificats wildcard (ex : `*.cozy-flaow.cloudns.org`).
- Conserver une approche reproductible et auditable (“infra-as-code”).

Le domaine public est géré via ClouDNS (`cozy-flaow.cloudns.org`), et les noms de services publics incluent `matrix.cozy-flaow.cloudns.org`.

## Décision
Mettre en place Traefik sur le VPS en tant que reverse proxy edge, avec émission/renouvellement automatique des certificats Let’s Encrypt via ACME en utilisant le challenge **DNS-01** et le provider **ClouDNS**.

## Critères de décision
- **Fiabilité** : capacité à obtenir/renouveler des certificats indépendamment de l’accessibilité du port `80/tcp`.
- **Évolutivité** : capacité à gérer plusieurs services/sous-domaines, y compris des certificats wildcard.
- **Sécurité** : limitation de surface d’exposition, protection des secrets, terminaison TLS au point d’entrée public.
- **Reproductibilité** : configuration versionnable (hors secrets), exécutable via runbook et automatisable ultérieurement.
- **Portabilité** : éviter les dépendances à des composants non transférables.

## Options considérées
### Option A — ACME HTTP-01 (challenge HTTP via port 80)
**Motifs :**
- Mise en œuvre simple et ne nécessitant pas d’accès API au DNS.

**Limites :**
- Dépendance à l’ouverture et à l’accessibilité du port `80/tcp`.
- Non compatible avec l’émission de certificats wildcard.

### Option B — ACME DNS-01 via ClouDNS (retenue)
**Motifs :**
- Ne nécessite pas l’ouverture de `80/tcp` pour la validation ACME.
- Compatible avec les certificats wildcard.
- Alignée avec une approche edge et multi-services.

## Justification
1. **Réduction de la dépendance réseau** : DNS-01 évite un prérequis strict sur l’accessibilité HTTP pendant la validation ACME.
2. **Support des wildcard** : DNS-01 est requis pour `*.domain` et facilite l’exposition de plusieurs services.
3. **Centralisation edge** : Traefik demeure le composant unique de terminaison TLS sur le VPS.
4. **Interopérabilité** : Traefik supporte nativement ACME DNS-01 et une intégration efficace avec Docker.

## Conséquences

### Positives
- Automatisation robuste de l’émission et du renouvellement de certificats TLS.
- Possibilité d’émettre des certificats wildcard.
- Possibilité de rendre `80/tcp` optionnel sans bloquer l’ACME.
- Architecture cohérente : exposition publique au VPS, workloads applicatifs isolés dans le lab via WireGuard.

### Négatives / risques
- Nécessité de stocker des secrets ClouDNS sur le VPS.
- Risque opérationnel en cas de permissions API insuffisantes ou de propagation DNS lente.
- Dépannage potentiellement plus complexe (dépendance au DNS provider).

### Mesures de mitigation (préconisées)
- Utiliser un sous-utilisateur ClouDNS dédié avec droits minimaux (écriture TXT sur la zone).
- Stocker les secrets hors Git (fichier `.env` protégé, permissions strictes).
- Protéger le fichier `acme.json` (permissions strictes, sauvegarde si nécessaire).
- Ne pas exposer le dashboard Traefik publiquement.
- Mettre en place une vérification périodique du renouvellement (logs/monitoring).

## Definition of Done (validation)
- Traefik déployé sur le VPS.
- Certificat Let’s Encrypt émis via DNS-01 pour `matrix.cozy-flaow.cloudns.org`.
- Renouvellement automatique fonctionnel (validation par logs Traefik et/ou observation d’un renouvellement).
- Secrets ClouDNS stockés hors Git avec permissions strictes.
- Runbook exécutable présent dans `platform/runbooks/`.