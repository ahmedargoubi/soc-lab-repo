# Installation & Configuration — Shuffle SOAR

Shuffle est le composant SOAR (Security Orchestration, Automation and Response) du lab — il reçoit les alertes Wazuh via webhook et exécute des workflows automatisés (voir `reports/phase-b-hardening/` pour les playbooks construits dessus : blocage réseau, isolation d'hôte, désactivation de compte AD).

---

## 1. Introduction

### 1.1 – Qu'est-ce que Shuffle SOAR

Shuffle est une plateforme SOAR open-source, auto-hébergeable, qui orchestre des réponses automatisées à partir d'alertes de sécurité. Elle reçoit des événements (ici, via un webhook déclenché par Wazuh), applique une logique de workflow (branchements conditionnels, approbation humaine, appels API), et déclenche des actions concrètes (blocage d'IP, création de cas, notification).

### 1.2 – Pourquoi dans ce lab

Sans Shuffle, chaque réponse à incident du lab serait manuelle. Avec Shuffle, une alerte à haute sévérité peut suivre un chemin structuré — notification, approbation, action automatisée — reproduisant la logique d'un vrai SOC où l'automatisation réduit le temps de réponse (MTTR) sans éliminer le contrôle humain sur les actions sensibles.

---

## 2. Environnement

| Paramètre | Valeur |
|---|---|
| Serveur Shuffle | VM CentOS `node4` (`192.168.9.144`) |
| Méthode d'installation | Docker Compose (self-hosted) |
| Wazuh Manager | Security-Core (`192.168.9.133`) |

---

## 3. Installation

### 3.1 – Docker et Docker Compose

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
```

### 3.2 – Cloner le dépôt Shuffle

```bash
sudo git clone https://github.com/Shuffle/Shuffle /opt/shuffle
cd /opt/shuffle
```

### 3.3 – Fichier `.env`

Variables clés utilisées :
```env
FRONTEND_PORT=3001
FRONTEND_PORT_HTTPS=3002
BACKEND_PORT=5001
OPENSEARCH_PORT=9201
BACKEND_HOSTNAME=shuffle-backend
OUTER_HOSTNAME=192.168.9.144
DB_LOCATION=/opt/shuffle/database
SHUFFLE_OPENSEARCH_URL=http://shuffle-opensearch:9200
SHUFFLE_OPENSEARCH_USERNAME=admin
SHUFFLE_OPENSEARCH_PASSWORD=Shuffle123!
```

⚠️ `OUTER_HOSTNAME` doit correspondre à l'IP réellement joignable depuis Security-Core (`192.168.9.144`) — c'est cette valeur qui détermine l'URL de base utilisée par Shuffle pour générer ses propres liens (webhooks, notifications).

### 3.4 – Démarrage des conteneurs

```bash
docker compose up -d
```

![Logs d'installation et de démarrage Docker Compose](screenshots/shuffle/01-docker-compose-install-logs.png)
![Shuffle en attente de la base de données au premier démarrage](screenshots/shuffle/02-shuffle-waiting-for-database.png)

---

## 4. Configuration

### 4.1 – Accès à l'interface web et création du compte admin

Interface : **`http://192.168.9.144:3001`**

![Formulaire de création du compte admin](screenshots/shuffle/03-admin-account-creation-form.png)
![Tableau de bord Shuffle (vue Open Source)](screenshots/shuffle/04-shuffle-dashboard-opensource.png)
![Page Discover Workflows](screenshots/shuffle/05-discover-workflows-page.png)

### 4.2 – Intégration Wazuh (webhook)

**Côté Shuffle :** création d'un workflow avec un déclencheur **Webhook**, générant une URL unique (`webhook_25c99880-a47d-4252-a989-8ce1833e63c6`).

![Configuration du déclencheur webhook dans Shuffle](screenshots/shuffle/07-webhook-trigger-config.png)

