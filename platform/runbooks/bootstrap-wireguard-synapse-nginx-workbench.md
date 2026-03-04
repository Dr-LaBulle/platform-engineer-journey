# WORKBENCH (LAB) — Runbook : WireGuard peer + Synapse + Nginx interne (backend unique)

## Objectif
- Établir un tunnel WireGuard sortant depuis le lab (CGNAT) vers le VPS (Edge).
- Fournir un backend unique dans le tunnel :
  **http://10.100.0.2:8080**
- Router le trafic Matrix via Nginx interne :
  - `/_matrix/*` → Synapse :8008 (Client-Server)
  - `/_matrix/federation/*` → Synapse :8008 (Federation, mode compatible workbench)
- Éviter toute exposition de ports entrants publics sur le lab.

## Plan d’adressage
- **Réseau WireGuard : 10.100.0.0/30**
  - VPS (wg0) : `10.100.0.1/30`
  - LAB (wg0) : `10.100.0.2/30`
- **Port WireGuard : UDP/51820** (sortant lab → VPS)

## Pré-requis
- Docker + Docker Compose disponibles sur la machine *workbench*.
- Le flux sortant **UDP/51820** depuis le lab vers l’IP publique du VPS est autorisé.
- Le répertoire `./synapse/data` contient une configuration Synapse valide (`homeserver.yaml` + clés).
  - Si `./synapse/data` est vide, une initialisation Synapse est nécessaire avant démarrage.

# 1) Arborescence (tout dans `/opt/workbench`)
```
/opt/workbench/
  compose.yaml
  wireguard/
    wg0.conf
  synapse/
    data/
  nginx/
    default.conf
```

## Préparation (permissions)
```bash
mkdir -p /opt/workbench/wireguard /opt/workbench/synapse/data /opt/workbench/nginx
chmod 0750 /opt/workbench /opt/workbench/wireguard /opt/workbench/synapse /opt/workbench/nginx
touch /opt/workbench/wireguard/wg0.conf
chmod 0600 /opt/workbench/wireguard/wg0.conf
```

# 2) WireGuard (LAB) : `/opt/workbench/wireguard/wg0.conf`
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

## Notes importantes
- Le peer côté VPS doit inclure `10.100.0.2/32` dans `AllowedIPs`.
- Aucun port entrant n’est publié sur l’hôte du lab : tout passe dans le tunnel.

# 3) Docker Compose (LAB) : `/opt/workbench/compose.yaml`
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
      - SYNAPSE_SERVER_NAME=matrix.example.tld
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

# 4) Nginx (LAB) : `/opt/workbench/nginx/default.conf`
```nginx
server {
  listen 8080;
  server_name _;

  location = /healthz {
    return 200 "ok
";
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

# 5) Déploiement / Ops
```bash
cd /opt/workbench
docker compose up -d
docker compose ps
```

# 6) VPS (Edge) : routage Traefik attendu
Router **matrix.example.tld** vers :
```
http://10.100.0.2:8080
```

# 7) Vérifications

### WireGuard
```bash
docker exec -it wb-wireguard wg show
```

### Logs
```bash
docker logs -f wb-wireguard
docker logs -f wb-synapse
docker logs -f wb-nginx
```

### Tests (depuis le VPS via tunnel)
```bash
curl -sS -I http://10.100.0.2:8080/healthz | head
curl -sS -I http://10.100.0.2:8080/_matrix/client/versions | head
curl -sS -I http://10.100.0.2:8080/_matrix/federation/v1/version | head
```

## Point de cohérence Synapse (federation)
- Vérifier `homeserver.yaml` : listeners, server_name, public_baseurl, x_forwarded.
- Vérifier headers Nginx.
- Tester endpoint federation.
- Utiliser valideur externe si besoin.
