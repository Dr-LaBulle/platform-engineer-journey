# Runbook — Bootstrap WireGuard site-to-site (VPS ⇄ `wg-lab`)

## Objectif
Mettre en place un tunnel **WireGuard** entre :
- **VPS (Suisse)** : endpoint public (écoute `51820/udp`), IP WG `10.100.0.1/30`
- **Lab (Portugal)** : CT LXC Debian `wg-lab` (DMZ `192.168.30.2`), IP WG `10.100.0.2/30`

**Scope MVP (strict, D=2)** : le VPS doit pouvoir atteindre **uniquement** `wg-lab` (`192.168.30.2`) via le tunnel (pas de routage DMZ complète).

Référence design contractuel : `platform/network/wireguard.md`

---

## Pré-requis
- Accès root (ou sudo) sur le VPS.
- Accès à Proxmox pour créer/configurer un CT LXC.
- Le VPS possède une IP publique et un pare-feu contrôlable.
- Le lab peut initier des connexions sortantes UDP vers Internet (5G).

---

## 1) Préparation VPS (packages, firewall, sysctl)

### 1.1 Installer WireGuard
Sur le VPS (Debian/Ubuntu) :
- Installer `wireguard` / `wireguard-tools` (selon distro)

Exemples (à adapter) :
- Debian/Ubuntu : `apt update && apt install -y wireguard wireguard-tools`

### 1.2 Ouvrir le port WireGuard
Sur le pare-feu du VPS :
- Autoriser `51820/udp`
- (Optionnel mais typique) Autoriser `80/tcp` et `443/tcp` si reverse proxy sur VPS
- SSH : restreindre au strict nécessaire

> La mise en œuvre dépend de l’outil (ufw/nftables/iptables). Documenter la commande exacte utilisée dans ce runbook lors de l’implémentation.

### 1.3 Activer l’IP forwarding (sysctl)
Même en scope strict (un seul hôte), activer le forwarding simplifie les évolutions futures.

- Vérifier : `sysctl net.ipv4.ip_forward`
- Activer temporairement : `sysctl -w net.ipv4.ip_forward=1`
- Rendre persistant (ex.) : créer un fichier sous `/etc/sysctl.d/` (ex. `99-wireguard.conf`) contenant :
  - `net.ipv4.ip_forward=1`
Puis :
- `sysctl --system`

---

## 2) Création du CT LXC `wg-lab` (Debian, IP 192.168.30.2, autostart)

### 2.1 Créer le CT
Sur Proxmox :
- Créer un CT Debian minimal (recommandé : Debian 12)
- Nom logique : `wg-lab`
- Ressources : 1 vCPU, 512MB–1GB RAM
- Disque : 4–8GB (largement suffisant)
- Réseau :
  - VLAN : **30 (DMZ)**
  - IPv4 fixe : **192.168.30.2/24**
  - Gateway : **IP passerelle DMZ sur l’UDM Pro** (ex. `192.168.30.1` si c’est ton plan)

### 2.2 Activer l’autostart
- Activer le démarrage automatique du CT au boot Proxmox.
- (Optionnel) Définir un ordre/priorité de démarrage si tu as d’autres services dépendants.

### 2.3 Vérifier la connectivité de base
Dans le CT :
- Vérifier IP : `ip a`
- Vérifier route : `ip r`
- Test DNS/ping (ex.) :
  - `ping -c 3 1.1.1.1`
  - `ping -c 3 google.com`

---

## 3) Génération des clés (VPS + `wg-lab`)

> Ne jamais commiter les clés privées dans Git.

### 3.1 Sur le VPS
Créer les clés :
- `wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey`
Permissions :
- `chmod 600 /etc/wireguard/privatekey`

Récupérer la clé publique du VPS :
- `cat /etc/wireguard/publickey`

### 3.2 Sur `wg-lab`
Créer les clés :
- `wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey`
Permissions :
- `chmod 600 /etc/wireguard/privatekey`

