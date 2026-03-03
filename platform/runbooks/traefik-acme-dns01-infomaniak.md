# Runbook — Traefik (edge) + Let’s Encrypt (ACME DNS-01) via Infomaniak + Fallback maintenance (interception 5xx)

## Objectif
1) Déployer **Traefik** sur le VPS (edge reverse proxy) et obtenir automatiquement des certificats TLS via **Let’s Encrypt** en utilisant le challenge **DNS-01** avec **Infomaniak**.  
2) Mettre en place un **fallback automatique** vers une page de maintenance (hébergée sur le VPS) en cas d’erreurs applicatives **HTTP 5xx (500–599)** du backend (ex : Synapse indisponible côté lab).

## Contexte (valeurs génériques)
> Ce runbook est volontairement générique (pas de nom de domaine réel / endpoints sensibles commités).

- VPS public (edge) : `<VPS_PUBLIC_IPV4>` (+ éventuellement `<VPS_PUBLIC_IPV6>`)
- Domaine : `example.net`
- Service public Matrix/Synapse : `matrix.example.net`
- Service public maintenance : `maintenance.example.net`
- Dépôt GitHub (documentation) : https://github.com/Dr-LaBulle/platform-engineer-journey/

## Pré-requis
- Accès root sur le VPS
- Docker Engine installé
- Docker Compose disponible (`docker compose version`)
- Zone DNS gérée chez **Infomaniak**
- DNS public (recommandé) :
  - `A example.net -> <VPS_PUBLIC_IPV4>`
  - `AAAA example.net -> <VPS_PUBLIC_IPV6>` (si IPv6 utilisé)
  - `CNAME matrix.example.net -> example.net`
  - `CNAME maintenance.example.net -> example.net`

## Réseau / Firewall (VPS + provider)
Ports à autoriser **en entrée** :
- `443/tcp` (obligatoire)
- `80/tcp` (recommandé : redirect HTTP→HTTPS, diagnostics)
- `51820/udp` (WireGuard, hors périmètre de ce runbook)

Important :
- Si un **firewall provider Infomaniak** est activé, il doit autoriser `443/tcp` (et `80/tcp` si utilisé), sinon vous verrez des **timeouts** depuis Internet même si le service écoute localement.

## Secrets (NE PAS COMMITER)
Traefik doit pouvoir créer des enregistrements TXT `_acme-challenge.*` via l’API Infomaniak.

Variable requise :
- `INFOMANIAK_ACCESS_TOKEN`
- `ACME_EMAIL`

Stockage recommandé sur le VPS :
- `/opt/traefik/.env` (permissions `0600`, propriétaire `root:root`)

## Arborescence recommandée (VPS)
- `/opt/traefik/`
  - `.env` (secret Infomaniak)
  - `docker-compose.yml`
  - `dynamic/` (config Traefik via file provider)
  - `letsencrypt/acme.json` (state ACME, contient des clés privées)
  - `maintenance/index.html` (landing page statique)

---

## Procédure

### 1) Créer les répertoires
```bash
sudo mkdir -p /opt/traefik/letsencrypt
sudo mkdir -p /opt/traefik/dynamic
sudo mkdir -p /opt/traefik/maintenance
```

### 2) Initialiser le fichier de state ACME
```bash
sudo touch /opt/traefik/letsencrypt/acme.json
sudo chmod 600 /opt/traefik/letsencrypt/acme.json
```

### 3) Créer le fichier `.env` (secret Infomaniak)
Créer `/opt/traefik/.env` :
```bash
sudo nano /opt/traefik/.env
```

Contenu :
```text
INFOMANIAK_ACCESS_TOKEN=...
ACME_EMAIL=admin@example.net
```

Astuce : un template versionné est disponible dans `tech/docker-compose/stacks/VPS/.env.example`.

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

