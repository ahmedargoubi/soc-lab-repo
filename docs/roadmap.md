# Roadmap

Ce projet est construit en trois phases successives, chacune rejouant les mêmes scénarios d'attaque contre la même infrastructure segmentée, à des niveaux de maturité de sécurité croissants.

---

## Phase A — Baseline

**Statut : 🟢 Terminée (4/4 simulations)**

Attaques menées contre le réseau tel que construit initialement, **sans durcissement supplémentaire ni automatisation**. Objectif : établir une ligne de base réaliste des forces et faiblesses de détection/segmentation avant toute amélioration.

| # | Simulation | Statut |
|---|---|---|
| 1 | DVWA — SQL Injection & Command Injection (DMZ) | ✅ Terminée |
| 2 | Metasploitable2 — Backdoor & pivot test (Legacy) | ✅ Terminée |
| 3 | WannaCry — Ransomware réel (User_LAN) | ✅ Terminée |
| 4 | Active Directory — LLMNR/PtH/BloodHound (Server_LAN) | ✅ Terminée |

➡️ Synthèse : [`reports/phase-a-baseline/phase-a-summary.md`](../reports/phase-a-baseline/phase-a-summary.md)

---

## Phase B — Durcissement + Automatisation

**Statut : ⏳ Pas encore commencée**

Application des mesures de remédiation identifiées en Phase A, et ajout de capacités d'automatisation/détection avancées :

- **MFA** : déploiement de privacyIDEA
- **SOAR** : automatisation des réponses via Shuffle
- **Durcissement Active Directory** : LAPS, désactivation LLMNR/NBT-NS, application de Kerberos, moindre privilège
- **Règles de détection personnalisées** : nouvelles règles Wazuh (ex. NTLM depuis IP non fiable, process ransomware) et signatures Suricata (ex. anomalies SMB/NTLM)
- **Intégration MISP ↔ Wazuh** pour la corrélation automatique d'IOCs

➡️ Rapports (à venir) : [`reports/phase-b-hardening/`](../reports/phase-b-hardening/)

---

## Phase C — Ré-attaque

**Statut : ⏳ Pas encore commencée**

Rejeu des **mêmes simulations** (1 à 4) contre l'infrastructure durcie de la Phase B, avec un comparatif direct avant/après sur :

- Temps de détection (TTD)
- Efficacité de la segmentation
- Succès/échec de chaque étape de la chaîne d'attaque
- Maturité GRC (NIST CSF 2.0)

➡️ Rapports (à venir) : [`reports/phase-c-reattack/`](../reports/phase-c-reattack/)

---

*Dernière mise à jour : Phase A terminée, en attente du lancement de la Phase B.*
