# Runbook 01 — Tunnel WireGuard VPS ↔ LAB

## Objectif
Établir un tunnel WireGuard site-to-site minimal entre :
- VPS (CH) : endpoint public
- LAB (PT, CGNAT) : peer sortant

Scope MVP strict :
- Le VPS atteint uniquement `10.100.0.2` (WG) et `192.168.30.2` (wg-lab DMZ)

## Plan
- Réseau WG : `10.100.0.0/30`
  - VPS : `10.100.0.1/30`
  - LAB : `10.100.0.2/30`
- Port : `51820/udp`

## Pré-requis
- Accès root VPS + LAB
- UDP sortant lab vers VPS autorisé
- Firewall VPS : `51820/udp` ouvert

## 1) Installation WireGuard

### VPS (Debian/Ubuntu)
```bash
apt update && apt install -y wireguard wireguard-tools
```

### LAB (Debian/Ubuntu)
```bash
apt update && apt install -y wireguard wireguard-tools
```

## 2) Activation forwarding (VPS)
```bash
sysctl -w net.ipv4.ip_forward=1
printf "net.ipv4.ip_forward=1\n" > /etc/sysctl.d/99-wireguard.conf
sysctl --system
```

## 3) Génération des clés

### VPS
```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
chmod 600 /etc/wireguard/privatekey
cat /etc/wireguard/publickey
```

### LAB
```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
chmod 600 /etc/wireguard/privatekey
cat /etc/wireguard/publickey
```

## 4) Configuration

### VPS `/etc/wireguard/wg0.conf`
```ini
[Interface]
Address = 10.100.0.1/30
ListenPort = 51820
PrivateKey = <VPS_PRIVATE_KEY>

[Peer]
PublicKey = <LAB_PUBLIC_KEY>
AllowedIPs = 10.100.0.2/32, 192.168.30.2/32
```

### LAB `/etc/wireguard/wg0.conf`
```ini
[Interface]
Address = 10.100.0.2/30
PrivateKey = <LAB_PRIVATE_KEY>

[Peer]
PublicKey = <VPS_PUBLIC_KEY>
Endpoint = <VPS_PUBLIC_IP>:51820
AllowedIPs = 10.100.0.1/32
PersistentKeepalive = 25
```

```bash
chmod 600 /etc/wireguard/wg0.conf
```

## 5) Démarrage
```bash
systemctl enable --now wg-quick@wg0
wg show
```

## 6) Validation
Depuis le VPS :
```bash
ping -c 3 10.100.0.2
ping -c 3 192.168.30.2
```

Test négatif MVP :
```bash
ping -c 3 192.168.30.10
```
(doit échouer si scope strict intact)

## 7) Troubleshooting
- Port `51820/udp` non ouvert
- Clés inversées / mal copiées
- Endpoint VPS incorrect
- Horloge système/NTP incorrecte
- Interface wg0 non démarrée