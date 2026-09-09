# Threat Model — WannaCry (Ransomware)

**Zone :** User_LAN · **STRIDE :** Tampering · **MITRE ATT&CK :** T1204 (User Execution), T1210 (Exploitation of Remote Services — SMB/EternalBlue), T1486 (Data Encrypted for Impact)

```mermaid
flowchart TD
    A(["Exécution du ransomware\n(échantillon réel WannaCry)"]) --> B{"Domaine killswitch\naccessible ?"}
    B -- "Oui" --> C["Ransomware s'arrête\nde lui-même"]
    B -- "Non (lab isolé)" --> D["Chiffrement démarre"]

    D --> E{"Wazuh FIM : x N fichiers\nmodifiés en 1 minute\nsur un même hôte ?"}
    E -- "Non (sous le seuil)" --> F["Surveillance continue,\naucune alerte"]

    E -- "Oui" --> G{"Taux de modification\ndépasse le seuil critique\n(x2 le seuil initial) ?"}
    G -- "Non" --> H["Alerte Wazuh (niveau moyen)\n→ Notifier l'analyste SOC"]
    G -- "Oui" --> I["Alerte Wazuh (niveau critique)\n→ Isoler l'hôte du réseau"]

    D --> J{"Tentative de propagation\nSMB (port 445) vers\nd'autres hôtes ?"}
    J -- "Oui" --> K{"AD/postes durcis\n(signature SMB active) ?"}
    K -- "Non (Phase B incomplète)" --> L["Propagation possible\nvers d'autres hôtes User_LAN"]
    K -- "Oui" --> M["Propagation bloquée"]
    J -- "Non" --> N["Confiné à l'hôte initial"]
```

## Seuils / logique réelle

- Le seuil exact "x N fichiers/minute" n'est pas documenté précisément
  dans ce lab — Wazuh FIM (File Integrity Monitoring) a détecté le
  changement de masse **rapidement** en Phase A, mais sans qu'un seuil
  numérique formel à deux niveaux (alerte / isolation automatique) ait
  été configuré. C'est une amélioration concrète identifiable pour la
  suite : définir explicitement ces deux seuils, sur le modèle de ce qui
  a été fait pour le brute-force SSH (voir le 5ᵉ modèle de menace).
- **La branche d'isolation automatique (`I`) n'est pas encore
  implémentée** — contrairement au blocage IP automatique du pipeline
  SSH, il n'existe pas aujourd'hui d'action Shuffle qui isole
  automatiquement un hôte sur détection FIM critique.

## Résultat mesuré (Phase A)

Chiffrement **partiel** (contenu avant complétion), détection Wazuh
confirmée rapide via FIM.

## Statut Phase B

🔴 **Vecteur de propagation non fermé.** Le durcissement AD/postes
clients (signature SMB, entre autres) est encore 🟡 en cours, non
confirmé terminé — la branche `K → Non` reste donc la situation réelle
actuelle. L'isolation automatique sur détection critique (`G → Oui → I`)
n'existe pas encore comme automatisation — reste manuelle.
