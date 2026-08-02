# Wazuh — Configuration Guide

Guide complet de l'installation du manager, du déploiement des agents (Linux/Windows), et des correctifs de configuration appliqués pendant le build.

## 1. Installation du Manager (Security-Core)

### Prérequis
- Ubuntu Server (26.04 dans ce lab)
- Minimum 4 Go de RAM, 50+ Go de disque (les indices Wazuh grossissent vite)

### Installation officielle (script all-in-one)
```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```
Cette commande installe et configure ensemble : Wazuh indexer, Wazuh manager, Wazuh dashboard, Filebeat.

**Conserver précieusement les identifiants générés en fin d'installation** (admin + mot de passe du dashboard). Pour les récupérer plus tard :
```bash
sudo tar -xf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O
```

### Accès au dashboard
```
https://<IP-Security-Core>
```

![Phase A Topology](config_images/cap40.png)


## 2. Vérification de l'état du manager

```bash
sudo systemctl status wazuh-manager
sudo /var/ossec/bin/wazuh-control status
```

![Phase A Topology](config_images/cap41.png)



Tous les daemons doivent afficher `is running` : `wazuh-modulesd`, `wazuh-monitord`, `wazuh-logcollector`, `wazuh-remoted`, `wazuh-syscheckd`, `wazuh-analysisd`, `wazuh-execd`, `wazuh-db`, `wazuh-authd`, `wazuh-integratord`, `wazuh-apid`.


![Phase A Topology](config_images/cap42.png)

**Si plusieurs daemons montrent `failed`** (bug rencontré à plusieurs reprises dans ce lab, souvent lié à la pression mémoire causée par Docker/MISP/Velociraptor tournant en parallèle sur la même VM) :
```bash
sudo systemctl restart wazuh-manager
sleep 20
sudo /var/ossec/bin/wazuh-control status
```

## 3. Gestion des agents (manager)

### Lister les agents enregistrés
```bash
sudo /var/ossec/bin/agent_control -l
```

### Ajouter / supprimer / extraire une clé manuellement
```bash
sudo /var/ossec/bin/manage_agents
# (A)dd, (E)xtract key, (L)ist, (R)emove, (Q)uit
```


![Phase A Topology](config_images/cap43.png)

### ⚠️ Piège récurrent : "Duplicate agent name"
Si un agent est réinstallé/renommé sans avoir été explicitement supprimé du manager au préalable, l'enregistrement échoue avec `ERROR: Duplicate agent name`. **Toujours supprimer l'ancien agent (option R) avant tout nouvel essai d'enrôlement** — ne jamais relancer `agent-auth` plusieurs fois de suite sans nettoyer, cela empile des ID orphelins (022 → 023 → 024...).

## 4. Déploiement agent — Linux (Debian/Ubuntu)

```bash
curl -so wazuh-agent.deb \
  https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.2-1_amd64.deb
sudo WAZUH_MANAGER='192.168.9.133' dpkg -i ./wazuh-agent.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

![Phase A Topology](config_images/cap32.png) <br>
![Phase A Topology](config_images/cap33.png)


## 5. Déploiement agent — Linux (CentOS/RHEL)

```bash
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

sudo tee /etc/yum.repos.d/wazuh.repo << 'EOF'
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protect=1
EOF

sudo WAZUH_MANAGER='192.168.9.133' yum install wazuh-agent -y
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

![Phase A Topology](config_images/cap21.png) <br>
![Phase A Topology](config_images/cap22.png) <br>
![Phase A Topology](config_images/cap23.png) <br>


## 6. Déploiement agent — Windows

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi -OutFile "$env:tmp\wazuh-agent.msi"
msiexec.exe /i "$env:tmp\wazuh-agent.msi" /q WAZUH_MANAGER='192.168.9.133'
NET START WazuhSvc
Get-Service WazuhSvc
```

> **Important** : conserver l'extension `.msi` dans le nom du fichier téléchargé — un fichier renommé sans extension peut faire échouer silencieusement le passage du paramètre `WAZUH_MANAGER`, provoquant une adresse `0.0.0.0` dans la config finale (voir section 8).

## 7. Firewall — ports requis

Sur OPNsense, ouvrir entre chaque zone contenant un agent et Security_LAN :
- **1514/tcp** — trafic d'événements agent → manager
- **1515/tcp** — enrôlement (`wazuh-authd`)

Voir l'alias `Wazuh_Ports` documenté dans [`opnsense.md`](opnsense.md).

## 8. Résolution de problèmes fréquents

### 8.1 — Agent "Pending" ou "Disconnected" indéfiniment
```bash
# Sur l'agent, vérifier la connectivité au port :
Test-NetConnection -ComputerName 192.168.9.133 -Port 1514    # Windows
nc -zv 192.168.9.133 1514                                     # Linux

