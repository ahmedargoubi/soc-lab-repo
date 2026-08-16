# Enterprise SOC / Red Team / DFIR / CTI / GRC Lab

> Simulation d'un réseau d'entreprise segmenté, attaqué et défendu à travers 5 rôles de cybersécurité successifs (Red Team, SOC Analyst, DFIR, CTI, GRC) sur la même infrastructure virtualisée.

Inspiré par des dépôts portfolio comme [Lab4PurpleSec](https://github.com/0xMR007/Lab4PurpleSec).

---

## 📌 Statut du projet

| Phase | Description | Statut |
|---|---|---|
| **Phase A — Baseline** | Attaques contre l'infrastructure telle que construite initialement, sans durcissement | ✅ Terminée (4/4 simulations + synthèse) |
| **Phase B — Durcissement + Automatisation** | MFA, SOAR (Shuffle), case management (TheHive), WAF, intégrations threat intel, durcissement AD, règles Wazuh/Suricata personnalisées | 🟡 En cours |
| **Phase C — Ré-attaque** | Rejeu des mêmes simulations contre le réseau durci, comparatif avant/après | ⏳ À venir |

---

## 🎯 Objectif du projet

Ce lab répond à une question simple : **qu'est-ce que change réellement,
concrètement, quand on durcit un réseau et qu'on automatise la réponse à
incident ?** Plutôt que de documenter des outils isolément, le projet
rejoue les **mêmes 4 attaques** contre la **même infrastructure**, à trois
moments différents de sa maturité :

1. **Phase A** mesure une baseline honnête, sans aucune protection
   supplémentaire — segmentation réseau brute + détection Wazuh par
   défaut.
2. **Phase B** applique un durcissement progressif et documenté (MFA,
   SOAR, case management, threat intel, WAF, durcissement AD) — chaque
   mesure est testée individuellement, pas seulement supposée efficace.
3. **Phase C** rejoue les 4 attaques de la Phase A contre le réseau
   durci, et compare directement : temps de détection, efficacité de
   segmentation, réussite/échec de chaque étape de la chaîne d'attaque,
   maturité GRC (NIST CSF 2.0).

Chaque décision de conception — y compris les gaps volontaires — est
documentée explicitement dans
[`docs/security-principles.md`](docs/security-principles.md).

---

## 🏗️ Architecture

- **Hyperviseur** : VMware Workstation Pro (laptop unique, Windows)
- **Pare-feu central** : OPNsense — 6 interfaces réseau, une par zone, filtrage first-match/deny-by-default
- **Zones réseau** :

| Zone | Rôle | Machines hébergées |
|---|---|---|
| **Security_LAN** | Cœur SOC : détection, DFIR, threat intel, SOAR, case management | Security-Core (Wazuh + MISP + Velociraptor + TheHive + VirusTotal), Shuffle, WAF, Management Workstation |
| **DMZ** | Services exposés | DMZ-Web (Apache + DVWA), derrière un WAF |
| **User_LAN** | Postes utilisateurs | 2 clients Windows 10 (AHMED, HAROUN), 1 client CentOS (node1) |
| **Server_LAN** | Serveurs internes | AD-DC (Windows Server 2022 — domaine `PROJET.local`, hostname `TEKUP-DC`) |
| **Legacy** | Machine volontairement vulnérable, non supervisée | Metasploitable2 |
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
| **VirusTotal** | Enrichissement threat intel | Intégré à Wazuh pour la vérification de hashs/IOCs |
| **Shuffle** | SOAR — automatisation de la réponse | Self-hosted, VM dédiée (`node4`, `192.168.9.144:3001`) |
| **TheHive** | Case management | Sur Security-Core, `192.168.9.133` |
| **SafeLine WAF** | Protection applicative devant la DMZ | Devant DVWA |
| **Suricata** | NIDS | Interfaces WAN, DMZ et Security_LAN d'OPNsense |
| **Kali Toolset** | Red Team | Metasploit, nmap, sqlmap, Responder, impacket, SharpHound/BloodHound, John the Ripper, Hydra |

📄 Guides d'installation/configuration détaillés : [`config/`](config/)

---

## 🎭 Les 5 rôles simulés

```
Red Team  →  SOC Analyst (Wazuh)  →  DFIR (Velociraptor)  →  CTI (MISP + MITRE ATT&CK)  →  GRC (NIST CSF 2.0)
```

Chaque simulation d'attaque est rejouée à travers ces 5 angles d'analyse, produisant un rapport unique qui couvre l'attaque, la détection, l'investigation, le renseignement sur la menace et l'évaluation de conformité.

| Rôle | Ce qu'il fait concrètement dans ce lab |
|---|---|
| **Red Team** | Exécute l'attaque (Kali) — injection, exploitation, mouvement latéral, ransomware réel selon le scénario |
| **SOC Analyst** | Analyse les alertes Wazuh générées, mesure le temps de détection (TTD), et — depuis la Phase B — supervise le pipeline SOAR (Shuffle) qui répond automatiquement aux alertes qualifiantes |
| **DFIR** | Utilise Velociraptor pour la collecte forensique post-incident (artefacts, timeline, mémoire) |
| **CTI** | Extrait les IOCs de l'attaque, les corrèle via MISP et VirusTotal, mappe la chaîne d'attaque sur MITRE ATT&CK |
| **GRC** | Évalue la maturité de la réponse selon NIST CSF 2.0, documente les gaps de conception et les compromis assumés |

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

## 🔧 Phase B — Durcissement + Automatisation

| Item | Statut | Détails |
|---|---|---|
| MFA — OPNsense (admin GUI) | ✅ Configuré | TOTP natif OPNsense — [`config/opnsense-mfa.md`](reports/phase-b-hardening/config/opnsense-mfa.md) |
| MFA — SSH (Security-Core) | ✅ Testé et confirmé | `libpam-google-authenticator` — [`config/security-core-ssh-mfa.md`](reports/phase-b-hardening/config/security-core-ssh-mfa.md) |
| Intégration Wazuh ↔ MISP | ✅ Fonctionnelle | Corrélation de hashs via FIM — [`config/wazuh-misp-integration.md`](reports/phase-b-hardening/config/wazuh-misp-integration.md) |
| Intégration Wazuh ↔ VirusTotal | ✅ Fonctionnelle | [`config/virustotal-integration.md`](reports/phase-b-hardening/config/virustotal-integration.md) |
| SOAR (Shuffle) — détection & blocage automatique brute-force | ✅ Testé de bout en bout | Wazuh → Shuffle → blocage actif de l'IP → TheHive → email — [`phase-b-brute-force-blocking.md`](reports/phase-b-hardening/config/phase-b-brute-force-blocking.md) |
| TheHive — case management | ✅ Fonctionnel | Intégré au pipeline ci-dessus |
| SafeLine WAF (DMZ) | ✅ Déployé | [`config/safeline-waf.md`](reports/phase-b-hardening/config/safeline-waf.md) |
| Durcissement Active Directory (LAPS, LLMNR/NBT-NS, Kerberos) | 🟡 En cours | [`config/phase-b-ad-hardening.md`](reports/phase-b-hardening/config/phase-b-ad-hardening.md) |
| Durcissement postes clients Windows | 🟡 En cours | [`config/windows-client-hardening.md`](reports/phase-b-hardening/config/windows-client-hardening.md) |
| MFA — Windows/AD logon | ⏳ Non fait | Nécessiterait un agent dédié (ex. privacyIDEA Credential Provider) |
| Règles Wazuh/Suricata personnalisées | ⏳ À venir | |
| Déploiement Sysmon | ⏳ À venir | |

⚠️ **Point GRC à noter :** le MFA n'a pas été centralisé sur un seul mécanisme comme prévu initialement (privacyIDEA partout) — contrainte de temps. Résultat : trois approches différentes selon le point d'accès (TOTP natif OPNsense, PAM Google Authenticator pour SSH, rien encore pour Windows/AD). Ce compromis est documenté en détail dans chaque guide `config/` correspondant, avec la recommandation de centraliser via privacyIDEA en itération future.

📄 Suivi détaillé : [`reports/phase-b-hardening/README.md`](reports/phase-b-hardening/README.md) · [`opnsense-firewall-rules-post-hardening.md`](network/opnsense-firewall-rules-post-hardening.md) (comparatif des règles de pare-feu avant/après durcissement — à venir)

---

## 🧭 Principes de conception

Les choix de segmentation, les gaps volontaires (ex. Legacy_LAN sans
agent), l'équilibre entre automatisation et contrôle humain, et la
philosophie de transparence sur les incohérences rencontrées en cours de
projet sont documentés dans
[`docs/security-principles.md`](docs/security-principles.md).

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
│   ├── shuffle.md
│   ├── thehive.md
│   ├── velociraptor.md
│   ├── misp.md
│   └── active-directory.md
├── network/
│   ├── opnsense-firewall-rules.md
│   ├── opnsense-firewall-rules-post-hardening.md
│   └── diagrams/
│       ├── phase-a-topology.png
│       └── phase-b-topology.jpg
├── reports/
│   ├── phase-a-baseline/
│   │   ├── simulation-1-dvwa.md
│   │   ├── simulation-2-metasploitable-pivot.md
│   │   ├── simulation-3-wannacry-ransomware.md
│   │   ├── simulation-4-active-directory.md
│   │   └── phase-a-summary.md
│   ├── phase-b-hardening/
│   │   ├── README.md
│   │   └── config/
│   │       ├── phase-b-brute-force-blocking.md
│   │       ├── phase-b-ad-hardening.md
│   │       ├── opnsense-mfa.md
│   │       ├── security-core-ssh-mfa.md
│   │       ├── safeline-waf.md
│   │       ├── virustotal-integration.md
│   │       ├── wazuh-misp-integration.md
│   │       └── windows-client-hardening.md
│   └── phase-c-reattack/       (à venir)
├── scripts/
│   ├── wazuh-install.sh
│   ├── custom-w2thive.py
│   └── custom-w2thive
└── DFIR/
    └── velociraptor-dfir.md
```

---


