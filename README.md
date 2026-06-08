# Réseau d'Entreprise Segmenté avec Pare-feu Central (pfSense)

## 📝 Contexte du Projet
Ce projet consiste à concevoir, déployer et documenter la maquette virtuelle d'un réseau d'entreprise sécurisé et segmenté. L'objectif principal est d'isoler le réseau bureautique (LAN) des serveurs critiques, tout en exposant un serveur web de manière sécurisée via une zone démilitarisée (DMZ). L'ensemble des flux est contrôlé et filtré par un pare-feu central pfSense selon une politique de sécurité stricte.

## 🛠️ Technologies Mobilisées
* **Pare-feu / Routeur :** pfSense 2.8.1
* **Annuaire :** Windows Server 2022 (Active Directory — domaine `entreprise.local`)
* **Serveur Web :** Nginx (Ubuntu Server 24.04 LTS)
* **Accès Distant :** OpenVPN (serveur sur pfSense, UDP 1194)
* **Analyse de trafic :** Wireshark / Nmap
* **Virtualisation :** VirtualBox

---

## 📐 Architecture Réseau & Plan d'Adressage

| Zone | Sous-réseau (CIDR) | Description / Rôle | IP Passerelle (pfSense) |
| :--- | :--- | :--- | :--- |
| **WAN** | DHCP NAT (`10.0.2.x`) | Accès Internet simulé via NAT VirtualBox | `10.0.2.15` |
| **LAN** | `10.10.10.0/24` | Réseau interne — Windows Server 2022 (AD/DNS) | `10.10.10.1` |
| **DMZ** | `10.10.20.0/24` | Zone publique — Serveur Nginx | `10.10.20.1` |
| **SERVEURS (OPT2)** | `10.10.30.0/24` | Segment isolé — Serveur cible interne | `10.10.30.1` |

### Machines Virtuelles

| VM | OS | Zone | IP | Rôle |
| :--- | :--- | :--- | :--- | :--- |
| **pfSense** | pfSense 2.8.1 | WAN/LAN/DMZ/OPT2 | `10.10.10.1` / `10.10.20.1` / `10.10.30.1` | Routeur central, pare-feu, DHCP, DNS, VPN |
| **Serveur-AD-LAN** | Windows Server 2022 | LAN | `10.10.10.10` (statique) | Active Directory (`entreprise.local`), DNS, GPO |
| **Serveur-DMZ** | Ubuntu Server 24.04 | DMZ | `10.10.20.101` (DHCP) | Serveur web Nginx, SSH par clé |
| **serveur_cible** | Ubuntu Server | OPT2 | `10.10.30.10` (statique) | Serveur interne isolé |
| **VM Attaquante** *(à venir)* | Windows / Kali | WAN | DHCP NAT | Tests d'intrusion, client OpenVPN |

---

## 🔒 Politique de Sécurité — Règles pfSense

Politique **"Deny by default"** : tout ce qui n'est pas explicitement autorisé est interdit.

### WAN
| Action | Protocole | Source | Destination | Port | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ❌ Block | * | RFC 1918 | * | * | Block private networks |
| ❌ Block | * | Bogon | * | * | Block bogon networks |
| ✅ Allow | IPv4 UDP | * | WAN address | 1194 | OpenVPN — Accès distant |

### LAN
| Action | Protocole | Source | Destination | Port | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ✅ Allow | * | * | LAN Address | 443/80 | Anti-Lockout Rule |
| ✅ Allow | IPv4 | LAN subnets | * | * | Default allow LAN to any |
| ✅ Allow | IPv6 | LAN subnets | * | * | Default allow LAN IPv6 to any |

### DMZ
| Action | Protocole | Source | Destination | Port | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ❌ Block | IPv4 | DMZ subnets | `10.10.30.10` | * | Bloquer DMZ vers SERVEURS |
| ✅ Allow | IPv4 | DMZ subnets | * | * | Allow DMZ outbound |

### OPT2 (SERVEURS)
| Action | Protocole | Source | Destination | Port | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ✅ Allow | IPv4 | * | * | * | Allow OPT2 outbound |

---

## 🔐 OpenVPN — Accès Distant

