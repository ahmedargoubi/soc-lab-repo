# Threat Model — DVWA (SQL Injection / Command Injection)

**Zone :** DMZ_LAN · **Cible :** DVWA (`192.168.11.177`) · **STRIDE :** Tampering, Elevation of Privilege · **MITRE ATT&CK :** T1190 (Exploit Public-Facing Application), T1059 (Command and Scripting Interpreter)

```mermaid
flowchart TD
    A(["Attaquant — Attaque application web\n(injection SQL / commande)"]) --> B{"Signature connue\ndétectée par le WAF ?"}
    B -- "Oui" --> C["WAF bloque la requête\n+ log 'Access Forbidden'"]
    C --> D["Fin — requête rejetée"]

    B -- "Non" --> E{"Wazuh détecte un pattern\nanormal dans les logs web\n(SQLi / CmdInj) ?"}
    E -- "Oui" --> F["Alerte Wazuh générée\n→ Notifier l'analyste SOC"]

    E -- "Non" --> G{"Un processus enfant inattendu\nest observé sous www-data\n(ex. shell spawné) ?"}
    G -- "Oui" --> H["Alerte Wazuh (anomalie process)\n→ Notifier l'analyste SOC"]

    G -- "Non" --> I["Non détecté\n(risque résiduel)"]
```

## Seuils / logique réelle

- **Détection primaire (couche 1) :** le WAF SafeLine, déployé en Phase B, filtre les signatures connues avant même que la requête n'atteigne DVWA.
- **Détection secondaire (couche 2) :** en l'absence de blocage WAF (payload non signé), Wazuh reste la ligne de défense suivante — surveillance des logs applicatifs.
- **Détection tertiaire (couche 3) :** même sans signature de log reconnue, un process enfant inattendu sous le compte de service web (`www-data`) est un signal fort d'exécution de commande — c'est ce type de comportement qui a permis d'obtenir un shell en Phase A.

## Résultat mesuré (Phase A)

Shell `www-data` obtenu, tentative d'escalade root **échouée**, tentative de pivot vers d'autres zones **bloquée par la segmentation**. Détection Wazuh confirmée.

## Statut Phase B

🟡 **Réduit, pas éliminé.** Le WAF ajoute une première couche, mais reste contournable par un payload non signé — les couches 2 et 3 restent donc la vraie ligne de défense, inchangées depuis la Phase A.
