# Installation & Configuration — TheHive (plateforme de gestion des cas)

TheHive sert de plateforme de gestion des incidents ("case management") pour le SOC du lab — les alertes Wazuh, relayées via Shuffle, y seront escaladées et suivies comme de vrais dossiers d'investigation.

---

## 1. Introduction

### 1.1 – Qu'est-ce que TheHive ?

TheHive est une plateforme open-source de réponse à incident (Security Incident Response Platform). Elle permet de transformer une alerte brute (venant d'un SIEM comme Wazuh) en un **cas** structuré : tâches à réaliser, observables (IOCs) à analyser, notes d'investigation, statut de résolution — le tout traçable et partageable entre analystes.


![Installation des dépendances et de Java 11](thehive+shuffle/b4d8bee8-ae9a-480a-bd44-c153abb5181d.png)

### 1.2 – Pourquoi dans ce lab

Jusqu'ici, le lab dispose de détection (Wazuh), de threat intelligence (MISP), d'automatisation (Shuffle) — mais d'aucun endroit centralisé pour **suivre le cycle de vie d'un incident** (de la détection à la clôture). TheHive comble ce manque : c'est la pièce "GRC/gestion opérationnelle" qui manquait pour boucler la boucle SOC complète.

---

## 2. Environnement

| Paramètre | Valeur |
|---|---|
| Serveur | Security-Core (`192.168.9.133`) |
| OS | Ubuntu 26.04 LTS |
| Version TheHive | 5.2 |
| Base de données | Apache Cassandra |
| Indexation | Elasticsearch 7.x |
| Runtime Java | Java 11 (Amazon Corretto) |

---

## 3. Prérequis

- Dépendances système : `wget`, `gnupg`, `apt-transport-https`, `ca-certificates`, `software-properties-common`
- Java 11 (Amazon Corretto — recommandé officiellement par TheHive pour la compatibilité avec Cassandra/Elasticsearch)
- Ports libres : `9042` (Cassandra), `9200` (Elasticsearch), `9000` (TheHive)

---

## 4. Installation

### 4.1 – Dépendances et Java 11 (Amazon Corretto)

```bash
sudo apt update
sudo apt install -y wget gnupg apt-transport-https ca-certificates software-properties-common
wget -O- https://apt.corretto.aws/corretto.key | sudo apt-key add -
sudo add-apt-repository 'deb https://apt.corretto.aws stable main'
sudo apt update
sudo apt install -y java-11-amazon-corretto-jdk
```

![Installation des dépendances et de Java 11](screenshots/thehive/01-dependencies-java-install.png)

### 4.2 – Apache Cassandra

```bash
echo "deb https://debian.cassandra.apache.org 40x main" | sudo tee /etc/apt/sources.list.d/cassandra.sources.list
curl https://downloads.apache.org/cassandra/KEYS | sudo apt-key add -
sudo apt update
sudo apt install -y cassandra
```

### 4.3 – Elasticsearch 7.x

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install -y elasticsearch
```

### 4.4 – TheHive 5.2

```bash
wget -qO - https://raw.githubusercontent.com/TheHive-Project/TheHive/master/PGP-PUBLIC-KEY | sudo apt-key add -
echo "deb https://deb.thehive-project.org release main" | sudo tee /etc/apt/sources.list.d/thehive-project.list
sudo apt update
sudo apt install -y thehive
```

![Installation de TheHive](screenshots/thehive/02-thehive-install.png)

---

## 5. Configuration

### 5.1 – Cassandra (`/etc/cassandra/cassandra.yaml`)

```yaml
cluster_name: 'thp'
listen_address: 192.168.9.133
rpc_address: 192.168.9.133
seeds: "192.168.9.133,7000"
```

![Configuration Cassandra — cluster/listen](screenshots/thehive/03-cassandra-yaml-cluster.png)
![Configuration Cassandra — listen_address/rpc_address](screenshots/thehive/04-cassandra-listen-rpc-address.png)
![Configuration Cassandra — seeds](screenshots/thehive/05-cassandra-seeds.png)

### 5.2 – Elasticsearch (`/etc/elasticsearch/elasticsearch.yml`)

```yaml
network.host: 192.168.9.133
cluster.initial_master_nodes: ["node-1"]
```

![Configuration Elasticsearch — network.host](screenshots/thehive/06-elasticsearch-network-host.png)
![Configuration Elasticsearch — discovery seed hosts](screenshots/thehive/07-elasticsearch-discovery-seeds.png)

### 5.3 – TheHive (`/etc/thehive/application.conf`)

```hocon
application.baseUrl = "http://192.168.9.133:9000"

db.janusgraph {
  storage {
    backend = cql
    hostname = ["192.168.9.133"]
    cql {
      cluster-name = thp
      keyspace = thehive
    }
  }
  index.search {
    backend = elasticsearch
    hostname = ["192.168.9.133"]
    index-name = thehive
  }
}

storage {
  provider = locals
  locals.location = /opt/thp/thehive/files
}
```

![Configuration TheHive — section base de données](screenshots/thehive/09-thehive-application-conf-db.png)
![Configuration TheHive — baseUrl](screenshots/thehive/10-thehive-application-conf-baseurl.png)

---

## 6. Démarrage et vérification des services

```bash
sudo systemctl enable --now cassandra
sudo systemctl enable --now elasticsearch
sudo systemctl enable --now thehive
```

```bash
sudo systemctl status cassandra
```
![Statut Cassandra](screenshots/thehive/08-cassandra-status.png)

```bash
sudo systemctl status elasticsearch
```
![Statut Elasticsearch](screenshots/thehive/11-elasticsearch-status.png)

```bash
sudo systemctl status thehive
```

⚠️ **Ordre de démarrage important :** Cassandra doit être pleinement opérationnel (`nodetool status` → `UN` = Up/Normal) avant qu'Elasticsearch et TheHive ne démarrent proprement — un démarrage simultané à froid échoue souvent. Vérifier avec `nodetool status` en cas de souci au premier lancement.

---

## 7. Connexion initiale

Interface web : **`http://192.168.9.133:9000`**

**Identifiants par défaut :**
| Champ | Valeur |
|---|---|
| Login | `admin@thehive.local` |
| Mot de passe | `secret` |

![Page de connexion TheHive](screenshots/thehive/12-thehive-login-page.png)
![Liste des organisations après connexion](screenshots/thehive/13-thehive-organisation-list.png)

⚠️ **Changer ce mot de passe immédiatement après la première connexion** — identifiants par défaut publics et bien connus.

---

## 8. Intégration avec Wazuh (prévue — à documenter séparément)

Flux prévu, une fois câblé :

```
Wazuh (alerte) → Shuffle (webhook + logique de playbook) → TheHive (création automatique d'un cas)
```

Shuffle appellera l'API TheHive (`POST /api/v1/case`) pour créer un cas structuré à partir de chaque alerte à haute sévérité, avec les observables (IPs, hashs) extraits automatiquement de l'alerte source. Détail des étapes à ajouter une fois cette intégration testée (cf. `config/shuffle.md` pour l'état actuel de Shuffle).

---

## 9. Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Cassandra ne démarre pas | RAM insuffisante (Cassandra est gourmand par défaut) | Réduire `MAX_HEAP_SIZE` / `HEAP_NEWSIZE` dans `/etc/cassandra/cassandra-env.sh`, ou allouer plus de RAM à la VM |
| Elasticsearch échoue au démarrage | `vm.max_map_count` trop bas (limite noyau Linux) | `sudo sysctl -w vm.max_map_count=262144` (et le rendre persistant dans `/etc/sysctl.conf`) |
| TheHive reste bloqué sur "waiting for Cassandra" | Cassandra pas encore prêt, ou `listen_address`/`rpc_address` mal alignés avec l'IP réelle de la VM | Vérifier `nodetool status`, confirmer que l'IP dans `cassandra.yaml` correspond à l'IP réelle de Security-Core |
| Conflit de port (9000, 9042, 9200) | Un autre service (MISP, Wazuh dashboard...) occupe déjà le port | `sudo ss -tlnp \| grep <port>` pour identifier le conflit — cf. le même type de conflit déjà rencontré avec privacyIDEA/MISP sur le port 8443 |
| Erreurs SSL au premier accès | Certificat auto-signé ou HTTPS non configuré | Accéder en HTTP simple (`http://`, pas `https://`) tant que TLS n'est pas explicitement configuré |

---

## 10. Statut actuel

✅ Installation et configuration de base terminées (Cassandra, Elasticsearch, TheHive, tous démarrés).
⏳ Intégration avec Wazuh/Shuffle — prévue, pas encore réalisée (section 8).

> 🚧 Captures en attente — les emplacements ci-dessus sont prêts à recevoir les images une fois envoyées.
