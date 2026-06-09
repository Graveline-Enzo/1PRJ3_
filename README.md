# Réseau d'Entreprise Segmenté avec Pare-feu Central (pfSense)

## 📝 Contexte du Projet
Ce projet consiste à concevoir, déployer et documenter la maquette virtuelle d'un réseau d'entreprise sécurisé et segmenté. L'objectif principal est d'isoler le réseau bureautique (LAN) des serveurs critiques, tout en exposant un serveur web de manière sécurisée via une zone démilitarisée (DMZ). L'ensemble des flux est contrôlé et filtré par un pare-feu central pfSense selon une politique de sécurité stricte.

## 🛠️ Technologies Mobilisées
* **Pare-feu / Routeur :** pfSense 2.8.1
* **Annuaire :** Windows Server 2022 (Active Directory — domaine `entreprise.local`)
* **Serveur Web :** Nginx (Ubuntu Server 24.04 LTS)
* **Accès Distant :** OpenVPN (serveur sur pfSense, UDP 1194)
* **Analyse de trafic :** Wireshark / Nmap / Nikto
* **Tests d'intrusion :** Hydra, Ettercap (Kali Linux)
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
| **Serveur-DMZ** | Ubuntu Server 24.04 | DMZ | `10.10.20.101` (statique) | Serveur web Nginx, SSH durci (port 4520) |
| **serveur_cible** | Ubuntu Server | OPT2 | `10.10.30.10` (statique) | Serveur interne isolé |
| **VM Attaquante** | Kali Linux | LAN | `10.10.10.102` | Tests d'intrusion |

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
* **Port :** 80 (HTTP) → redirection HTTPS 301
* **HTTPS :** port 443, certificat auto-signé, TLS AES-256-GCM-SHA384
* **SSH :** port 4520 (non-standard), clé uniquement, `PermitRootLogin no`
* **Accessible depuis :** LAN (`10.10.10.0/24`) sur ports 80/443
* **SSH bloqué depuis LAN :** port 4520 filtré par pfSense ✅

### Headers de sécurité HTTP (ajoutés J4)
```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Referrer-Policy "no-referrer" always;
add_header Content-Security-Policy "default-src 'self'" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
add_header X-Frame-Options "SAMEORIGIN" always;
```

---

## 🛡️ Durcissement Serveur-DMZ (Ubuntu 24.04)

| Mesure | Détail | Statut |
| :--- | :--- | :--- |
| UFW actif | Ports 80, 443, 4520 uniquement | ✅ |
| fail2ban | jail sshd — port 4520, bantime 3600s, maxretry 3 | ✅ |
| SSH durci | Port 4520, PermitRootLogin no, X11Forwarding no, MaxAuthTries 3 | ✅ |
| Sysctl | accept_redirects=0, send_redirects=0, syncookies=1, suid_dumpable=0 | ✅ |
| rkhunter | 0 rootkit détecté | ✅ |
| auditd | Actif | ✅ |
| arpwatch | Protection ARP Poisoning | ✅ |
| Backup rsync | Script `backup.sh` — cron toutes les 2h | ✅ |
| Password policy | pwquality + login.defs | ✅ |
| Protocoles blacklistés | dccp, sctp, rds, tipc, usb-storage | ✅ |
| Services désactivés | ModemManager, unattended-upgrades | ✅ |
| Banners légaux | `/etc/issue` configuré | ✅ |
| Hostname | `serveur-dmz.entreprise.local` | ✅ |

### Score Lynis

| Étape | Score |
| :--- | :--- |
| Avant durcissement | 57 / 100 |
| Après durcissement | **74 / 100** (+17 pts) |

---

## 🔑 Accès SSH par Clé

* `PasswordAuthentication no` dans `/etc/ssh/sshd_config` sur tous les serveurs Linux
* Clé générée sur machine hôte : `ssh-keygen -t ed25519 -C "projet-pfsense"`
* Clé générée sur Serveur-AD-LAN : `ssh-keygen -t ed25519 -C "projet-pfsense-windows"`
* Clés publiques déployées dans `~/.ssh/authorized_keys` sur Serveur-DMZ

---

## 🧪 Tests d'Intrusion — Résultats J4

Tous les tests ont été réalisés depuis la VM Kali (`10.10.10.102`).

| Test | Outil | Résultat | Conclusion |
| :--- | :--- | :--- | :--- |
| Scan réseau LAN + DMZ | Nmap | SSH port 4520 invisible depuis LAN | pfSense filtre correctement ✅ |
| Brute-force SSH | Hydra | Connection refused | pfSense bloque le port 4520 depuis LAN ✅ |
| Scan vulnérabilités web | Nikto | 5 headers manquants (corrigés), pas de faille critique | Remédiation appliquée ✅ |
| ARP Poisoning | Ettercap | Empoisonnement actif, sysctl protège le Serveur-DMZ | accept_redirects=0, send_redirects=0 ✅ |

