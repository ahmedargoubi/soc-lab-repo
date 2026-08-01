# Troubleshooting Journal — Réseau & Virtualisation

> Ce journal documente les bugs de configuration VMware/OPNsense rencontrés au cours de la construction du lab, et pourquoi certaines machines ont changé de sous-réseau plusieurs fois. Les incohérences d'IP visibles entre l'architecture globale et les rapports de simulation individuels ne sont **pas** des erreurs de documentation — elles reflètent l'historique réel du projet, expliqué ici.

---

## 🚧 Contenu à venir

Pour rédiger ce journal, j'ai besoin — pour **chaque incident réseau** que tu veux documenter — des informations suivantes :

| Champ | Détail |
|---|---|
| **Machine(s) concernée(s)** | Ex. DMZ-Web, AD-DC, Kali... |
| **Symptôme observé** | Ex. "la VM DMZ-Web répondait sur 192.168.6.x au lieu de 192.168.11.x", "route asymétrique, ping OK mais TCP échoue"... |
| **Cause identifiée** | Ex. mauvais adaptateur réseau lié dans VMware, mauvaise interface OPNsense assignée, DHCP mal configuré... |
| **Date approximative** | Pour situer l'incident dans la chronologie du projet |
| **Résolution appliquée** | Ce qui a été changé pour corriger (ou : toujours non résolu / contournement uniquement) |
| **Impact sur la documentation** | Quel(s) rapport(s) de simulation utilisent l'ancienne IP vs. la nouvelle |
| **Capture(s) d'écran** | Config réseau VMware (VMnet/adaptateur), interfaces OPNsense, tables de routage, `ip addr`/`ipconfig`, tests `ping`/`traceroute` pertinents |

**Ce que je sais déjà** (à partir de ton brief initial, à confirmer/compléter) :
- Le tableau d'architecture liste `DMZ` et `Legacy` avec le même sous-réseau `192.168.6.0/24` — probablement un résidu d'une reconfiguration
- Le rapport Simulation 1 utilise `192.168.11.177` pour DMZ-Web, différent du tableau d'architecture
- Cause générale mentionnée : "mauvais adaptateur réseau lié, routes asymétriques"

➡️ Envoie-moi les détails et captures pour chaque incident (autant de fois que nécessaire), et je remplirai ce journal entrée par entrée, dans l'ordre chronologique.

---

## Entrées

*(à venir)*