# Vérifier le log de l'agent :
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30   # Windows
sudo tail -f /var/ossec/logs/ossec.log                                 # Linux
```

![Phase A Topology](config_images/cap1.png)

### 8.2 — Adresse manager `0.0.0.0` dans la config
```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```
Corriger manuellement :
```xml
<client>
  <server>
    <address>192.168.9.133</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

![Phase A Topology](config_images/cap20.png)


### 8.3 — Ré-enrôlement propre après un échec
```powershell
Stop-Service WazuhSvc
Remove-Item "C:\Program Files (x86)\ossec-agent\client.keys" -Force -ErrorAction SilentlyContinue
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m 192.168.9.133
Start-Service WazuhSvc
```
**Ne pas oublier** de supprimer l'ancien agent côté manager avant (section 3).

### 8.4 — Routage asymétrique (cause racine de nombreux échecs de connexion)
Si les tests de port échouent malgré des règles de pare-feu correctes, vérifier la table de routage de **Security-Core** :
```bash
ip route get <IP-du-client>
```
Si la réponse indique une route via l'interface NAT (`ens33`) au lieu de l'interface Security_LAN (`ens192`), ajouter :
```bash
sudo ip route add <subnet-du-client>/24 via 192.168.9.1 dev ens192
```
Rendre permanent via `/etc/netplan/50-cloud-init.yaml` :
```yaml
      routes:
        - to: <subnet-du-client>/24
          via: 192.168.9.1
```
```bash
sudo netplan apply
```

## 9. File Integrity Monitoring (FIM / syscheck) — pièges spécifiques Windows

Le module `<syscheck>` de l'agent Windows a besoin de **deux lignes qui ne sont PAS présentes par défaut** et sont **réinitialisées à chaque réinstallation** :

```xml
<syscheck>
  <disabled>no</disabled>
  <scan_on_start>yes</scan_on_start>
  <alert_new_files>yes</alert_new_files>
  ...
```

De plus, **éviter les chemins avec wildcard** (`C:\Users\*\Desktop`) pour la surveillance temps réel — peu fiables sur Windows. Utiliser des chemins littéraux :
```xml
<directories realtime="yes">C:\Users\haroun\Desktop</directories>
<directories realtime="yes">C:\Users\haroun\Downloads</directories>
<directories realtime="yes">C:\Users\haroun\Documents</directories>
```

Vérifier l'initialisation après redémarrage du service :
```powershell
Restart-Service WazuhSvc
Start-Sleep -Seconds 30
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30 | Select-String "syscheck"
```

## 10. Règles personnalisées (local_rules.xml)

Emplacement : `/var/ossec/etc/rules/local_rules.xml` (sur le manager)

Exemple — détection de ransomware (fichiers chiffrés + volume élevé de créations) :
```xml
<group name="ransomware,wannacry">
  <rule id="100001" level="12">
    <if_sid>554</if_sid>
    <field name="syscheck.path" type="pcre2">(?i)\.WNCRYT$</field>
    <description>WannaCry ransomware detected: Encrypted file created at $(syscheck.path).</description>
  </rule>

  <rule id="100004" level="10" frequency="12" timeframe="60">
    <if_matched_sid>554</if_matched_sid>
    <same_source_ip />
    <description>Possible ransomware activity: High number of files created in a short time.</description>
  </rule>
</group>
```

Après toute modification, redémarrer le manager :
```bash
sudo systemctl restart wazuh-manager
```

## 11. Intégrations activées

### VirusTotal (dans `/var/ossec/etc/ossec.conf` du manager)
```xml
<integration>
  <name>virustotal</name>
  <api_key>VOTRE_CLE_API</api_key>
  <rule_id>100001,100003,100004</rule_id>
  <alert_format>json</alert_format>
</integration>
```

### YARA
```xml
<wodle name="yara">
  <rules>/var/ossec/etc/yara/index.yar</rules>
</wodle>
```
