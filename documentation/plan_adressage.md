| Machine | Réseau | Adresse IP | Sous-Réseau | Masque de sous réseau | 
|-------|-------|-----------|-----------|----------|
| pfSense (interface vers LAN) | LAN | 10.10.10.1 | 10.10.10.0/24 | 255.255.255.0 |
| pfSense (interface vers DMZ) | DMZ | 10.10.20.1 | 10.10.20.0/24 | 255.255.255.0 |
| pfSense (interface vers SERVEURS) | SERVEURS | 10.10.30.1 | 10.10.30.0/24 | 255.255.255.0 |
| Client| LAN | 10.10.10.10 | 10.10.10.0/24 | 255.255.255.0 |
|Serveur Nginx| DMZ| 10.10.20.10 | 10.10.20.0/24 | 255.255.255.0 |
|Serveur| SERVEURS | 10.10.30.10 | 10.10.30.0/24 | 255.255.255.0 |





