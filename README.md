# Enterprise SOC / Red Team / DFIR / CTI / GRC Lab

> Simulation d'un réseau d'entreprise segmenté, attaqué et défendu à travers 5 rôles de cybersécurité successifs (Red Team, SOC Analyst, DFIR, CTI, GRC) sur la même infrastructure virtualisée.

Inspiré par des dépôts portfolio comme [Lab4PurpleSec](https://github.com/0xMR007/Lab4PurpleSec).

---

## 📌 Statut du projet

| Phase | Description | Statut |
|---|---|---|
| **Phase A — Baseline** | Attaques contre l'infrastructure telle que construite initialement, sans durcissement | ✅ Terminée (4/4 simulations + synthèse) |
| **Phase B — Durcissement + Automatisation** | MFA (privacyIDEA), SOAR (Shuffle), durcissement AD, règles Wazuh/Suricata personnalisées | ⏳ À venir |
| **Phase C — Ré-attaque** | Rejeu des mêmes simulations contre le réseau durci, comparatif avant/après | ⏳ À venir |

---

## 🏗️ Architecture

- **Hyperviseur** : VMware Workstation Pro (laptop unique, Windows)
- **Pare-feu central** : OPNsense — 6 interfaces réseau, une par zone
- **Zones réseau** :

| Zone | Rôle | Machines hébergées |
|---|---|---|
| **Security_LAN** | Outils de sécurité | Security-Core (Wazuh + MISP + Velociraptor), Ansible/Control |
| **DMZ** | Services exposés | DMZ-Web (Apache + DVWA) |
| **User_LAN** | Postes utilisateurs | 2 clients Windows 10 (AHMED, HAROUN), 1 client CentOS (node1) |
| **Server_LAN** | Serveurs internes | AD-DC (Windows Server 2022 — domaine `PROJET.local`, hostname `TEKUP-DC`) |
| **Legacy** | Machines volontairement vulnérables, non supervisées | Metasploitable2 |
| **Attacker** | Poste d'attaque (NAT VMware) | Kali Linux |

📄 Détails complets : [`docs/architecture-overview.md`](docs/architecture-overview.md) · [`docs/network-topology.md`](docs/network-topology.md)

**Note :** certaines machines ont changé de sous-réseau plusieurs fois au cours du projet à cause de bugs de configuration VMware/OPNsense. Les IPs réelles utilisées dans chaque simulation sont documentées dans les rapports individuels et dans le [journal de dépannage](docs/troubleshooting-journal.md) — elles ne sont pas rétroactivement uniformisées.

---

## 🛠️ Stack de sécurité

| Outil | Rôle | Déploiement |
|---|---|---|
| **Wazuh** | SIEM | Manager sur Security-Core, agents sur AD-DC, DMZ-Web, clients Windows/CentOS |
| **Velociraptor** | DFIR | Installation native sur Security-Core, agents sur les mêmes cibles |
| **MISP** | Threat Intelligence | Docker sur Security-Core |
| **Suricata** | NIDS | Interfaces WAN, DMZ et Security_LAN d'OPNsense |
| **Kali Toolset** | Red Team | Metasploit, nmap, sqlmap, Responder, impacket, SharpHound/BloodHound, John the Ripper |

📄 Guides d'installation/configuration détaillés : [`config/`](config/)

---

## 🎭 Les 5 rôles simulés

```
Red Team  →  SOC Analyst (Wazuh)  →  DFIR (Velociraptor)  →  CTI (MISP + MITRE ATT&CK)  →  GRC (NIST CSF 2.0)
```

Chaque simulation d'attaque est rejouée à travers ces 5 angles d'analyse, produisant un rapport unique qui couvre l'attaque, la détection, l'investigation, le renseignement sur la menace et l'évaluation de conformité.

---

## 📂 Simulations — Phase A (Baseline)

| # | Simulation | Zone cible | Rapport |
|---|---|---|---|
| 1 | DVWA — SQL Injection & Command Injection | DMZ | [`simulation-1-dvwa.md`](reports/phase-a-baseline/simulation-1-dvwa.md) |
| 2 | Metasploitable2 — Backdoor & pivot test | Legacy | [`simulation-2-metasploitable-pivot.md`](reports/phase-a-baseline/simulation-2-metasploitable-pivot.md) |
| 3 | WannaCry — Exécution de ransomware réel | User_LAN | [`simulation-3-wannacry-ransomware.md`](reports/phase-a-baseline/simulation-3-wannacry-ransomware.md) |
| 4 | Active Directory — LLMNR Poisoning, Pass-the-Hash, BloodHound | Server_LAN | [`simulation-4-active-directory.md`](reports/phase-a-baseline/simulation-4-active-directory.md) |

📊 Comparatif des 4 simulations : [`phase-a-summary.md`](reports/phase-a-baseline/phase-a-summary.md)

---

## 🗺️ Structure du dépôt

```
/
├── README.md
├── docs/
│   ├── architecture-overview.md
│   ├── network-topology.md
│   ├── security-principles.md
│   ├── troubleshooting-journal.md
│   └── roadmap.md
├── config/
│   ├── README.md
│   ├── opnsense.md
│   ├── wazuh.md
│   ├── velociraptor.md
│   ├── misp.md
│   └── suricata.md
├── network/
│   ├── opnsense-firewall-rules.md
│   └── diagrams/
├── reports/
│   ├── phase-a-baseline/
│   │   ├── simulation-1-dvwa.md
│   │   ├── simulation-2-metasploitable-pivot.md
│   │   ├── simulation-3-wannacry-ransomware.md
│   │   ├── simulation-4-active-directory.md
│   │   └── phase-a-summary.md
│   ├── phase-b-hardening/      (à venir)
│   └── phase-c-reattack/       (à venir)
└── scripts/
    ├── wazuh/
    └── velociraptor/
```

---

## 🗺️ Roadmap

Voir [`docs/roadmap.md`](docs/roadmap.md) pour le détail des trois phases et leur avancement.

---

*Projet en cours — Phase A (Baseline) terminée, Phase B (Durcissement) à venir.*
