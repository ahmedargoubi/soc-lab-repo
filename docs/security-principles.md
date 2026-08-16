# Security Principles

Principes de conception sécuritaire appliqués à ce lab — ce qui relève de
choix **intentionnels** (pour créer un scénario pédagogique réaliste) par
opposition aux **gaps réels** identifiés en cours de route et corrigés (ou
prévus à corriger) en Phase B.

---

## 1. Segmentation réseau par zone de confiance

Le réseau est découpé en 6 zones (`Attacker`, `DMZ_LAN`, `Legacy_LAN`,
`Security_LAN`, `Server_LAN`, `User_LAN`), chacune raccordée à une
interface dédiée d'OPNsense, avec un filtrage **first-match / deny-by-default**.

**Pourquoi 6 zones et pas moins :** chaque zone représente un niveau de
confiance et un rôle métier distinct — un service exposé publiquement
(DMZ) n'a pas les mêmes besoins d'accès qu'un poste utilisateur, qui n'a
pas les mêmes besoins qu'un serveur d'infrastructure critique (AD) ou que
la zone qui héberge les outils de sécurité eux-mêmes (Security_LAN).
Séparer ces zones permet de mesurer, simulation après simulation, si un
attaquant qui compromet une zone peut effectivement pivoter vers les
autres — c'est directement ce qui a été mesuré en Phase A (voir
[`phase-a-summary.md`](../reports/phase-a-baseline/phase-a-summary.md),
section 3).

**Constat Phase A :** la segmentation a bloqué le pivot dans 3 cas sur 4.
Le seul cas où elle a laissé passer une attaque (Simulation 4 — Active
Directory) concernait un flux **légitimement nécessaire** (SMB/NTLM pour
l'authentification AD) — la conclusion retenue est que la segmentation
réseau seule ne peut pas couvrir ce type de trafic légitime détourné ; une
détection comportementale est nécessaire en complément (voir Section 3
ci-dessous).

---

## 2. Gap de conception assumé — Legacy_LAN sans agent

`Legacy_LAN` (Metasploitable2) est **volontairement** dépourvu d'agent
Wazuh et Velociraptor, et ne dispose d'aucune règle de pare-feu manuelle
sortante. Ce n'est pas un oubli : l'objectif est de mesurer ce qui se
passe quand une machine vulnérable **n'est pas du tout instrumentée** —
un scénario réaliste dans beaucoup d'environnements réels ("shadow IT",
legacy systems non intégrés au SOC).

