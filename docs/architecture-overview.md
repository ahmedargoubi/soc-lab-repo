# Architecture Overview

Vue d'ensemble de l'architecture complète du lab — hyperviseur, zones réseau,
machines, flux de trafic, et empilement d'outils de sécurité déployés sur
Security_LAN.

![Architecture Phase B](../network/diagrams/phase-b-topology.jpg)

---

## 1. Hyperviseur

Le lab entier tourne sur une seule machine physique (laptop), sous
**VMware Workstation Pro**. Toutes les VMs listées ci-dessous sont des
machines virtuelles rattachées à des `VMnet` dédiés, chacun mappé sur une
interface réseau distincte d'OPNsense.

![Architecture Phase B](screens/vmware.png)

---

## 2. Pare-feu central — OPNsense

OPNsense est le point de passage obligé entre la zone Attaquant (Kali,
techniquement sur le WAN d'OPNsense dans ce lab) et l'ensemble des zones
internes. Chaque zone est raccordée à une interface dédiée, et le
filtrage fonctionne en **first-match** : la première règle qui correspond
à un paquet s'applique, tout le reste est bloqué par défaut.

- **Accès admin GUI OPNsense** : protégé par **2FA (TOTP natif OPNsense)**
  — voir [`config/opnsense-mfa.md`](../reports/phase-b-hardening/config/opnsense-mfa.md).
- Détail des règles de pare-feu par interface :
  [`network/opnsense-firewall-rules.md`](../network/opnsense-firewall-rules.md)
  (baseline Phase A) et
  [`network/opnsense-firewall-rules-post-hardening.md`](../network/opnsense-firewall-rules-post-hardening.md)
  (Phase B, à venir).

---

## 3. Zones réseau

| Zone | Rôle | Machines / services |
|---|---|---|
| **Attacker** | Poste d'attaque, hors zones internes (NAT VMware, vu comme WAN par OPNsense) | Kali Linux — `192.168.163.164` |
| **DMZ_LAN** | Service exposé, cible volontaire | DVWA ("DMZ Web page") — `192.168.11.177` |
| **Legacy_LAN** | Machine volontairement vulnérable, non supervisée | Metasploitable2 — `192.168.6.144` |
| **Security_LAN** | Cœur SOC — détection, threat intel, SOAR, case management | Voir détail Section 4 — `192.168.9.0/24` |
| **Server_LAN** | Infrastructure serveur interne | AD-DC / `TEKUP-DC` (Windows Server) — `192.168.7.139` |
| **User_LAN** | Postes utilisateurs | Win10-Client (AHMED), Win10-Client (HAROUN) — `192.168.8.*` ; client CentOS (`node1`) — `192.168.8.127` |


---

## 4. Security_LAN — détail de la stack SOC

Security_LAN (`192.168.9.0/24`) concentre l'ensemble de la chaîne
détection → réponse → investigation → renseignement :

| Composant | Rôle | Adresse |
|---|---|---|
| **WAF** (SafeLine) | Filtrage applicatif devant DVWA — bloque et affiche "Access Forbidden / Blocked For Attack Detected" sur détection d'attaque | Security_LAN |
| **Wazuh Manager** | SIEM — détection, corrélation de règles, déclenchement des webhooks vers Shuffle | `192.168.9.133` |
| **Velociraptor** | DFIR — collecte forensique à la demande sur les agents | `192.168.9.133:8889` |
| **MISP** | Threat Intelligence — partage et corrélation d'IOCs avec Wazuh | Security_LAN |
| **VirusTotal** | Enrichissement — vérification de hashs/IOCs via l'intégration Wazuh↔VirusTotal | Security_LAN |
| **Shuffle** | SOAR — orchestre la réponse automatisée (blocage IP, notification, création de case) | `192.168.9.144:3001` |
| **TheHive** | Case management — transforme les alertes qualifiées en dossiers d'investigation suivis | `192.168.9.133` |


Le flux logique à l'intérieur de Security_LAN :

```
Wazuh Manager ──┬──> MISP (corrélation IOC)
                ├──> VirusTotal (enrichissement)
                ├──> TheHive (alerte → case, si règle qualifiante)
                └──> Shuffle (webhook → workflow automatisé)
                          │
                          ├──> Blocage actif de l'IP attaquante (active-response)
                          └──> Email (notification analyste)
```

Voir [`reports/phase-b-hardening/config/phase-b-brute-force-blocking.md`](../reports/phase-b-hardening/config/phase-b-brute-force-blocking.md)
pour le détail complet, testé et validé, de cette chaîne pour la détection
et le blocage automatique d'une attaque par force brute SSH.

---

## 5. Poste de management

Une **Management Workstation** dédiée (`192.168.9.150`) est raccordée à
Security_LAN pour l'administration de la stack SOC (Wazuh, Velociraptor,
MISP, TheHive, Shuffle).

L'accès **SSH** vers Security-Core (`192.168.9.133:22`) est protégé par
**2FA** (`libpam-google-authenticator`) — voir
[`config/security-core-ssh-mfa.md`](../reports/phase-b-hardening/config/security-core-ssh-mfa.md).

---

## 6. Flux d'agents SOC

Les agents Wazuh et Velociraptor déployés sur `Server_LAN`, `User_LAN` et
`DMZ_LAN` communiquent vers Security_LAN via :

| Port | Service | Direction |
|---|---|---|
| `1514/1515` | Wazuh (agent → manager) | Server_LAN, User_LAN, DMZ_LAN → Security_LAN |
| `8000` | Velociraptor (agent → serveur) | Server_LAN, User_LAN, DMZ_LAN → Security_LAN |

Aucun agent n'est déployé sur `Legacy_LAN` — gap de conception volontaire,
voir [Simulation 2](../reports/phase-a-baseline/simulation-2-metasploitable-pivot.md).

---

## 7. Ce qui a changé entre Phase A et Phase B

| Élément | Phase A (baseline) | Phase B (durcissement) |
|---|---|---|
| Accès admin OPNsense | Sans MFA | 2FA (TOTP natif) |
| Accès SSH Security-Core | Sans MFA | 2FA (`libpam-google-authenticator`) |
| Détection ↔ Threat Intel | Wazuh seul | Wazuh ↔ MISP, Wazuh ↔ VirusTotal |
| Réponse à incident | Manuelle | Automatisée (Shuffle SOAR) pour les alertes qualifiantes |
| Suivi d'incident | Aucun outil dédié | TheHive (case management structuré) |
| Protection applicative DMZ | Aucune | WAF (SafeLine) devant DVWA |

Pour le détail exhaustif de l'avancement Phase B, voir
[`reports/phase-b-hardening/README.md`](../reports/phase-b-hardening/README.md).
