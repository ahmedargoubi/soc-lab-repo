# Threat Model — Active Directory (LLMNR Poisoning → Pass-the-Hash → BloodHound)

**Zone :** User_LAN → Server_LAN · **STRIDE :** Spoofing, Elevation of Privilege, Information Disclosure · **MITRE ATT&CK :** T1557.001 (LLMNR/NBT-NS Poisoning), T1110.002 (Password Cracking), T1550.002 (Pass the Hash), T1482/T1087 (Domain Trust/Account Discovery)

```mermaid
flowchart TD
    A(["Attaquant — LLMNR/NBT-NS\nPoisoning sur User_LAN"]) --> B{"LLMNR/NBT-NS désactivé\nsur les postes clients ?"}
    B -- "Oui (cible)" --> C["Empoisonnement impossible\n→ chaîne arrêtée ici"]
    B -- "Non (état actuel)" --> D["Hash NTLMv2 capturé"]

    D --> E{"Hash cassé hors-ligne\n(John the Ripper) ?"}
    E -- "Non" --> F["Attaquant bloqué\n(hash inutilisable)"]
    E -- "Oui" --> G["Identifiants valides obtenus"]

    G --> H{"Wazuh Rule 92652\n(Pass-the-Hash) déclenchée ?"}
    H -- "Non" --> I["Authentification silencieuse\nvers l'AD-DC"]
    H -- "Oui" --> J["Alerte critique quasi instantanée\n→ Notifier l'analyste SOC"]

    J --> K{"Cette règle est-elle intégrée\nau pipeline de réponse\nautomatisée (Shuffle) ?"}
    K -- "Non (état actuel)" --> L["Réponse manuelle uniquement\n→ fenêtre d'exposition avant\nintervention humaine"]
    K -- "Oui (amélioration future)" --> M["Blocage/containment\nautomatique possible"]

    G --> N["Reconnaissance BloodHound\n(cartographie des relations AD)"]
    N --> O{"Contrôle spécifique sur\nl'usage de BloodHound ?"}
    O -- "Non (état actuel)" --> P["Reconnaissance complète,\nnon détectée"]
```

## Seuils / logique réelle

- **Détection confirmée rapide et fiable** (`H → Oui`) : c'est le point
  fort mesuré en Phase A — Wazuh Rule `92652` a détecté le Pass-the-Hash
  quasi instantanément.
- **Mais la détection reste manuelle** (`K → Non`) : contrairement au
  brute-force SSH (5ᵉ modèle de menace), ce type d'alerte n'est pas
  aujourd'hui branché sur Shuffle pour une réponse automatique — c'est
  une extension naturelle et concrète du pipeline SOAR déjà construit.
- **BloodHound reste un angle mort total** (`O → Non`) : aucun contrôle
  identifié à ce jour pour cette étape spécifique de reconnaissance.

## Résultat mesuré (Phase A)

Chaîne complète réussie jusqu'à la compromission **Domain Admin**.
Détection Wazuh confirmée rapide sur l'étape Pass-the-Hash — mais sans
containment automatique, la chaîne d'attaque a le temps d'aboutir avant
toute réponse humaine.

## Statut Phase B

🔴 **La branche `B → Non` reste la situation réelle** : le durcissement
AD (désactivation LLMNR/NBT-NS) est 🟡 en cours, non confirmé terminé —
le point d'entrée de toute la chaîne reste donc ouvert aujourd'hui.
