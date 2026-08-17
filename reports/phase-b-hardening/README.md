# Phase B — Durcissement + Automatisation

🟢 **En cours.**

## Avancement

| Item | Statut | Détails |
|---|---|---|
| MFA — OPNsense (admin GUI) | ✅ Configuré | TOTP natif OPNsense (serveur `Local_OTP`) — voir [`config/opnsense-mfa.md`](config/opnsense-mfa.md) |
| MFA — SSH (Security-Core) | ✅ Testé et confirmé | `libpam-google-authenticator` — voir [`config/security-core-ssh-mfa.md`](config/security-core-ssh-mfa.md) |
| MFA — Windows/AD logon | ⏳ Non fait | Nécessiterait un agent dédié (ex. privacyIDEA Credential Provider) |
| Intégration Wazuh ↔ MISP | ✅ Fonctionnelle | Corrélation de hashs via FIM — voir [`config/wazuh-misp-integration.md`](config/wazuh-misp-integration.md) |
| Intégration Wazuh ↔ VirusTotal | ✅ Fonctionnelle | Enrichissement automatique des hashs/IOCs — voir [`config/virustotal-integration.md`](config/virustotal-integration.md) |
| SOAR (Shuffle) — détection & blocage automatique brute-force | ✅ Testé de bout en bout | Wazuh → Shuffle → blocage actif de l'IP (active-response) → TheHive → email — voir [`config/phase-b-brute-force-blocking.md`](config/phase-b-brute-force-blocking.md) |
| TheHive — case management | ✅ Fonctionnel | Intégré au pipeline ci-dessus, alertes scopées à `rule_id 5763` pour éviter le bruit |
| SafeLine WAF (DMZ) | ✅ Déployé | Devant DVWA — voir [`config/safeline-waf.md`](config/safeline-waf.md) |
| Durcissement des règles de pare-feu OPNsense | 🟡 Proposé, non encore appliqué | Moindre privilège strict appliqué à la baseline Phase A — voir [`../../network/opnsense-firewall-rules-post-hardening.md`](../../network/opnsense-firewall-rules-post-hardening.md) |
| Durcissement Active Directory (LAPS, désactivation LLMNR/NBT-NS, Kerberos) | 🟡 En cours | voir [`config/phase-b-ad-hardening.md`](config/phase-b-ad-hardening.md) |
| Durcissement postes clients Windows | 🟡 En cours | voir [`config/windows-client-hardening.md`](config/windows-client-hardening.md) |
| Règles Wazuh/Suricata personnalisées | ⏳ À venir | |
| Déploiement Sysmon | ⏳ À venir | |

⚠️ **Point GRC à noter :** le MFA n'a pas été centralisé sur un seul mécanisme comme prévu initialement (privacyIDEA partout) — contrainte de temps. Résultat : trois approches différentes selon le point d'accès (TOTP natif OPNsense, PAM Google Authenticator pour SSH, rien encore pour Windows/AD). Ce compromis est documenté en détail dans chaque guide `config/` correspondant, avec la recommandation de centraliser via privacyIDEA en itération future.

⚠️ **Point à noter sur le pare-feu :** le durcissement des règles OPNsense (moindre privilège strict, isolation de l'attaquant, suppression des règles "tout port/tout protocole") est entièrement documenté et prêt, mais **pas encore appliqué dans OPNsense lui-même** — voir [`opnsense-firewall-rules-post-hardening.md`](../../network/opnsense-firewall-rules-post-hardening.md) pour le détail rule-by-rule et les points restant à trancher avant application (incohérence DMZ/Legacy, alias `AD_Ports` à confirmer).

📄 [`opnsense-firewall-rules-post-hardening.md`](../../network/opnsense-firewall-rules-post-hardening.md) — comparatif des règles de pare-feu avant/après durcissement, avec isolation stricte de l'attaquant

Le contenu détaillé (rapport complet Phase B façon Mandiant/GRC) sera rédigé une fois l'ensemble des items ci-dessus terminés — voir [`config/phase-b-brute-force-blocking.md`](config/phase-b-brute-force-blocking.md) pour un exemple déjà complet de ce niveau de détail, appliqué au pipeline SOAR.
