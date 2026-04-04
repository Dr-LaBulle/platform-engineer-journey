# Runbook 02 — Traefik VPS en Swarm mono-node + ACME DNS-01 (Infomaniak via secret)

## Objectif
Déployer Traefik sur le VPS en mode **Swarm mono-node** pour gérer proprement le token Infomaniak en **Docker secret**.

## Pré-requis
- WireGuard opérationnel (runbook 01)
- Docker Engine installé
- Ports entrants ouverts : `80/tcp`, `443/tcp`
- Domaine prêt (ex. `matrix.example.net`, `maintenance.example.net`)

## 1) Init Swarm mono-node
```bash
docker swarm init
docker node ls
```

## 2) Préparer arborescence
```bash
mkdir -p /opt/traefik/{dynamic,letsencrypt,maintenance}
touch /opt/traefik/letsencrypt/acme.json
chmod 600 /opt/traefik/letsencrypt/acme.json
```

## 3) Créer secret ACME Infomaniak
```bash
printf '%s' 'REPLACE_WITH_INFOMANIAK_TOKEN' | docker secret create infomaniak_access_token -
docker secret ls
```

## 4) Variables non sensibles
Créer `/opt/traefik/.env`
```env
ACME_EMAIL=admin@example.net
```

```bash
chmod 600 /opt/traefik/.env
```

## 5) Page maintenance
Créer `/opt/traefik/maintenance/index.html` (contenu statique libre).

## 6) Stack Swarm `/opt/traefik/stack.yml`
```yaml
version: "3.8"

secrets:
  infomaniak_access_token:
    external: true

networks:
  proxy:
    driver: overlay
    attachable: true

services:
  traefik:
    image: traefik:v2.11
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
    environment:
      - ACME_EMAIL=${ACME_EMAIL}
      - INFOMANIAK_ACCESS_TOKEN_FILE=/run/secrets/infomaniak_access_token
    secrets:
      - infomaniak_access_token
    command:
      - --providers.docker=true
      - --providers.docker.swarmMode=true
      - --providers.docker.exposedbydefault=false
      - --providers.file.directory=/dynamic
      - --providers.file.watch=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.email=${ACME_EMAIL}
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.le.acme.keytype=RSA4096
      - --certificatesresolvers.le.acme.dnschallenge=true
      - --certificatesresolvers.le.acme.dnschallenge.provider=infomaniak
      - --certificatesresolvers.le.acme.dnschallenge.delaybeforecheck=30
      - --certificatesresolvers.le.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53
      - --log.level=INFO
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/traefik/letsencrypt:/letsencrypt
      - /opt/traefik/dynamic:/dynamic:ro
    networks:
      - proxy
    deploy:
      placement:
        constraints:
          - node.role == manager

  maintenance-page:
    image: nginx:alpine
    volumes:
      - /opt/traefik/maintenance/index.html:/usr/share/nginx/html/index.html:ro
    networks:
      - proxy
    deploy:
      labels:
        - traefik.enable=true
        - traefik.http.services.matrix-maintenance-svc.loadbalancer.server.port=80
        - traefik.http.routers.matrix-maintenance.rule=Host(`maintenance.example.net`)
        - traefik.http.routers.matrix-maintenance.entrypoints=websecure
        - traefik.http.routers.matrix-maintenance.tls=true
        - traefik.http.routers.matrix-maintenance.tls.certresolver=le
        - traefik.http.routers.matrix-maintenance.service=matrix-maintenance-svc
```

## 7) Dynamic config `/opt/traefik/dynamic/matrix.yml`
```yaml
http:
  routers:
    matrix:
      rule: Host(`matrix.example.net`)
      entryPoints: [websecure]
      tls:
        certResolver: le
      service: synapse-svc
      middlewares:
        - matrix-fallback

  services:
    synapse-svc:
      loadBalancer:
        servers:
          - url: "http://10.100.0.2:8080"

    matrix-maintenance-svc:
      loadBalancer:
        servers:
          - url: "http://maintenance-page:80"

  middlewares:
    matrix-fallback:
      errors:
        status: ["500-599"]
        service: matrix-maintenance-svc
        query: "/"
```

## 8) Déployer
```bash
cd /opt/traefik
set -a; source .env; set +a
docker stack deploy -c stack.yml edge
docker stack services edge
docker service logs -f edge_traefik
```

## 9) Validation
- `https://maintenance.example.net`
- `https://matrix.example.net/_matrix/client/versions`
- `dig TXT _acme-challenge.matrix.example.net`