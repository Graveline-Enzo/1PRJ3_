# Réseau d'Entreprise Segmenté avec Pare-feu Central (pfSense)

## 📝 Contexte du Projet
Ce projet consiste à concevoir, déployer et documenter la maquette virtuelle d'un réseau d'entreprise sécurisé et segmenté. L'objectif principal est d'isoler le réseau bureautique (LAN) des serveurs critiques, tout en exposant un serveur web de manière sécurisée via une zone démilitarisée (DMZ). L'ensemble des flux est contrôlé et filtré par un pare-feu central pfSense selon une politique de sécurité stricte.

## 🛠️ Technologies Mobilisées
* **Pare-feu / Routeur :** pfSense
* **Serveur Web :** Nginx
* **Accès Distant :** OpenVPN
* **Analyse de trafic :** Wireshark
* **Virtualisation :** VirtualBox / VMware

---

## 📐 Architecture Réseau & Plan d'Adressage

Le réseau est découpé en 3 zones distinctes et isolées, interconnectées par le pfSense central :

| Zone | Sous-réseau (CIDR) | Description / Rôle | IP Passerelle (pfSense) |
| :--- | :--- | :--- | :--- |
| **WAN** | *Dépendant de l'hôte* | Accès à Internet (Simulé via NAT/Pont) | DHCP Hôte |
| **LAN** | `10.10.10.0/24` | Réseau bureautique interne (Clients) | `10.10.10.254` |
| **DMZ** | `10.10.20.0/24` | Zone publique exposant le serveur Nginx | `10.10.20.254` |
| **SERVEURS** | `10.10.30.0/24` | Segment critique contenant les serveurs internes | `10.10.30.254` |

### Architecture des Machines Virtuelles (5 VMs minimum)
1.  **pfSense :** Routeur central doté de 4 interfaces réseau (WAN, LAN, DMZ, SERVEURS).
2.  **Client LAN (Ubuntu/Windows) :** Machine de test située dans la zone bureautique.
3.  **Serveur Web DMZ (Ubuntu + Nginx) :** Héberge le service web accessible depuis le LAN.
4.  **Serveur Interne (Ubuntu/Windows) :** Machine cible dans la zone SERVEURS (totalement isolée de la DMZ).
5.  **Client Externe (Machine Hôte ou VM dédiée) :** Équipé d'un client OpenVPN pour simuler un accès distant.

---

## 🔒 Politique de Sécurité (Filtrage pfSense)

Le pare-feu applique une politique stricte de type **"Deny by default"** (tout ce qui n'est pas explicitement autorisé est interdit).

* **Règles de flux implémentées :**
    * Le **LAN** peut accéder au serveur Nginx en **DMZ** (HTTP/HTTPS).
    * La **DMZ** est **strictement isolée** du segment **SERVEURS**.
    * La journalisation (logging) est activée sur toutes les règles de blocage pour assurer la traçabilité des connexions refusées.

---

## 🚀 Guide de Déploiement

### 1. Configuration des interfaces sur l'hyperviseur
Pour chaque VM, configurez les cartes réseaux de la manière suivante :
* **pfSense :** Carte 1 -> NAT (WAN) | Carte 2 -> Réseau Interne `lan-internal` | Carte 3 -> Réseau Interne `dmz-internal` | Carte 4 -> Réseau Interne `srv-internal`.
* **VMs Clientes/Serveurs :** Assignez chaque VM à son réseau interne respectif (`lan-internal`, `dmz-internal` ou `srv-internal`).

### 2. Services Réseau (DHCP & DNS)
* **DHCP :** Activé et configuré sur pfSense pour la zone **LAN** (Plage : `10.10.10.50` à `10.10.10.150`).
* **DNS :** Géré par le *DNS Resolver* de pfSense avec redirection des flux et enregistrements d'hôtes locaux (Host Overrides) pour la résolution des noms du laboratoire.

### 3. Accès Distant (OpenVPN)
* Mise en place d'un serveur OpenVPN sur pfSense.
* Exportation du profil de configuration client (`.ovpn`) pour permettre l'accès distant sécurisé au segment LAN depuis l'extérieur.

---

## 🧪 Validation & Tests (Recette)

Les dossiers de capture et de preuve sont disponibles dans le répertoire `./captures` :

1.  **Vérification de l'isolation (Wireshark) :** Captures de requêtes de la DMZ vers la zone SERVEURS démontrant le rejet des paquets (Drop/Reject) conformément à la politique de sécurité.
2.  **Journalisation :** Capture d'écran du dashboard pfSense (Logs du pare-feu en temps réel) montrant les lignes de flux bloqués en rouge.
3.  **Test d'accès Web :** Succès de la connexion HTTP vers le serveur Nginx depuis le LAN.

---

## ⚠️ Procédure de Haute Disponibilité / Bascule (Failover)

En cas de panne du pare-feu central, la procédure de bascule suivante est préconisée pour assurer la continuité d'activité :
* *Option logicielle (Maquette) :* Sauvegarde régulière et automatisée du fichier `config.xml` de pfSense permettant un redéploiement rapide sur une VM clone.
* *Option physique (Production) :* Configuration de deux instances pfSense redondantes en cluster via le protocole **CARP** (Common Address Redundancy Protocol) avec synchronisation des tables d'états (pfSync).
