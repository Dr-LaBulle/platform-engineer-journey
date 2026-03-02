# Runbook — Traefik (edge) + Let’s Encrypt (ACME DNS-01) via ClouDNS

## Objectif
Déployer **Traefik** sur le VPS (edge reverse proxy) et obtenir automatiquement des certificats TLS via **Let’s Encrypt** en utilisant le challenge **DNS-01** avec **ClouDNS**.

## Contexte (valeurs connues)
- VPS public (edge): `83.228.221.69`
- Domaine: `cozy-flaow.cloudns.org`
- Service public Matrix/Synapse: `matrix.cozy-flaow.cloudns.org`
- Système: Linux + Docker + Docker Compose

## Pré-requis
- Accès root sur le VPS
- Docker Engine installé
- Docker Compose disponible (`docker compose version`)
- Zone DNS gérée via ClouDNS (zone correspondant au domaine utilisé)
- Enregistrement DNS public :
  - `A matrix.cozy-flaow.cloudns.org -> 83.228.221.69`

## Réseau / Firewall (VPS)
Ports à autoriser :
- `443/tcp` (obligatoire)
- `80/tcp` (optionnel mais recommandé : redirect HTTP→HTTPS, diagnostics)
- `51820/udp` (WireGuard, hors périmètre de ce runbook)

> Le challenge DNS-01 ne nécessite pas `80/tcp` pour ACME. Le port 80 reste utile pour des redirections et certains tests.

## Secrets (NE PAS COMMITER)
Traefik doit pouvoir créer des enregistrements TXT `_acme-challenge.*` via l’API ClouDNS.

Variables requises (exactes) :
- `CLOUDNS_AUTH_ID`
- `CLOUDNS_AUTH_PASSWORD`
- `CLOUDNS_SUB_AUTH_ID` (optionnel)

Stockage recommandé sur le VPS :
- `/opt/traefik/.env` (permissions `0600`, propriétaire `root:root`)

## Arborescence recommandée (VPS)
- `/opt/traefik/`
  - `.env` (secrets ClouDNS)
  - `docker-compose.yml`
  - `letsencrypt/acme.json` (state ACME, contient des clés privées)

## Procédure

### 1) Créer les répertoires
```bash
sudo mkdir -p /opt/traefik/letsencrypt
```

### 2) Initialiser le fichier de state ACME
```bash
sudo touch /opt/traefik/letsencrypt/acme.json
sudo chmod 600 /opt/traefik/letsencrypt/acme.json
```

### 3) Créer le fichier `.env` (secrets ClouDNS)
Créer `/opt/traefik/.env` :
```bash
sudo nano /opt/traefik/.env
```

Contenu (exemple) :
```text
CLOUDNS_AUTH_ID=...
CLOUDNS_AUTH_PASSWORD=...
# optionnel:
CLOUDNS_SUB_AUTH_ID=...
```

Durcissement :
```bash
sudo chown root:root /opt/traefik/.env
sudo chmod 600 /opt/traefik/.env
```

### 4) Créer un réseau Docker dédié (proxy)
Ce réseau permet �� Traefik de joindre d’autres containers (présents ou futurs) de manière stable.
```bash
sudo docker network create proxy 2>/dev/null || true
```

### 5) Créer `/opt/traefik/docker-compose.yml`
```bash
sudo nano /opt/traefik/docker-compose.yml
```

Contenu :
```yaml
services:
  traefik:
    image: traefik:v3.3
    container_name: traefik
    restart: unless-stopped

    environment:
      - CLOUDNS_AUTH_ID=${CLOUDNS_AUTH_ID}
      - CLOUDNS_SUB_AUTH_ID=${CLOUDNS_SUB_AUTH_ID}
      - CLOUDNS_AUTH_PASSWORD=${CLOUDNS_AUTH_PASSWORD}

    command:
      # Provider Docker (Traefik lit les labels)
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false

      # Points d’entrée
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443

      # ACME (Let’s Encrypt) via DNS-01 (ClouDNS)
      - --certificatesresolvers.le.acme.dnschallenge=true
      - --certificatesresolvers.le.acme.dnschallenge.provider=cloudns
      - --certificatesresolvers.le.acme.email=admin@cozy-flaow.cloudns.org
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json

      # Logs (ajustable)
      - --log.level=INFO

    ports:
      - "443:443"
      - "80:80"

    volumes:
      # Lecture seule du socket Docker (découverte des containers/labels)
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Stockage persistant ACME (contient des clés privées)
      - ./letsencrypt:/letsencrypt

    networks:
      - proxy

networks:
  proxy:
    external: true
    name: proxy
```

### 6) Démarrer Traefik
```bash
cd /opt/traefik
sudo docker compose up -d
sudo docker ps
```

### 7) Vérifier les logs
```bash
sudo docker logs -f traefik
```

## Validation (ACME)
Le certificat n’est obtenu que lorsqu’un **routeur Traefik** utilise le resolver `le`.

Une fois un routeur en place (ex : pour `matrix.cozy-flaow.cloudns.org`) :
- Traefik crée un TXT `_acme-challenge.matrix.cozy-flaow.cloudns.org`
- Let’s Encrypt vérifie le TXT
- Traefik stocke le certificat dans `letsencrypt/acme.json`

Contrôle DNS pendant l’émission :
```bash
dig TXT _acme-challenge.matrix.cozy-flaow.cloudns.org
```

## Dépannage
### Erreurs d’authentification ClouDNS
- Vérifier `CLOUDNS_AUTH_ID` / `CLOUDNS_AUTH_PASSWORD`
- Si `CLOUDNS_SUB_AUTH_ID` est utilisé, vérifier sa validité et ses permissions

### Propagation DNS lente
- Attendre (délai variable selon TTL/propagation)
- Vérifier le TXT avec `dig` sur plusieurs resolvers

### Permissions des fichiers
- `letsencrypt/acme.json` doit être en `0600`
- `.env` doit être en `0600`

## Notes de sécurité
- Ne pas commiter `.env` ni `acme.json`
- Ne pas exposer le dashboard Traefik publiquement (hors scope de ce runbook)
- Privilégier un sous-utilisateur ClouDNS dédié et à privilèges minimaux