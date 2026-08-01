# Network Topology — Phase A

Topologie réseau réelle de la Phase A (Baseline), telle que représentée sur le schéma [`diagrams/phase-a-topology.png`](/soc-lab-repo/network/diagrams/phase-a-topology.png).

![Phase A Topology](../network/diagrams/phase-a-topology.png)

---

## 1. Zones et sous-réseaux (confirmés par le schéma)

| Zone | Sous-réseau | Machines hébergées |
|---|---|---|
| **Security_LAN** | `192.168.9.0/24` | Wazuh Manager, Velociraptor, MISP (Threat Sharing) — appliance Ubuntu Server unique |
| **Server_LAN** | `192.168.7.0/24` | AD-DC (Windows Server — `TEKUP-DC`) |
| **User_LAN** | `192.168.8.0/24` ⚠️ *(à confirmer — voir note ci-dessous)* | Win10-Client (AHMED), Win10-Client (HAROUN), CentOS client |
| **DMZ_LAN** | `192.168.6.0/24` | DVWA (Ubuntu Server) |
| **Legacy_LAN** | `192.168.11.0/24` | Metasploitable2 |
| **Attacker** | — | Kali Linux — `192.168.163.164` |

⚠️ **Note sur User_LAN :** le schéma indique `192.168.81.0/24`, alors que les rapports de simulation (Sim 3 — WannaCry) utilisent des IPs en `192.168.8.x` (ex. `192.168.8.118` pour WIN10-HAROUN). À confirmer : s'agit-il bien de `192.168.81.0/24`, ou d'une coquille dans le schéma pour `192.168.8.0/24` ?


---

## 2. Machine de management

Une **Management VM** dédiée à l'administration est  raccordée :
- une interface sur **Security_LAN** (`192.168.9.0/24`) — pour gérer Wazuh, Velociraptor et MISP


---

## 4. Flux d'agents SOC (Security_LAN → toutes les zones)

L'appliance Security_LAN (Wazuh Manager + Velociraptor + MISP) communique avec les agents déployés sur `Server_LAN`, `User_LAN` et `DMZ_LAN` via les ports suivants :

| Port | Service | Direction |
|---|---|---|
| `1514/1515` | Wazuh (agents → manager) | Server_LAN, User_LAN, DMZ_LAN → Security_LAN |
| `8000` | Velociraptor (agents → serveur) | Server_LAN, User_LAN, DMZ_LAN → Security_LAN |

**Note :** aucune ligne d'agent n'est représentée vers `Legacy_LAN` — cohérent avec l'absence volontaire d'agent Wazuh/Velociraptor sur Metasploitable2 (gap de conception assumé, voir Simulation 2).

---

## 5. Interfaces OPNsense

OPNsense centralise le routage entre l'Attacker Zone (Kali) et toutes les zones internes. Chaque zone est raccordée à une interface dédiée d'OPNsense (Security_LAN, User_LAN, DMZ_LAN, Legacy_LAN, Server_LAN via la Management VM).

📄 Détail des règles de pare-feu par zone : [`opnsense-firewall-rules.md`](../network/opnsense-firewall-rules.md) *(à venir — captures nécessaires)*
