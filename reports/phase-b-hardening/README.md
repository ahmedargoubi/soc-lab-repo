# Phase B — Durcissement + Automatisation

🟢 **En cours.**

## Avancement

| Item | Statut | Détails |
|---|---|---|
| MFA — OPNsense (admin GUI) | ✅ Configuré | TOTP natif OPNsense (serveur `Local_OTP`) — voir [`config/opnsense-mfa.md`](config/opnsense-mfa.md) |
| MFA — SSH (Security-Core) | ✅ Configuré | `libpam-google-authenticator` — voir [`config/security-core-ssh-mfa.md`](config/security-core-ssh-mfa.md) |
| Intégration Wazuh ↔ MISP |  ✅ Configuré  | Corrélation de hashs via FIM — voir [`config/wazuh-misp-integration.md`](config/wazuh-misp-integration.md) |
| Intégration Wazuh ↔ VirusTotal | ✅ Configuré  | Enrichissement automatique des hashs/IOCs — voir [`config/virustotal-integration.md`](config/virustotal-integration.md) |
| SOAR (Shuffle) — détection & blocage automatique brute-force | ✅ Configuré  | Wazuh → Shuffle → blocage actif de l'IP (active-response) → TheHive → email — voir [`config/phase-b-brute-force-blocking.md`](config/phase-b-brute-force-blocking.md) |
| TheHive — case management | ✅ Configuré  | Intégré au pipeline ci-dessus, alertes scopées à `rule_id 5763` pour éviter le bruit |
| SafeLine WAF (DMZ) | ✅ Configuré  | Devant DVWA — voir [`config/safeline-waf.md`](config/safeline-waf.md) |
| Durcissement des règles de pare-feu OPNsense | ✅ Configuré  | Moindre privilège strict appliqué à la baseline Phase A — voir [`../../network/opnsense-firewall-rules-post-hardening.md`](../../network/opnsense-firewall-rules-post-hardening.md) |
| Durcissement Active Directory (LAPS, désactivation LLMNR/NBT-NS, Kerberos) | ✅ Configuré  | voir [`config/phase-b-ad-hardening.md`](config/phase-b-ad-hardening.md) |
| Durcissement postes clients Windows | ✅ Configuré  | voir [`config/windows-client-hardening.md`](config/windows-client-hardening.md) |


⚠️ **Point GRC à noter :** le MFA n'a pas été centralisé sur un seul mécanisme comme prévu initialement (privacyIDEA partout) — contrainte de temps. Résultat : trois approches différentes selon le point d'accès (TOTP natif OPNsense, PAM Google Authenticator pour SSH, rien encore pour Windows/AD). Ce compromis est documenté en détail dans chaque guide `config/` correspondant, avec la recommandation de centraliser via privacyIDEA en itération future.


📄 [`opnsense-firewall-rules-post-hardening.md`](../../network/opnsense-firewall-rules-post-hardening.md) — comparatif des règles de pare-feu avant/après durcissement, avec isolation stricte de l'attaquant

Le contenu détaillé (rapport complet Phase B façon Mandiant/GRC) sera rédigé une fois l'ensemble des items ci-dessus terminés — voir [`config/phase-b-brute-force-blocking.md`](config/phase-b-brute-force-blocking.md) pour un exemple déjà complet de ce niveau de détail, appliqué au pipeline SOAR.
