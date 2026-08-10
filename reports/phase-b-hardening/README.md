# Phase B — Durcissement + Automatisation

🟢 **En cours.**

## Avancement

| Item | Statut | Détails |
|---|---|---|
| MFA — OPNsense (admin GUI) | ✅ Configuré | TOTP natif OPNsense (serveur `Local_OTP`) — voir [`config/opnsense-mfa.md`](../../config/opnsense-mfa.md) |
| MFA — SSH (Security-Core) | ✅ Testé et confirmé | `libpam-google-authenticator` — voir [`config/security-core-ssh-mfa.md`](../../config/security-core-ssh-mfa.md) |
| MFA — Windows/AD logon | ⏳ Non fait | Nécessiterait un agent dédié (ex. privacyIDEA Credential Provider) |
| Intégration Wazuh ↔ MISP | ✅ Fonctionnelle | Corrélation de hashs via FIM — voir [`config/wazuh-misp-integration.md`](../../config/wazuh-misp-integration.md) |
| SOAR (Shuffle) | ⏳ À venir | |
| Durcissement Active Directory (LAPS, désactivation LLMNR/NBT-NS, Kerberos) | ⏳ À venir | |
| Règles Wazuh/Suricata personnalisées | ⏳ À venir | |
| Déploiement Sysmon | ⏳ À venir | |
| SafeLine WAF (DMZ) | ⏳ À venir | |
| Wazuh ↔ VirusTotal | ⏳ À venir | |

⚠️ **Point GRC à noter :** le MFA n'a pas été centralisé sur un seul mécanisme comme prévu initialement (privacyIDEA partout) — contrainte de temps. Résultat : trois approches différentes selon le point d'accès (TOTP natif OPNsense, PAM Google Authenticator pour SSH, rien encore pour Windows/AD). Ce compromis est documenté en détail dans chaque guide `config/` correspondant, avec la recommandation de centraliser via privacyIDEA en itération future.

📄 [`opnsense-firewall-rules-post-hardening.md`](opnsense-firewall-rules-post-hardening.md) — comparatif des règles de pare-feu avant/après durcissement (à venir, une fois le durcissement réseau lancé)

Le contenu détaillé (rapport complet Phase B façon Mandiant/GRC) sera rédigé une fois l'ensemble des items ci-dessus terminés.