Récupérer la clé publique de `wg-lab` :
- `cat /etc/wireguard/publickey`

---

## 4) Création des fichiers `wg0.conf` (VPS et `wg-lab`) avec placeholders

### 4.1 Variables à renseigner (sans les commiter)
- `<VPS_PUBLIC_IP>` : IP publique du VPS
- `<VPS_PRIVATE_KEY>` : contenu de `/etc/wireguard/privatekey` sur le VPS
- `<VPS_PUBLIC_KEY>` : contenu de `/etc/wireguard/publickey` sur le VPS
- `<LAB_PRIVATE_KEY>` : contenu de `/etc/wireguard/privatekey` sur `wg-lab`
- `<LAB_PUBLIC_KEY>` : contenu de `/etc/wireguard/publickey` sur `wg-lab`

### 4.2 Configuration VPS : `/etc/wireguard/wg0.conf`
Créer/éditer ce fichier sur le VPS :

```ini
[Interface]
Address = 10.100.0.1/30
ListenPort = 51820
PrivateKey = <VPS_PRIVATE_KEY>

# (Optionnel) durcissement: limiter l'output de logs si nécessaire
# SaveConfig = false

[Peer]
PublicKey = <LAB_PUBLIC_KEY>

# Scope MVP D=2 (strict):
# - joindre l'IP WG du peer
# - joindre uniquement l'hôte wg-lab en DMZ
AllowedIPs = 10.100.0.2/32, 192.168.30.2/32
```

### 4.3 Configuration `wg-lab` : `/etc/wireguard/wg0.conf`
Créer/éditer ce fichier sur `wg-lab` :

```ini
[Interface]
Address = 10.100.0.2/30
PrivateKey = <LAB_PRIVATE_KEY>

# Important sur 5G/CGNAT : maintenir l'état NAT côté opérateur
# (valeur typique : 25s)
# NOTE: côté "client" (ici wg-lab), on configure le keepalive dans [Peer]

[Peer]
PublicKey = <VPS_PUBLIC_KEY>
Endpoint = <VPS_PUBLIC_IP>:51820

# On n'annonce rien de plus que l'IP WG du VPS au MVP.
AllowedIPs = 10.100.0.1/32

PersistentKeepalive = 25
```

Permissions :
- `chmod 600 /etc/wireguard/wg0.conf`

---

## 5) Démarrage et tests (`wg show`, ping)

### 5.1 Démarrer WireGuard
Sur le VPS :
- `systemctl enable --now wg-quick@wg0`

Sur `wg-lab` :
- `systemctl enable --now wg-quick@wg0`

### 5.2 Vérifier l’état
Sur les deux :
- `wg show`
Attendus :
- un handshake récent
- bytes rx/tx qui évoluent

Vérifier interfaces/routes :
- `ip a show wg0`
- `ip r`

### 5.3 Tests de connectivité (MVP)
Depuis le VPS :
1) Ping IP WireGuard du lab :
   - `ping -c 3 10.100.0.2`
2) Ping IP DMZ de `wg-lab` :
   - `ping -c 3 192.168.30.2`

Test négatif (scope strict) :
- `ping -c 3 192.168.30.10` doit **échouer** (ou au minimum ne pas être routé) tant que tu n’as pas étendu `AllowedIPs`.

---

## Dépannage rapide (si pas de handshake)
1) Vérifier port UDP ouvert côté VPS : `51820/udp`.
2) Vérifier que `wg-lab` peut joindre `<VPS_PUBLIC_IP>:51820` (sortant UDP autorisé).
3) Vérifier les clés publiques/privées (erreur de copier/coller très fréquente).
4) Vérifier l’heure système (NTP) sur VPS et `wg-lab`.
5) Vérifier l’interface `wg0` up : `systemctl status wg-quick@wg0`.

---

## Notes de sécurité
- Ne pas commiter de clés privées.
- Restreindre SSH sur le VPS.
- Conserver le scope strict tant que tu n’as pas documenté les règles de filtrage et les besoins d’exposition.