# Runbook 03 — Stack Matrix sur le workbench LAB (Synapse + Nginx via WireGuard)

## Objectif
Déployer la stack Matrix côté LAB sans exposition publique directe.
Le VPS edge (Traefik) consomme le backend tunnelisé : `http://10.100.0.2:8080`.

## Pré-requis
- Tunnel WireGuard opérationnel (runbook 01)
- VPS Traefik Swarm opérationnel (runbook 02)
- Répertoire Synapse prêt : `./synapse/data` (homeserver.yaml + clés)

## 1) Arborescence LAB
```text
/opt/workbench/
  compose.yaml
  wireguard/wg0.conf
  synapse/data/
  nginx/default.conf
```

## 2) WireGuard container `/opt/workbench/wireguard/wg0.conf`
```ini
[Interface]
Address = 10.100.0.2/30
PrivateKey = __LAB_PRIVATE_KEY__
DNS = 1.1.1.1

[Peer]
PublicKey = __VPS_PUBLIC_KEY__
Endpoint = __VPS_PUBLIC_IP__:51820
AllowedIPs = 10.100.0.1/32
PersistentKeepalive = 25
```

## 3) Compose `/opt/workbench/compose.yaml`
```yaml
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wb-wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - TZ=Europe/Zurich
    volumes:
      - ./wireguard:/config
      - /lib/modules:/lib/modules:ro
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped

  synapse:
    image: matrixdotorg/synapse:latest
    container_name: wb-synapse
    network_mode: "service:wireguard"
    depends_on:
      - wireguard
    environment:
      - TZ=Europe/Zurich
      - SYNAPSE_SERVER_NAME=matrix.example.net
      - SYNAPSE_REPORT_STATS=no
    volumes:
      - ./synapse/data:/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    container_name: wb-nginx
    network_mode: "service:wireguard"
    depends_on:
      - wireguard
      - synapse
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    restart: unless-stopped
```

## 4) Nginx `/opt/workbench/nginx/default.conf`
```nginx
server {
  listen 8080;
  server_name _;

  location = /healthz {
    return 200 "ok\n";
    add_header Content-Type text/plain;
  }

  location /_matrix/ {
    proxy_pass http://127.0.0.1:8008;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Port 443;
  }

  location /_matrix/federation/ {
    proxy_pass http://127.0.0.1:8008;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Port 443;
  }
}
```

## 5) Déploiement
```bash
cd /opt/workbench
docker compose up -d
docker compose ps
```

## 6) Vérifications
```bash
docker exec -it wb-wireguard wg show
docker logs -f wb-wireguard
docker logs -f wb-synapse
docker logs -f wb-nginx
```

Depuis le VPS :
```bash
curl -sS -I http://10.100.0.2:8080/healthz | head
curl -sS -I http://10.100.0.2:8080/_matrix/client/versions | head
curl -sS -I http://10.100.0.2:8080/_matrix/federation/v1/version | head
```

## 7) Cohérence Synapse
Vérifier `homeserver.yaml` :
- `server_name`
- `public_baseurl`
- listeners
- `x_forwarded: true` (si applicable)