### Détail scan Nikto (avant / après remédiation)
| ID | Problème | Avant | Après |
| :--- | :--- | :--- | :--- |
| 999966 | BREACH attack (Content-Encoding: deflate) | ⚠️ Présent | ⚠️ Résiduel (très faible risque) |
| 013587 | X-Content-Type-Options manquant | ❌ | ✅ Corrigé |
| 013587 | Strict-Transport-Security manquant | ❌ | ✅ Corrigé |
| 013587 | Referrer-Policy manquant | ❌ | ✅ Corrigé |
| 013587 | Content-Security-Policy manquant | ❌ | ✅ Corrigé |
| 013587 | Permissions-Policy manquant | ❌ | ✅ Corrigé |

---

## ⚠️ Failles Résiduelles Connues

| Faille | Criticité | Justification / Plan |
| :--- | :--- | :--- |
| Certificat SSL auto-signé | Faible | Acceptable en maquette lab — Let's Encrypt nécessite un domaine public |
| Pas de mot de passe GRUB | Faible | Accès physique impossible en production virtualisée |
| Logs non centralisés (pas de syslog externe) | Moyenne | Hors périmètre du projet — amélioration possible avec un SIEM |
| Partitions /home /tmp /var non séparées | Faible | Acceptable en maquette — séparation recommandée en production |
| Pas de DAI contre ARP Poisoning sur pfSense | Moyenne | pfSense ne supporte pas DAI nativement — arpwatch compense |
| BREACH attack résiduelle (Nikto) | Très faible | Exploitation pratique quasi-impossible sans conditions très spécifiques |

---

## 🚀 Guide de Déploiement

### 1. Configuration VirtualBox

| VM | Carte 1 | Carte 2 | Carte 3 | Carte 4 |
| :--- | :--- | :--- | :--- | :--- |
| pfSense | NAT (WAN) | Réseau interne `lan-internal` | Réseau interne `dmz-internal` | Réseau interne `srv-internal` |
| Serveur-AD-LAN | Réseau interne `lan-internal` | — | — | — |
| Serveur-DMZ | Réseau interne `dmz-internal` | — | — | — |
| serveur_cible | Réseau interne `srv-internal` | — | — | — |
| VM Kali | Réseau interne `lan-internal` | — | — | — |

### 2. Services Réseau
* **DHCP :** pfSense — plage LAN `10.10.10.100` à `10.10.10.200`
* **DNS :** DNS Resolver pfSense (`10.10.10.1`)
* **Domaine AD :** `entreprise.local` (Windows Server 2022)

---

## 📦 Snapshots VirtualBox (J4 Final)

| VM | Nom du snapshot |
| :--- | :--- |
| Serveur-DMZ | `J4-Final-Durci` |
| Serveur-AD-LAN | `J4-Final-AD` |
| pfSense | `J4-Final-pfSense` |
| VM-Kali | `J4-Final-Kali` |

---

## ⚙️ Procédure de Bascule en cas de Panne pfSense

* **Option maquette :** Sauvegarde automatisée du fichier `config.xml` de pfSense via script rsync. Restauration en moins de 15 minutes.
* **Option production :** Cluster pfSense redondant via protocole **CARP** avec synchronisation des tables d'états (pfSync).

---
# Audit Nmap — Serveur-DMZ (10.10.20.101)

**Date :** J4  
**Source :** VM Kali (`enzo@kali-externe` — `10.10.10.102`)  
**Commande :** `nmap -sV 10.10.20.101`  
**Durée :** 32.42 secondes (256 IP scannées, 2 hôtes actifs)

---

## Résultats

| Port | État | Service | Version |
| :--- | :--- | :--- | :--- |
| 80/tcp | open | http | nginx 1.24.0 (Ubuntu) |
| 443/tcp | open | ssl/http | nginx 1.24.0 (Ubuntu) |
| 998 autres ports | filtered | — | no-response |

---

## Analyse

### Points positifs ✅
- **998 ports filtrés** — pfSense applique correctement la politique "deny by default"
- **Port 4520 (SSH) invisible** depuis le LAN — règle pfSense efficace
- **Nginx 1.24.0** — version récente, pas de CVE critiques connues
- **TLS actif** sur le port 443 — chiffrement AES-256-GCM-SHA384
- Surface d'attaque minimale : seulement 2 ports exposés

### Avertissements Nmap
- OS detection peu fiable (pas d'1 port ouvert + 1 port fermé) — normal car pfSense filtre agressivement
- Guesses OS : Linux 4.x/5.x (97%) — cohérent avec Ubuntu Server 24.04

---

## Conclusion

Le périmètre exposé est conforme à la politique de sécurité définie :  
seuls les ports 80 (redirection HTTPS) et 443 (Nginx) sont accessibles depuis le LAN.  
Le SSH (port 4520) est correctement masqué par pfSense.
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

---