Exemple (auto-refresh 300s) :
```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="refresh" content="300">
  <title>Maintenance</title>
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
    <p>Passerelle (edge) : <span class="ok">ACTIVE</span></p>
    <p>Backend lab : <span class="status">HORS-LIGNE</span></p>

    <hr style="border: 0; border-top: 1px solid #333; margin: 1.5rem 0;">

    <p>Documentation du projet :</p>
    <p>
      <a href="https://github.com/Dr-LaBulle/platform-engineer-journey/">
        Accéder au dépôt ➔
      </a>
    </p>

    <p class="muted">Cette page se rafraîchit automatiquement toutes les 5 minutes.</p>
    <p class="muted">Point d’entrée public : <code>matrix.example.net</code></p>
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
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped

    env_file:
      - ./.env

    command:
      # Providers
      - --providers.docker=true
      - --providers.docker.endpoint=unix:///var/run/docker.sock
      - --providers.docker.exposedbydefault=false

      # Provider file (dynamic config)
      - --providers.file.directory=/dynamic
      - --providers.file.watch=true

      # EntryPoints
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443

      # Logs
      - --log.level=INFO

      # ACME (Let's Encrypt PRODUCTION) - DNS-01 Infomaniak
      - --certificatesresolvers.le.acme.email=<ACME_EMAIL>
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.le.acme.keytype=RSA4096
      - --certificatesresolvers.le.acme.dnschallenge=true
      - --certificatesresolvers.le.acme.dnschallenge.provider=infomaniak
      # Recommandé: éviter le DNS Docker (127.0.0.11) pour les checks de propagation
      - --certificatesresolvers.le.acme.dnschallenge.delaybeforecheck=30
      - --certificatesresolvers.le.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53

    ports:
      - "443:443"
      - "80:80"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
      - ./dynamic:/dynamic:ro

    networks:
      - proxy

  maintenance-page:
    image: nginx:alpine
    container_name: gateway_maintenance
    restart: unless-stopped

    volumes:
      - ./maintenance/index.html:/usr/share/nginx/html/index.html:ro

    networks:
      - proxy

    labels:
      - traefik.enable=true

      # Service Traefik => nginx:80
      - traefik.http.services.matrix-maintenance-svc.loadbalancer.server.port=80

      # Router maintenance (exposition directe de la page pour tests)
      - traefik.http.routers.matrix-maintenance.rule=Host(`maintenance.example.net`)
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
- Le router `maintenance.example.net` est utile pour valider rapidement :
  - la page maintenance
  - le resolver ACME
  - l’accessibilité Internet/Firewall
  sans dépendre du backend Synapse.

---

## 7) Interception automatique des erreurs 5xx (middleware)
### Objectif
Intercepter les réponses `500-599` du backend principal (Synapse) et servir à la place la page de maintenance.

### Principe Traefik
Utiliser un middleware `errors` :
- `status=500-599`
- `service=matrix-maintenance-svc`
- `query=/` (sert `index.html`)

> Le middleware doit être attaché au router du service principal (router Synapse). Ici on le fait via **file provider**.

Créer `/opt/traefik/dynamic/matrix.yml` :
```bash
sudo nano /opt/traefik/dynamic/matrix.yml
```

Contenu :
```yaml
http:
  routers:
    matrix:
      rule: Host(`matrix.example.net`)
      entryPoints:
        - websecure
      tls:
        certResolver: le
      service: synapse-svc
      middlewares:
        - matrix-fallback

  services:
    synapse-svc:
      loadBalancer:
        servers:
          # IMPORTANT: éviter une IP “figée” si possible.
          # Préférer un endpoint stable (DNS interne, nom de service docker, IP WireGuard stable, etc.)
          - url: "http://<SYNAPSE_PRIVATE_ENDPOINT>:8008"

    matrix-maintenance-svc:
      loadBalancer:
        servers:
          - url: "http://gateway_maintenance:80"

  middlewares:
    matrix-fallback:
      errors:
        status:
          - "500-599"
        service: matrix-maintenance-svc
        query: "/"
```

Remarques :
- Cette interception fonctionne sur des **réponses 5xx**.
- Si le backend est totalement injoignable, Traefik renvoie souvent `502/504`, ce qui est bien couvert par `500-599`.

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
dig TXT _acme-challenge.matrix.example.net
```

### 2) Validation de la page de maintenance
- Accès HTTPS à `https://maintenance.example.net`

### 3) Validation du fallback 5xx
- Simuler un backend Synapse indisponible (arrêt du service ou blocage réseau).
- Vérifier que `https://matrix.example.net` sert la landing page de maintenance (souvent avec un statut HTTP 502).

---

## Réinitialiser ACME (si mélange staging/prod ou état incohérent)
```bash
cd /opt/traefik
sudo docker compose down
sudo rm -f ./letsencrypt/acme.json
sudo touch ./letsencrypt/acme.json
sudo chmod 600 ./letsencrypt/acme.json
sudo docker compose up -d
```

---

## Dépannage
- Auth Infomaniak : vérifier `INFOMANIAK_ACCESS_TOKEN` (présence dans `.env`, droits, pas de leak).
- Firewall provider : en cas de **timeout depuis Internet** mais OK en local, vérifier l’ouverture de `443/tcp` côté provider.
- Propagation DNS : attendre et vérifier TXT via `dig`.
- Permissions : `.env` et `acme.json` en `0600`.
- Logs Traefik : rechercher `acme`, `infomaniak`, `challenge`, `error`.