**Côté Wazuh** (`/var/ossec/etc/ossec.conf`, sur le Manager Security-Core) :
```xml
<integration>
    <name>shuffle</name>
    <hook_url>http://192.168.9.144:3001/api/v1/hooks/webhook_25c99880-a47d-4252-a989-8ce1833e63c6</hook_url>
    <level>3</level>
    <alert_format>json</alert_format>
</integration>
```

![Bloc d'intégration Shuffle dans ossec.conf](screenshots/shuffle/06-ossec-conf-shuffle-integration.png)

⚠️ Ce bloc initial utilise `<level>3</level>` (tout événement de niveau ≥3) plutôt qu'un `<rule_id>` ciblé — utile pour valider que le webhook reçoit bien du trafic pendant les tests, mais **trop large pour un usage en production** (génère du bruit sur des événements mineurs). Voir `reports/phase-b-hardening/phase-b-shuffle-soar.md` pour l'évolution vers des intégrations séparées par type de playbook (réseau, malware, AD).

```bash
sudo systemctl restart wazuh-manager
```

### 4.3 – Workflow de test : `SOC_wazuh`

Workflow minimal utilisant le node **"Repeat back to me"** de Shuffle (fonctionne comme un echo — renvoie simplement ce qu'il reçoit), pour confirmer que la réception fonctionne avant de construire une logique plus complexe.

![Configuration du workflow de test repeat_back_to_me](screenshots/shuffle/08-test-workflow-repeat-back.png)

---

## 5. Test — vérification de la réception des alertes

Déclencher n'importe quel événement Wazuh de niveau ≥3 sur un agent surveillé, puis vérifier :

![Log d'exécution montrant une alerte Wazuh reçue avec succès](screenshots/shuffle/09-execution-log-alert-received.png)

✅ Confirmé — les alertes Wazuh atteignent bien Shuffle via le webhook, et le workflow de test les traite correctement.

---

## 6. Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Conflit de port (3001, 5001, 9201) | Un autre service occupe déjà le port sur `node4` | `sudo ss -tlnp \| grep <port>` — même type de diagnostic que pour le conflit privacyIDEA/MISP sur Security-Core |
| Erreur de permissions sur la base de données | `DB_LOCATION` (`/opt/shuffle/database`) pas accessible en écriture par l'utilisateur Docker | `sudo chown -R 1000:1000 /opt/shuffle/database` (ou l'UID utilisé par les conteneurs Shuffle) |
| Erreurs SSL/TLS au premier accès | Certificat auto-signé sur `FRONTEND_PORT_HTTPS` | Accéder via `http://192.168.9.144:3001` (port HTTP) plutôt que le port HTTPS tant que TLS n'est pas configuré explicitement |
| Webhook Wazuh → Shuffle ne reçoit rien | Webhook non démarré côté Shuffle (doit être explicitement activé/lancé dans l'éditeur de workflow avant de recevoir du trafic) | Vérifier que le workflow est bien "en ligne" (statut actif) dans l'interface, pas juste sauvegardé |

---

## 7. Prochaines étapes

- Construire des workflows avancés au-delà du test `SOC_wazuh` : blocage d'IP via l'API Wazuh Active Response (`firewall-drop`), isolation d'hôte, désactivation de compte AD compromis — voir `reports/phase-b-hardening/phase-b-shuffle-soar.md`.
- Intégrer la création automatique de cas dans TheHive (`config/thehive.md`, section 8) pour chaque alerte escaladée.
- Affiner les intégrations `ossec.conf` : passer de `<level>3</level>` (large) à des `<rule_id>` ciblés par playbook, pour réduire le bruit.

---

## 8. Statut actuel

✅ Installation Docker Compose, démarrage des conteneurs, création du compte admin, intégration webhook de base — confirmés fonctionnels (réception d'alertes testée avec succès).
⏳ Workflows de réponse automatisée avancés (blocage IP, isolation, désactivation de compte) — en cours, voir rapport Phase B dédié.
⏳ Intégration TheHive — prévue, pas encore réalisée.

> 🚧 Captures en attente — les emplacements ci-dessus sont prêts à recevoir les images une fois envoyées.
