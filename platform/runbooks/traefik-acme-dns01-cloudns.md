# Runbook — Traefik (edge) + Let’s Encrypt (ACME DNS-01) via ClouDNS + Fallback maintenance (interception 5xx)

## Objectif
1) Déployer **Traefik** sur le VPS (edge reverse proxy) et obtenir automatiquement des certificats TLS via **Let’s Encrypt** en utilisant le challenge **DNS-01** avec **ClouDNS**.  
2) Mettre en place un **fallback automatique** vers une page de maintenance (hébergée sur le VPS) en cas d’erreurs applicatives **HTTP 5xx (500–599)** du backend (ex : Synapse indisponible côté lab).

## Contexte (valeurs connues)
- VPS public (edge): `83.228.221.69`
- Domaine: `cozy-flaow.cloudns.org`
- Service public Matrix/Synapse: `matrix.cozy-flaow.cloudns.org`
- Dépôt GitHub (documentation): https://github.com/Dr-LaBulle/sovereign-hybrid-infra-showcase/

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
  - `maintenance/index.html` (landing page statique)

## Procédure

### 1) Créer les répertoires
```bash
sudo mkdir -p /opt/traefik/letsencrypt
sudo mkdir -p /opt/traefik/maintenance
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
```bash
sudo docker network create proxy 2>/dev/null || true
```

---

## 5) Fallback portfolio — Landing page de maintenance (VPS)
### 5.1 Créer `/opt/traefik/maintenance/index.html`
```bash
sudo nano /opt/traefik/maintenance/index.html
```

Contenu (inclut un auto-refresh 300s) :
```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="refresh" content="300">
    <title>Sovereign Bridge - Maintenance</title>
    <style>
        body { font-family: sans-serif; background: #1a1a1a; color: #eee; display: flex; align-items: center; justify-content: center; height: 100vh; margin: 0; }
        .card { text-align: center; border: 1px solid #333; padding: 2rem; border-radius: 8px; background: #222; max-width: 820px; }
        a { color: #00aaff; text-decoration: none; font-weight: bold; }
        .status { color: #ff5555; }
        .ok { color: #55ff55; }
        code { background: #111; padding: 0.15rem 0.35rem; border-radius: 4px; }
        .muted { opacity: 0.85; font-size: 0.95rem; }
    </style>
</head>
<body>
    <div class="card">
        <h1>Sovereign Bridge</h1>
        <p>Passerelle Suisse (CH) : <span class="ok">ACTIVE</span></p>
        <p>Laboratoire Portugal (PT) : <span class="status">HORS-LIGNE</span></p>

        <hr style="border: 0; border-top: 1px solid #333; margin: 1.5rem 0;">

        <p>Ce projet (liaison CH↔PT, WireGuard + reverse proxy edge) est documenté sur GitHub :</p>
        <p>
          <a href="https://github.com/Dr-LaBulle/sovereign-hybrid-infra-showcase/">
            Accéder à la documentation du projet ➔
          </a>
        </p>

        <p class="muted">
          Cette page se rafraîchit automatiquement toutes les 5 minutes.
        </p>

        <p class="muted">
          Point d’entrée public : <code>matrix.cozy-flaow.cloudns.org</code>
        </p>
    </div>
</body>
</html>
```

---

## 6) Déployer Traefik + service maintenance (compose VPS)
Créer `/opt/traefik/docker-compose.yml` :
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
      # Providers
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false

      # EntryPoints
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443

      # ACME DNS-01 (ClouDNS)
      - --certificatesresolvers.le.acme.dnschallenge=true
      - --certificatesresolvers.le.acme.dnschallenge.provider=cloudns
      - --certificatesresolvers.le.acme.email=admin@cozy-flaow.cloudns.org
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json

      # Logs
      - --log.level=INFO

    ports:
      - "443:443"
      - "80:80"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt

    networks:
      - proxy

  maintenance-page:
    image: nginx:alpine
    container_name: gateway-maintenance
    restart: unless-stopped
    volumes:
      - ./maintenance/index.html:/usr/share/nginx/html/index.html:ro
    networks:
      - proxy
    labels:
      - traefik.enable=true

      # Service Traefik => nginx:80
      - traefik.http.services.matrix-maintenance-svc.loadbalancer.server.port=80

      # Router maintenance (sert uniquement à exposer la page si besoin de test direct)
      - traefik.http.routers.matrix-maintenance.rule=Host(`maintenance.matrix.cozy-flaow.cloudns.org`)
      - traefik.http.routers.matrix-maintenance.entrypoints=websecure
      - traefik.http.routers.matrix-maintenance.tls=true
      - traefik.http.routers.matrix-maintenance.tls.certresolver=le
      - traefik.http.routers.matrix-maintenance.service=matrix-maintenance-svc

networks:
  proxy:
    external: true
    name: proxy
```

Notes :
- Le router `maintenance.matrix.cozy-flaow.cloudns.org` est facultatif, mais utile pour valider rapidement la page de maintenance (et donc le resolver ACME) sans dépendre du backend Synapse.
- Cela implique un DNS additionnel si utilisé :
  - `A maintenance.matrix.cozy-flaow.cloudns.org -> 83.228.221.69`

---

## 7) Interception automatique des erreurs 5xx (middleware)
### Objectif
Intercepter les réponses `500-599` du backend principal (Synapse) et servir à la place la page de maintenance.

### Principe Traefik
Utiliser un middleware `errors` :
- `status=500-599`
- `service=matrix-maintenance-svc`
- `query=/{path}` (pour servir `index.html`)

> Le middleware doit être attaché au router du service principal (router Synapse). Il peut être défini via labels (si le router Synapse est défini via labels) ou via un fichier de config dynamique (file provider).

### Exemple (labels à appliquer sur le router Synapse)
Sur le router qui sert `matrix.cozy-flaow.cloudns.org`, ajouter :
- `traefik.http.routers.matrix.middlewares=matrix-fallback@docker`
- `traefik.http.middlewares.matrix-fallback.errors.status=500-599`
- `traefik.http.middlewares.matrix-fallback.errors.service=matrix-maintenance-svc`
- `traefik.http.middlewares.matrix-fallback.errors.query=/`

Remarques importantes :
- Cette interception fonctionne sur des **réponses 5xx**.  
  Si le backend est totalement injoignable, Traefik renvoie souvent un **502/504**, ce qui est bien couvert par `500-599` (inclut 502/504).
- La page de maintenance doit rester disponible localement (nginx sur VPS), ce qui est le cas.

---

## 8) Démarrer / redémarrer
```bash
cd /opt/traefik
sudo docker compose up -d
sudo docker ps
sudo docker logs -f traefik
```

---

## Validation
### 1) Validation DNS-01 / ACME
Lorsqu’un router TLS utilise `certResolver: le`, Traefik déclenche l’ACME DNS-01.

Contrôle du TXT (pendant émission) :
```bash
dig TXT _acme-challenge.matrix.cozy-flaow.cloudns.org
```

### 2) Validation de la page de maintenance
Optionnel (si DNS `maintenance.matrix.cozy-flaow.cloudns.org` configuré) :
- accès HTTPS à `https://maintenance.matrix.cozy-flaow.cloudns.org`

### 3) Validation du fallback 5xx
- Simuler un backend Synapse indisponible (arrêt du service ou blocage réseau).
- Vérifier que `https://matrix.cozy-flaow.cloudns.org` sert la landing page de maintenance.

---

## Dépannage
- Auth ClouDNS : vérifier variables `.env` et permissions API
- Propagation DNS : attendre et vérifier TXT via `dig`
- Permissions : `.env` et `acme.json` en `0600`
- Logs Traefik : rechercher `acme`, `cloudns`, `challenge`, `error`