**Résultat mesuré (Simulation 2) :** compromission root immédiate, **non
détectée** (gap de détection confirmé, pas un échec d'outil — Wazuh
n'était simplement pas déployé là), mais pivot réseau vers les autres
zones bloqué par la segmentation.

Ce principe illustre une idée volontairement centrale au lab : la
segmentation réseau et la détection host-based sont deux couches
**indépendantes** de défense en profondeur — l'une peut tenir quand
l'autre échoue complètement.

---

## 3. Défense en profondeur — plusieurs couches indépendantes

Aucune couche unique n'est traitée comme suffisante :

| Couche | Outil(s) | Ce qu'elle couvre |
|---|---|---|
| Filtrage réseau | OPNsense | Bloque le pivot inter-zones |
| Filtrage applicatif | WAF (SafeLine, devant DVWA) | Bloque les attaques HTTP connues avant qu'elles n'atteignent l'application |
| Détection host-based | Wazuh (agents) | Détecte l'activité malveillante sur l'hôte, y compris quand le trafic réseau est légitime |
| Threat Intelligence | MISP, VirusTotal | Corrèle les IOCs observés avec des indicateurs connus |
| Réponse automatisée | Shuffle (SOAR) | Réduit le temps entre détection et confinement (MTTR) sans attendre une intervention manuelle |
| Suivi / gouvernance | TheHive | Garantit qu'une action automatisée reste documentée, vérifiée et formellement close par un humain |

La Simulation 4 est l'exemple concret qui justifie cet empilement : la
segmentation seule n'aurait pas suffi (trafic AD légitime), mais la
détection Wazuh, elle, a identifié l'attaque quasi instantanément
(Pass-the-Hash, Rule 92652).

---

## 4. Automatisation rapide, jugement humain préservé

Le pipeline SOAR construit en Phase B (Wazuh → Shuffle → blocage actif →
TheHive → email) suit un principe précis : **l'automatisation gère la
vitesse de confinement, l'analyste garde le jugement et la
documentation.**

Concrètement :
- Le blocage de l'IP attaquante est **entièrement automatique** — aucune
  approbation humaine requise, pour minimiser le temps d'exposition.
- Mais chaque alerte qualifiante devient systématiquement un **case
  TheHive**, avec une checklist de tâches d'investigation
  (`iptables` vérifié, logs d'authentification passés en revue, intégrité
  du compte vérifiée, notification confirmée) — **rien n'est
  automatiquement classé "résolu"** sans qu'un analyste ait vérifié et
  documenté chaque hypothèse.

Ce principe est directement documenté et testé dans
[`phase-b-brute-force-blocking.md`](../reports/phase-b-hardening/config/phase-b-brute-force-blocking.md).

---

## 5. Moindre privilège appliqué progressivement

Le lab suit une trajectoire volontairement progressive plutôt que
"tout durci dès le départ" :

- Phase A mesure la baseline **sans aucun durcissement**, pour établir
  un point de comparaison honnête.
- Phase B applique les mesures une par une (MFA, règles de détection
  personnalisées, durcissement AD prévu — LAPS, désactivation
  LLMNR/NBT-NS, application de Kerberos), en documentant explicitement
  ce qui est fait vs. ce qui reste à faire.
- Phase C rejoue les mêmes attaques pour mesurer l'amélioration réelle,
  pas supposée.

**Compromis assumé et documenté :** le MFA n'a pas été centralisé sur un
seul mécanisme (privacyIDEA partout, comme prévu initialement) par
contrainte de temps — trois approches coexistent selon le point d'accès
(TOTP natif OPNsense pour l'admin GUI, PAM Google Authenticator pour SSH,
rien encore pour l'ouverture de session Windows/AD). Ce genre de
compromis est documenté explicitement plutôt que masqué, avec la
recommandation de centralisation en itération future — voir
[`reports/phase-b-hardening/README.md`](../reports/phase-b-hardening/README.md).

---

## 6. Transparence des incohérences plutôt que correction silencieuse

Plusieurs incohérences ont été rencontrées en cours de projet (bugs de
configuration VMware/OPNsense ayant changé des IPs plusieurs fois,
inversions de sous-réseau entre DMZ et Legacy, etc.). Le principe suivi
dans toute la documentation de ce lab est de **documenter ces
incohérences explicitement** — dans
[`troubleshooting-journal.md`](troubleshooting-journal.md) et directement
en note dans les rapports concernés — plutôt que de réécrire l'historique
rétroactivement pour que tout paraisse cohérent. Un rapport d'incident
réel n'a pas cette liberté ; ce lab ne se l'accorde pas non plus.

---

## 7. Résumé des principes

| Principe | Application dans ce lab |
|---|---|
| Segmentation par zone de confiance | 6 zones OPNsense, first-match/deny-by-default |
| Défense en profondeur | Réseau + applicatif (WAF) + host-based (Wazuh) + threat intel (MISP/VirusTotal) |
| Gaps de conception assumés et mesurés | Legacy_LAN sans agent, testé et documenté (Sim 2) |
| Automatisation rapide, contrôle humain préservé | Shuffle bloque automatiquement, TheHive impose une vérification humaine avant clôture |
| Durcissement progressif et honnête | Phase A (baseline) → B (durcissement) → C (mesure de l'amélioration) |
| Transparence sur les incohérences | Journal de dépannage dédié, notes explicites plutôt que réécriture |