| Paramètre | Valeur |
| :--- | :--- |
| **Interface** | WAN |
| **Protocole** | UDP / Port 1194 |
| **Réseau tunnel** | `10.8.0.0/24` |
| **Réseau local accessible** | `10.10.10.0/24` (LAN) |
| **Authentification** | Local User Access |
| **CA** | CA-OpenVPN |
| **Certificat serveur** | Cert-OpenVPN-Server |
| **Utilisateur VPN** | `vpnuser` (certificat : `cert-vpnuser`) |
| **Chiffrement** | AES-256-GCM / SHA256 / DH 2048 bit |

Le fichier `.ovpn` est exporté via le package `openvpn-client-export` et disponible dans `./configs/openvpn/`.

---

## 🌐 Serveur Web Nginx (DMZ)

* **IP :** `10.10.20.101`
* **Config :** `/etc/nginx/sites-enabled/default`
* **server_name :** `_` (répond à toutes les requêtes)
* **Port :** 80 (HTTP) avec redirection HTTPS (301)
* **Accessible depuis :** LAN (`10.10.10.0/24`)
* **Isolé de :** OPT2/SERVEURS (bloqué par règle pfSense)

---

## 🔑 Accès SSH par Clé

* `PasswordAuthentication no` dans `/etc/ssh/sshd_config` sur tous les serveurs Linux
* Clé générée sur machine hôte : `ssh-keygen -t ed25519 -C "projet-pfsense"`
* Clé générée sur Serveur-AD-LAN : `ssh-keygen -t ed25519 -C "projet-pfsense-windows"`
* Clés publiques déployées dans `~/.ssh/authorized_keys` sur Serveur-DMZ

---

## 🚀 Guide de Déploiement

### 1. Configuration VirtualBox

| VM | Carte 1 | Carte 2 | Carte 3 | Carte 4 |
| :--- | :--- | :--- | :--- | :--- |
| pfSense | NAT (WAN) | Réseau interne `lan-internal` | Réseau interne `dmz-internal` | Réseau interne `srv-internal` |
| Serveur-AD-LAN | Réseau interne `lan-internal` | — | — | — |
| Serveur-DMZ | Réseau interne `dmz-internal` | — | — | — |
| serveur_cible | Réseau interne `srv-internal` | — | — | — |
| VM externe | NAT (WAN) | — | — | — |

### 2. Services Réseau
* **DHCP :** pfSense — plage LAN `10.10.10.100` à `10.10.10.200`
* **DNS :** DNS Resolver pfSense (`10.10.10.1`)
* **Domaine AD :** `entreprise.local` (Windows Server 2022)

---

## 🧪 Validation & Tests

1. **Ping inter-zones** — connectivité selon politique de filtrage
2. **Isolation DMZ → SERVEURS** — captures Wireshark prouvant le rejet vers `10.10.30.10`
3. **Accès Web** — connexion HTTP depuis LAN vers Nginx DMZ
4. **OpenVPN** — connexion depuis VM attaquante (WAN) vers LAN via tunnel VPN
5. **Audit Nmap** — scan du périmètre depuis Serveur-DMZ (rapport dans `./docs/audit-nmap.md`)

---

## ⚠️ Procédure de Bascule en cas de Panne pfSense

* **Option maquette :** Sauvegarde automatisée du fichier `config.xml` de pfSense via script rsync. Restauration en moins de 15 minutes.
* **Option production :** Cluster pfSense redondant via protocole **CARP** avec synchronisation des tables d'états (pfSync).

---

## 📁 Structure du Dépôt

```
projet-1prj2/
├── docs/
│   ├── README.md
│   ├── architecture.png
│   ├── plan-ip.md
│   ├── procedure-installation.md
│   ├── procedure-restauration.md
│   └── audit-nmap.md
├── scripts/
│   ├── bash/
│   │   ├── deploy-linux.sh
│   │   └── backup.sh
│   └── powershell/
│       ├── deploy-ad.ps1
│       └── create-users.ps1
├── configs/
│   ├── pfsense/
│   │   └── config.xml
│   ├── openvpn/
│   │   └── vpnuser.ovpn
│   └── nginx/
│       └── default
└── soutenance/
    └── slides.pdf
```
