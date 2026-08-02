# Velociraptor — Configuration Guide

Guide complet de l'installation du serveur (déploiement natif, pas Docker — voir justification section 2), génération de la config client, et déploiement des agents.

## 1. Préparation

```bash
mkdir ~/velociraptor
cd ~/velociraptor
```

### Téléchargement du binaire (dernière version Linux)
```bash
curl -s https://api.github.com/repos/Velocidex/velociraptor/releases/latest \
  | grep "browser_download_url.*linux-amd64\"" \
  | cut -d '"' -f 4 \
  | xargs curl -L -o velociraptor
chmod +x velociraptor
```

## 2. Pourquoi déploiement natif et non Docker

Une tentative initiale de déploiement via Docker Compose a échoué de façon persistante : le service était accessible en local (`127.0.0.1`) mais refusé depuis toute autre machine, malgré des règles de pare-feu correctes et un routage vérifié. La cause réelle, découverte après un long dépannage réseau, était que **`GUI.bind_address` dans `server.config.yaml` était configuré sur `127.0.0.1` par défaut** — un problème de configuration applicative, pas réseau. Le déploiement natif a été conservé ensuite pour la simplicité (même modèle que Wazuh, qui tourne aussi nativement).

## 3. Génération de la configuration serveur

```bash
./velociraptor config generate -i
```

Répondre aux invites interactives :
| Question | Réponse |
|---|---|
| Deployment type | self signed SSL |
| Server type | Linux |
| Public name / hostname | **IP réelle de Security-Core** (ex: `192.168.9.133`) — **ne pas laisser `localhost`** |
| DNS Type | None - Configure DNS manually |
| Websocket experimental | No |
| Frontend port | 8000 |
| GUI port | 8889 |
| Datastore location | `/opt/velociraptor` (défaut) |
| Certificate expiration | 2 Years |

Résultat : génère `server.config.yaml` dans le répertoire courant.

## 4. Création de l'utilisateur admin

```bash
./velociraptor --config server.config.yaml user add admin --role administrator
```

## 5. Génération de la configuration client (pour les agents)

```bash
./velociraptor --config server.config.yaml config client > client.config.yaml
```

## 6. Déploiement en service natif (systemd)

```bash
sudo cp ~/velociraptor/velociraptor /usr/local/bin/velociraptor
sudo chmod +x /usr/local/bin/velociraptor
sudo mkdir -p /etc/velociraptor
sudo cp ~/velociraptor/server.config.yaml /etc/velociraptor/server.config.yaml
```

```bash
sudo tee /etc/systemd/system/velociraptor.service << 'EOF'
[Unit]
Description=Velociraptor
After=network.target

[Service]
ExecStart=/usr/local/bin/velociraptor --config /etc/velociraptor/server.config.yaml frontend -v
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable velociraptor
sudo systemctl start velociraptor
sudo systemctl status velociraptor
```

## 7. ⚠️ Correctif critique — GUI accessible uniquement en local

Après démarrage, vérifier l'adresse d'écoute réelle :
```bash
sudo ss -tulnp | grep -E '8000|8889'
```

**Si `8889` apparaît en `127.0.0.1:8889` au lieu de `0.0.0.0:8889`** — c'est le bug root-cause documenté en section 2. Corriger :
```bash
sudo nano /etc/velociraptor/server.config.yaml
```
Chercher la section `GUI:` et forcer :
```yaml
GUI:
  bind_address: 0.0.0.0
```
```bash
sudo systemctl restart velociraptor
sudo ss -tulnp | grep 8889   # doit maintenant montrer 0.0.0.0:8889
```

## 8. Accès au dashboard

```
https://<IP-Security-Core>:8889
```
Identifiants : le compte `admin` créé à l'étape 4.

## 9. Firewall — ports requis

| Port | Usage |
|---|---|
| 8000 | Communication agent ↔ frontend (seul port nécessaire pour les endpoints) |
| 8889 | Interface GUI web (accès analyste uniquement, pas les agents) |

Alias OPNsense `Velociraptor_Client_Port` = `8000` uniquement — les agents n'ont jamais besoin d'atteindre le port 8889.

## 10. Déploiement des agents

### 10.1 — Transfert de la configuration client
Comme les machines cibles n'ont pas toutes un accès internet direct, la méthode la plus fiable dans ce lab a été de copier-coller le contenu du fichier directement (via la console VMware) :
```bash
cat ~/velociraptor/client.config.yaml
```
Copier la sortie complète, puis sur la machine cible :
```bash
sudo mkdir -p /etc/velociraptor
sudo nano /etc/velociraptor/client.config.yaml
# coller le contenu, sauvegarder
```

### 10.2 — Agent Linux
```bash
sudo curl -L -o /usr/local/bin/velociraptor \
  https://github.com/Velocidex/velociraptor/releases/download/v0.76/velociraptor-v0.76.3-linux-amd64
sudo chmod +x /usr/local/bin/velociraptor
```
```bash
sudo tee /etc/systemd/system/velociraptor-client.service << 'EOF'
[Unit]
Description=Velociraptor Client
After=network.target

[Service]
ExecStart=/usr/local/bin/velociraptor --config /etc/velociraptor/client.config.yaml client -v
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable velociraptor-client
sudo systemctl start velociraptor-client
```

### 10.3 — Agent Windows
```powershell
mkdir C:\velociraptor
notepad C:\velociraptor\client.config.yaml
# coller le contenu de client.config.yaml, sauvegarder

Invoke-WebRequest -Uri https://github.com/Velocidex/velociraptor/releases/download/v0.76/velociraptor-v0.76.3-windows-amd64.exe -OutFile C:\velociraptor\velociraptor.exe

C:\velociraptor\velociraptor.exe --config C:\velociraptor\client.config.yaml service install
Get-Service Velociraptor
```

## 11. Vérification des agents connectés

Dans l'interface web : **Show all clients** (icône loupe/recherche). Chaque endpoint doit apparaître avec un statut actif (point vert) et son OS/hostname corrects.

## 12. Requêtes VQL utiles (exemples utilisés pendant les investigations DFIR)

| Objectif | Requête VQL |
|---|---|
| Lister les processus actifs | `SELECT Pid, Name, CommandLine FROM pslist()` |
| Trouver des fichiers chiffrés (ransomware) | `SELECT FullPath, Size, Mtime FROM glob(globs='C:\\Users\\**\\*.WNCRYT')` |
| Vérifier une clé de persistance registre | `SELECT * FROM read_reg_key(regpath='HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run')` |
| Connexions réseau vers une IP donnée | `SELECT * FROM netstat() WHERE RemoteAddress =~ '<IP>'` |
| Hash de fichiers (intégrité de preuve) | `SELECT FullPath, SHA256(FullPath) AS SHA256 FROM glob(globs='C:\\Windows\\*.exe')` |
| Journaux d'événements Windows (ex: logon) | `SELECT * FROM read_eventlog(channel='Security', count=500) WHERE EventID = 4624` |
