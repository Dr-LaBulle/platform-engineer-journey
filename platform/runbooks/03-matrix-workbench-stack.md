# Runbook 03 — Matrix Workbench (Docker Swarm) : Synapse + Nginx + secrets

## Objectif
Déployer Synapse en Docker Swarm sur le LAB, exposer Matrix via Nginx (`:8080`), gérer les secrets proprement (sans mot de passe en clair), et garder la création d’admin manuelle.

---

## 1) Pré-requis

- Docker Engine installé
- Docker Swarm initialisé
- Répertoire de travail: `/opt/workbench`
- DNS/routage edge en place (ex: `matrix.flaow.eu` -> reverse proxy vers `http://10.100.0.2:8080`)

Vérifier :
```bash
docker info | grep -i swarm
```

---

## 2) Arborescence

```text
/opt/workbench/
  .env
  stack.yml
  nginx/
    default.conf
  synapse/
    data/
```

Créer :
```bash
mkdir -p /opt/workbench/{nginx,synapse/data}
```

---

## 3) Initialiser Synapse (1ère installation uniquement)

```bash
docker run --rm -it \
  -v /opt/workbench/synapse/data:/data \
  -e SYNAPSE_SERVER_NAME=matrix.flaow.eu \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate
```

Permissions recommandées :
```bash
sudo chown -R 991:991 /opt/workbench/synapse/data
sudo chmod -R u+rwX,go-rwx /opt/workbench/synapse/data
```

---

## 4) Créer les secrets Swarm

```bash
# shared secret (API admin / registration selon ton usage)
openssl rand -hex 32 | docker secret create synapse_registration_shared_secret -

# mot de passe SMTP (si email activé)
printf '%s' 'REMPLACE_PAR_MDP_SMTP' | docker secret create synapse_smtp_pass -
```

Vérifier :
```bash
docker secret ls
```

---

## 5) Fichier `.env`

Créer `/opt/workbench/.env` :

```env
SYNAPSE_SERVER_NAME=matrix.flaow.eu
TZ=Europe/Zurich
```

---

## 6) Configuration complète Synapse (`/opt/workbench/synapse/data/homeserver.yaml`)

> Tout le `homeserver.yaml` est regroupé ici.

```yaml
server_name: "matrix.flaow.eu"
pid_file: /data/homeserver.pid
report_stats: false

listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    resources:
      - names: [client, federation]
        compress: false

database:
  name: sqlite3
  args:
    database: /data/homeserver.db

log_config: "/data/matrix.flaow.eu.log.config"
media_store_path: /data/media_store
signing_key_path: "/data/matrix.flaow.eu.signing.key"

trusted_key_servers:
  - server_name: "matrix.org"

# Inscription (sans Google captcha)
enable_registration: true
enable_registration_without_verification: false
registration_requires_token: true
enable_registration_captcha: false

# Vérification email (optionnel — décommenter si nécessaire)
# registrations_require_3pid:
#   - email
# email:
#   smtp_host: "smtp.example.com"
#   smtp_port: 587
#   smtp_user: "no-reply@flaow.eu"
#   smtp_pass: "${SYNAPSE_SMTP_PASS}"
#   require_transport_security: true
#   notif_from: "Matrix <no-reply@flaow.eu>"
```

Notes :
- Pas de mot de passe SMTP en clair.
- Si email activé, `smtp_host` doit être réel et joignable.
- Ne pas activer reCAPTCHA si tu ne veux pas Google.

---

## 7) Configuration Nginx (`/opt/workbench/nginx/default.conf`)

```nginx
server {
  listen 8080;
  server_name _;

  location = /healthz {
    return 200 "ok\n";
    add_header Content-Type text/plain;
  }

  location /_matrix/ {
    proxy_pass http://synapse:8008;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Port 443;

    # CORS pour clients web
    add_header Access-Control-Allow-Origin * always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "X-Requested-With, Content-Type, Authorization, Date" always;
    if ($request_method = OPTIONS) { return 204; }
  }

  location /_matrix/federation/ {
    proxy_pass http://synapse:8008;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Port 443;
  }
}
```

---

## 8) Stack Swarm (`/opt/workbench/stack.yml`)

```yaml
version: "3.9"

secrets:
  synapse_registration_shared_secret:
    external: true
  synapse_smtp_pass:
    external: true

networks:
  matrix_backend:
    driver: overlay
    attachable: true

services:
  synapse:
    image: matrixdotorg/synapse:latest
    networks:
      - matrix_backend
    volumes:
      - /opt/workbench/synapse/data:/data
    secrets:
      - synapse_registration_shared_secret
      - synapse_smtp_pass
    environment:
      SYNAPSE_SERVER_NAME: ${SYNAPSE_SERVER_NAME}
      SYNAPSE_REPORT_STATS: "no"
      TZ: ${TZ}
    entrypoint:
      - /bin/sh
      - -lc
      - |
        export SYNAPSE_SMTP_PASS="$(cat /run/secrets/synapse_smtp_pass)";
        exec /start.py
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  nginx:
    image: nginx:alpine
    networks:
      - matrix_backend
    volumes:
      - /opt/workbench/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    ports:
      - target: 8080
        published: 8080
        protocol: tcp
        mode: host
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
```

---

## 9) Déploiement

```bash
cd /opt/workbench
set -a; source .env; set +a
docker stack deploy -c stack.yml workbench
```

Vérifier :
```bash
docker service ls
docker service ps workbench_synapse
docker service ps workbench_nginx
docker service logs --tail=200 workbench_synapse
```

---

## 10) Création admin manuelle

```bash
docker exec -it $(docker ps --filter name=workbench_synapse --format '{{.ID}}' | head -n1) \
  register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```

---

## 11) Vérifications

Depuis le LAB :
```bash
curl -i http://10.100.0.2:8080/healthz
curl -i http://10.100.0.2:8080/_matrix/client/versions
```

Depuis Internet :
```bash
curl -i https://matrix.flaow.eu/_matrix/client/versions
curl -i https://matrix.flaow.eu/_matrix/federation/v1/version
```

---

## 12) Dépannage rapide

- `Ignoring unsupported options: network_mode`  
  -> normal en stack Swarm, utiliser réseau overlay.

- `No appropriate authentication flow found`  
  -> flow client incompatible avec config (token/email/captcha).

- `DNSLookupError ... ton-smtp`  
  -> SMTP invalide (placeholder non remplacé).

- `Failed to fetch` (Element)  
  -> vérifier HTTPS, reverse proxy edge, CORS, endpoint `/_matrix/client/versions`.

- `401 POST /_matrix/client/v3/register`  
  -> souvent normal si token requis mais absent/invalide.