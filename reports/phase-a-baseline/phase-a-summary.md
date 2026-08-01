# Phase A — Baseline: Comparative Summary

Synthèse comparative des 4 simulations d'attaque menées contre l'infrastructure telle que construite initialement, sans durcissement (privacyIDEA/MFA, SOAR, règles personnalisées) ni automatisation. Cette baseline servira de référence pour mesurer l'amélioration en Phase C, après le durcissement de Phase B.

---

## 1. Vue d'ensemble

| # | Simulation | Zone cible | Vecteur initial | Résultat final |
|---|---|---|---|---|
| 1 | DVWA – SQLi & Command Injection | DMZ | Injection SQL + injection de commande | Reverse shell `www-data`, privesc root échouée |
| 2 | Metasploitable2 – Backdoor & pivot | Legacy | Backdoor root non-authentifié (port 1524) | Accès root immédiat, pivot bloqué |
| 3 | WannaCry – Ransomware réel | User_LAN | Exécution manuelle du ransomware | Chiffrement partiel, killswitch bloqué |
| 4 | Active Directory – LLMNR/PtH/BloodHound | Server_LAN | Empoisonnement LLMNR → cassage hash → PtH | Compromission complète du domaine (Domain Admin) |

---

## 2. Temps de détection (TTD)

| Simulation | TTD | Mécanisme de détection |
|---|---|---|
| 1 — DVWA | ~0 s | Wazuh (logs Apache, Rule 31106/31164) |
| 2 — Metasploitable2 | **Non détecté** | Aucun agent Wazuh sur Legacy (gap volontaire) |
| 3 — WannaCry | ~9 s | Wazuh FIM (syscheck, Rule 554/553/550) |
| 4 — Active Directory | ~0 s | Wazuh (Windows Event Logs, Rule 92652 — Pass-the-Hash) |

**Constat :** la détection host-based (Wazuh) est quasi instantanée partout où un agent est déployé. Le seul échec de détection (Sim 2) est un gap de conception assumé, pas un échec technique de l'outil.

---

## 3. Segmentation réseau

| Simulation | Pivot tenté | Résultat |
|---|---|---|
| 1 — DVWA | DMZ → Security_LAN / Server_LAN / Legacy / Internet | ❌ Bloqué (0% de succès) |
| 2 — Metasploitable2 | Legacy → User_LAN / Server_LAN / Security_LAN | ❌ Bloqué (0% de succès, 100% packet loss) |
| 3 — WannaCry | User_LAN → Security_LAN / Server_LAN / Legacy | ❌ Bloqué (0% de succès) |
| 4 — Active Directory | User_LAN → Server_LAN (445/SMB) | ✅ **Autorisé** — trafic légitime d'authentification AD, ce qui a permis l'attaque |

**Constat :** la segmentation OPNsense a tenu dans 3 cas sur 4. Le seul cas où elle n'a pas empêché l'attaque (Sim 4) est un flux **légitimement nécessaire** (SMB/NTLM pour l'authentification AD) — la segmentation réseau seule ne peut pas couvrir ce type de trafic ; une détection comportementale est nécessaire.

---

## 4. Gaps de détection récurrents

| Gap identifié | Simulation(s) concernée(s) | Cause |
|---|---|---|
| Absence d'agent Wazuh sur Legacy | Sim 2 | Design volontaire (test de segmentation "à l'aveugle") |
| Absence de règles Suricata pour NTLM/SMB anomalies | Sim 4 | Signatures par défaut insuffisantes |
| Absence de règle process-based pour ransomware (`tasksche.exe`) | Sim 3 | Seule la détection FIM (fichiers) était en place, pas la détection process |
| Absence de règle dédiée à l'exfiltration via `UNION SELECT` | Sim 1 | Rule 31106 détecte le code 200 HTTP, pas l'exfiltration en tant que telle |

**Constat :** la détection réseau (Suricata) est systématiquement le point faible sur les 4 simulations — elle n'a détecté aucune des attaques de manière autonome. C'est l'axe d'amélioration prioritaire pour la Phase B.

---

## 5. Maturité GRC (NIST CSF 2.0) — vue consolidée

| Fonction | Constat récurrent sur les 4 simulations |
|---|---|
| **Govern** | Absence systématique de politique de durcissement formalisée (GPO, WAF, patch management) |
| **Identify** | Vulnérabilités connues et volontaires (DVWA, Metasploitable2, absence de patch AD) — cohérent avec un environnement de test |
| **Protect** | Gaps répétés sur le moindre privilège (comptes Domain Admin surexposés dans Sim 3 et 4) et le durcissement logiciel |
| **Detect** | Fort sur le host-based (Wazuh), faible sur le network-based (Suricata) |
| **Respond** | Correct dans les 4 cas — les alertes générées ont été triées et documentées |
| **Recover** | Non applicable en Phase A (baseline) — à évaluer en Phase C |

---

## 6. Priorités transverses pour la Phase B

1. **Suricata** : ajouter des règles personnalisées pour NTLM/SMB anomalies, connexions à des ports de backdoor connus (1524, 6200...), et signatures ransomware.
2. **Moindre privilège** : retirer les comptes utilisateurs (`ahmed`, `haroun`) des groupes Domain Admins ; utiliser des comptes d'administration dédiés.
3. **Durcissement AD** : désactiver LLMNR/NBT-NS, forcer Kerberos, déployer LAPS, politique de mots de passe renforcée.
4. **Couverture Wazuh** : décider si Legacy reste volontairement sans agent (pour continuer à tester la segmentation à l'aveugle) ou si un agent doit être ajouté.
5. **Automatisation (Shuffle/SOAR)** : blocage automatique d'IP, création d'événements MISP, enrichissement des alertes.

---

*Ce document sera complété en Phase C avec un comparatif direct avant/après durcissement sur les mêmes métriques (TTD, segmentation, maturité NIST CSF 2.0).*
