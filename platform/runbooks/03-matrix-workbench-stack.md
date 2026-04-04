# Runbook 03 — Stack Matrix (Swarm) avec création admin automatisée via Docker Secrets

## Objectif
Déployer Synapse + Nginx + WireGuard en Docker Swarm et créer automatiquement l’utilisateur admin Synapse **sans mot de passe en clair** (Docker Secret).

## Pré-requis
- Docker Swarm initialisé (`docker swarm init`)
- Node manager disponible
- Domaine Matrix fonctionnel côté edge (Traefik)
- Répertoire de travail: `/opt/workbench`

## 1) Arborescence
```text
/opt/workbench/
  stack.yml
  .env
  scripts/
    init-admin.sh
  synapse/data/
  nginx/default.conf
  wireguard/
    wg_confs/wg0.conf
```

## 2) Initialiser Synapse data (1ère fois)
```bash
mkdir -p /opt/workbench/synapse/data
docker run --rm -it \
  -v /opt/workbench/synapse/data:/data \
  -e SYNAPSE_SERVER_NAME=matrix.flaow.eu \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate

sudo chown -R 991:991 /opt/workbench/synapse/data
sudo chmod -R u+rwX,go-rwx /opt/workbench/synapse/data
```

## 3) Créer les secrets Swarm
```bash
# mot de passe admin
openssl rand -base64 32 | docker secret create synapse_admin_password -

# (optionnel) shared secret registration
openssl rand -hex 32 | docker secret create synapse_registration_shared_secret -
```

Vérifier :
```bash
docker secret ls
```

## 4) Variables `.env`
```env
SYNAPSE_SERVER_NAME=matrix.flaow.eu
SYNAPSE_ADMIN_USER=admin
```

## 5) Stack Swarm `stack.yml`
```yaml
version: "3.9"

secrets:
  synapse_admin_password:
    external: true
  synapse_registration_shared_secret:
    external: true

services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    environment:
      TZ: Europe/Zurich
      PUID: "911"
      PGID: "911"
    cap_add:
      - NET_ADMIN
    volumes:
      - /opt/workbench/wireguard:/config
      - /lib/modules:/lib/modules:ro
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    deploy:
      placement:
        constraints:
          - node.role == manager

  synapse:
    image: matrixdotorg/synapse:latest
    environment:
      SYNAPSE_SERVER_NAME: ${SYNAPSE_SERVER_NAME}
      SYNAPSE_REPORT_STATS: "no"
      TZ: Europe/Zurich
    volumes:
      - /opt/workbench/synapse/data:/data
    network_mode: "service:wireguard"
    depends_on:
      - wireguard
    deploy:
      placement:
        constraints:
          - node.role == manager

  nginx:
    image: nginx:alpine
    volumes:
      - /opt/workbench/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    network_mode: "service:wireguard"
    depends_on:
      - synapse
    deploy:
      placement:
        constraints:
          - node.role == manager
```

## 6) Script init admin `scripts/init-admin.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail

STACK_NAME="workbench"
SYNAPSE_SERVICE="${STACK_NAME}_synapse"
WIREGUARD_SERVICE="${STACK_NAME}_wireguard"
MARKER="/opt/workbench/synapse/data/.admin_created"
ADMIN_USER="${SYNAPSE_ADMIN_USER:-admin}"

if [ -f "$MARKER" ]; then
  echo "[init-admin] Admin déjà créé, skip."
  exit 0
fi

# Récupère un conteneur task en cours pour synapse et wireguard
SYNAPSE_CID="$(docker ps --filter "name=${SYNAPSE_SERVICE}" --format '{{.ID}}' | head -n1)"
WG_CID="$(docker ps --filter "name=${WIREGUARD_SERVICE}" --format '{{.ID}}' | head -n1)"

if [ -z "$SYNAPSE_CID" ] || [ -z "$WG_CID" ]; then
  echo "[init-admin] Conteneurs Swarm non trouvés."
  exit 1
fi

echo "[init-admin] Attente Synapse..."
for i in {1..60}; do
  if docker exec "$WG_CID" sh -lc "wget -q -O- http://127.0.0.1:8008/_matrix/client/versions >/dev/null"; then
    break
  fi
  sleep 2
done

# Lire secret Swarm via service temporaire (pattern sûr)
PASS="$(docker run --rm --secret synapse_admin_password alpine:3.22 sh -lc 'cat /run/secrets/synapse_admin_password')"

echo "[init-admin] Création admin ${ADMIN_USER}..."
docker exec -i "$SYNAPSE_CID" register_new_matrix_user \
  -u "$ADMIN_USER" \
  -p "$PASS" \
  -a \
  -c /data/homeserver.yaml \
  http://localhost:8008

touch "$MARKER"
chmod 600 "$MARKER"
echo "[init-admin] OK"
```

Rendre exécutable :
```bash
chmod +x /opt/workbench/scripts/init-admin.sh
```

## 7) Déploiement
```bash
cd /opt/workbench
set -a; source .env; set +a
docker stack deploy -c stack.yml workbench
```

## 8) Lancer init admin (une seule fois)
```bash
cd /opt/workbench
set -a; source .env; set +a
/opt/workbench/scripts/init-admin.sh
```

## 9) Vérifications
```bash
docker service ls
docker service logs -f workbench_synapse
curl -i https://matrix.flaow.eu/_matrix/client/versions
curl -i https://matrix.flaow.eu/_matrix/federation/v1/version
```

## 10) Rotation du secret admin
```bash
openssl rand -base64 32 | docker secret create synapse_admin_password_v2 -

# Mettre à jour les services/automation pour consommer _v2 puis supprimer l’ancien
docker secret rm synapse_admin_password
docker secret ls
```