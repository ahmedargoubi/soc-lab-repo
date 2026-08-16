# Network Topology

## Phase A — Baseline

Topologie réseau réelle de la Phase A (Baseline), telle que représentée sur le schéma [`diagrams/phase-a-topology.png`](../network/diagrams/phase-a-topology.png).

![Phase A Topology](../network/diagrams/phase-a-topology.png)

---

### 1. Zones et sous-réseaux (confirmés par le schéma)

| Zone | Sous-réseau | Machines hébergées |
|---|---|---|
| **Security_LAN** | `192.168.9.0/24` | Wazuh Manager, Velociraptor, MISP (Threat Sharing) — appliance Ubuntu Server unique |
| **Server_LAN** | `192.168.7.0/24` | AD-DC (Windows Server — `TEKUP-DC`) |
| **User_LAN** | `192.168.8.0/24`  | Win10-Client (AHMED), Win10-Client (HAROUN), CentOS client |
| **DMZ_LAN** | `192.168.6.0/24` | DVWA (Ubuntu Server) |
| **Legacy_LAN** | `192.168.11.0/24` | Metasploitable2 |
| **Attacker** | — | Kali Linux — `192.168.163.164` |

⚠️ **Note sur User_LAN :** le schéma indique `192.168.81.0/24`, alors que les rapports de simulation (Sim 3 — WannaCry) utilisent des IPs en `192.168.8.x` (ex. `192.168.8.118` pour WIN10-HAROUN). À confirmer : s'agit-il bien de `192.168.81.0/24`, ou d'une coquille dans le schéma pour `192.168.8.0/24` ?

⚠️ **Note DMZ / Legacy — à réconcilier avec la Phase B :** le schéma Phase B
(section suivante) assigne `192.168.11.0/24` à **DMZ_LAN** (DVWA à
`.11.177`) et `192.168.6.0/24` à **Legacy_LAN** (Metasploitable2 à
`.6.144`) — soit l'inverse exact de la table ci-dessus. Cela correspond à
l'incohérence déjà relevée dans
[`opnsense-firewall-rules.md`](../network/opnsense-firewall-rules.md)
(règle `WAN net → 192.168.11.197/32` apparaissant sous l'interface
`dmz`). Il est probable que l'assignation Phase B soit la version
corrigée/réelle et que cette table Phase A doive être mise à jour en
conséquence — **à confirmer avant correction rétroactive**, pour ne pas
réécrire l'historique sans certitude (voir principe de transparence dans
[`security-principles.md`](security-principles.md)).

---

### 2. Machine de management

Une **Management VM** dédiée à l'administration est  raccordée :
- une interface sur **Security_LAN** (`192.168.9.0/24`) — pour gérer Wazuh, Velociraptor et MISP


---

### 3. Flux d'agents SOC (Security_LAN → toutes les zones)

L'appliance Security_LAN (Wazuh Manager + Velociraptor + MISP) communique avec les agents déployés sur `Server_LAN`, `User_LAN` et `DMZ_LAN` via les ports suivants :

| Port | Service | Direction |
|---|---|---|
| `1514/1515` | Wazuh (agents → manager) | Server_LAN, User_LAN, DMZ_LAN → Security_LAN |
| `8000` | Velociraptor (agents → serveur) | Server_LAN, User_LAN, DMZ_LAN → Security_LAN |

**Note :** aucune ligne d'agent n'est représentée vers `Legacy_LAN` — cohérent avec l'absence volontaire d'agent Wazuh/Velociraptor sur Metasploitable2 (gap de conception assumé, voir Simulation 2).

---

### 4. Interfaces OPNsense

OPNsense centralise le routage entre l'Attacker Zone (Kali) et toutes les zones internes. Chaque zone est raccordée à une interface dédiée d'OPNsense (Security_LAN, User_LAN, DMZ_LAN, Legacy_LAN, Server_LAN via la Management VM).

📄 Détail des règles de pare-feu par zone : [`opnsense-firewall-rules.md`](../network/opnsense-firewall-rules.md) *(baseline Phase A)*

---

## Phase B — Durcissement

Topologie mise à jour après le durcissement et l'ajout des capacités
d'automatisation de la Phase B, telle que représentée sur le schéma
[`diagrams/phase-b-topology.jpg`](../network/diagrams/phase-b-topology.jpg).

![Phase B Topology](../network/diagrams/phase-b-topology.jpg)

### 1. Ce qui a changé par rapport à la Phase A

| Ajout / changement | Détail |
|---|---|
| **2FA sur OPNsense (admin GUI)** | TOTP natif OPNsense — voir [`config/opnsense-mfa.md`](../config/opnsense-mfa.md) |
| **2FA sur l'accès SSH Security-Core** | `192.168.9.133:22`, `libpam-google-authenticator` — voir [`config/security-core-ssh-mfa.md`](../config/security-core-ssh-mfa.md) |
| **WAF devant la DMZ** | Bloque activement les attaques HTTP contre DVWA (message "Access Forbidden / Blocked For Attack Detected" visible sur le schéma) |
| **TheHive** | Case management, `192.168.9.133` — ajouté à Security_LAN |
| **Shuffle** | SOAR, `192.168.9.144:3001` — orchestre la réponse automatisée |
| **VirusTotal** | Intégration d'enrichissement avec Wazuh |
| **Notification email** | Alerte automatique à l'analyste SOC dès qu'un blocage est déclenché |
| **Management Workstation** | `192.168.9.150`, raccordée à Security_LAN |

Détail complet de l'architecture Security_LAN et des flux entre ces
composants : [`docs/architecture-overview.md`](architecture-overview.md).

### 2. Zones et sous-réseaux (Phase B, selon ce schéma)

| Zone | Sous-réseau | Machines / services |
|---|---|---|
| **Attacker** | — | Kali Linux — `192.168.163.164` |
| **DMZ_LAN** | `192.168.11.0/24` | DVWA ("DMZ Web page") — `192.168.11.177` |
| **Legacy_LAN** | `192.168.6.0/24` | Metasploitable2 — `192.168.6.144` |
| **Security_LAN** | `192.168.9.0/24` | WAF, Velociraptor (`.9.133:8889`), MISP, TheHive, Wazuh Manager, Shuffle (`192.168.9.144:3001`), VirusTotal, Management Workstation (`.9.150`) |
| **Server_LAN** | `192.168.7.0/24`* | AD-DC / `TEKUP-DC` — `192.168.7.139` |
| **User_LAN** | `192.168.8.0/24` | Win10-Client (AHMED), Win10-Client (HAROUN) — `192.168.8.*` ; client CentOS (`node1`) — `192.168.8.127` |

\* *Le schéma affiche `192.168.8.0/24` dans l'encart Server_LAN, ce qui
entre en conflit avec l'IP réelle de l'AD-DC (`192.168.7.139`) —
probablement une coquille de libellé plutôt qu'un changement de
sous-réseau réel. À confirmer.*

📄 Détail des règles de pare-feu post-durcissement :
[`opnsense-firewall-rules-post-hardening.md`](../network/opnsense-firewall-rules-post-hardening.md)
*(à venir)*
