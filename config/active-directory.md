# Installation & Configuration — Active Directory

Configuration du contrôleur de domaine et des objets Active Directory pour le lab, sur la zone **Server_LAN** (`192.168.7.0/24`).

---

## 1. Contrôleur de domaine

| Paramètre | Valeur |
|---|---|
| Hostname | `TEKUP-DC` |
| OS | Windows Server 2022 Standard Evaluation |
| Domaine | `PROJET.local` |
| IP | `192.168.7.139` |
| Forest functional level | `Windows Server 2016` |
| Domain functional level | `Windows Server 2016` |
| NetBIOS domain name | `PROJET` |
| DSRM password | Défini lors de la promotion (non visible) |

### 1.1 – Promotion en Domain Controller

Le rôle **Active Directory Domain Services** a été installé et configuré via l'assistant AD DS Configuration Wizard. La promotion s'est terminée avec succès (`This server was successfully configured as a domain controller`), avec un redémarrage automatique du serveur.

![Add Roles and Features - Installation Type](AD_screens/cap2.png)<br>
*Sélection du type d'installation : "Role-based or feature-based installation".*

![Add Roles and Features - Server Roles](AD_screens/cap13.png)<br>
*Sélection du rôle "Active Directory Domain Services".*

![AD DS Configuration - Deployment Configuration](AD_screens/cap40.png)<br>
*Déploiement : "Add a new forest" avec le domaine `PROJET.local`.*

![AD DS Configuration - Domain Controller Options](AD_screens/cap41.png)<br>
*Options du contrôleur de domaine : DNS Server, Global Catalog, DSRM password.*

![AD DS Configuration - Additional Options](AD_screens/cap42.png)<br>
*NetBIOS domain name : `PROJET`.*

![AD DS Configuration - Paths](AD_screens/cap3.png)<br>
*Chemins par défaut pour la base de données NTDS, les logs et SYSVOL.*

![AD DS Configuration - Confirmation](AD_screens/cap35.png)<br>
*Confirmation des rôles et fonctionnalités à installer.*

![AD DS Configuration - Results](AD_screens/cap38.png)<br>
*Installation réussie avec message "Configuration required. Installation succeeded".*

**Avertissement noté pendant l'installation :** Windows Server 2022 applique par défaut le paramètre de sécurité *"Allow cryptography algorithms compatible with Windows NT 4.0"*, qui désactive les algorithmes de chiffrement faibles lors de l'établissement des canaux sécurisés — comportement par défaut, aucune action requise.

### 1.2 – Renommage du serveur

Le serveur a été renommé `TEKUP-DC` avant ou pendant la promotion, comme visible dans les paramètres système.

![Rename your PC - TEKUP-DC](AD_screens/cap39.png)
*Renommage du serveur en `TEKUP-DC`.*

### 1.3 – DNS

Le rôle DNS Server a été installé conjointement à la promotion AD DS. La zone `TEKUP-DC.PROJET.local` est créée automatiquement et visible dans la console **DNS Manager**.

![DNS Manager - TEKUP-DC.PROJET.local](AD_screens/cap5.png)<br>
*Zone DNS `TEKUP-DC.PROJET.local` visible dans DNS Manager.*

**Tests de résolution (capture `cap37`) :**

```cmd
C:\Users\ahmed>ping 192.168.7.139
Réponses reçues (0% perte)

C:\Users\ahmed>nslookup PROJET.local
Server:  Unknown
Address: 192.168.7.139
Name:    PROJET.local
Address: 192.168.7.139
```

![Ping et nslookup vers TEKUP-DC](AD_screens/cap37.png)<br>
*Ping et résolution DNS vers le contrôleur de domaine.*

---




## 2. Postes clients joints au domaine

| Nom de PC (avant jonction) | Nom final | Type |
|---|---|---|
| `DESKTOP-A9LLDAQ` | `AHMED` | Computer (Win10-Client) |
| `DESKTOP-A9LLDAQ` | `HAROUN` | Computer (Win10-Client) |

Les deux machines apparaissent dans **Active Directory Users and Computers → PROJET.local → Computers** une fois la jonction réussie.

![ADUC - Postes AHMED et HAROUN](AD_screens/cap9.png)<br>
*Postes AHMED et HAROUN visibles dans le conteneur Computers.*

![ADUC - Conteneur Computers](AD_screens/cap5.png)<br>
*Vue d'ensemble du conteneur Computers avec les postes joints.*

---

## 3. Comptes utilisateurs créés

Plusieurs comptes ont été créés par copie d'utilisateurs existants (`Copy Object - User`), avec l'option **"Password never expires"** activée systématiquement.

| Utilisateur créé | Copié depuis | Login | Notes |
|---|---|---|---|
| Ahmed arg (`ahmed`) | - | `ahmed@PROJET.local` | Compte standard (créé directement) |
| Haroun Rashid (`Haroun`) | Administrator | `Haroun@PROJET.local` | Compte privilégié (copié depuis Admin) |
| SQL service (`SQLservice`) | Haroun Rashid | `SQLservice@PROJET.local` | Compte de service — candidat probable pour Kerberoasting |
| Mohamed arg (`Mohamed`) | Ahmed arg | `Mohamed@PROJET.local` | Copié depuis un compte standard |

![Création du compte ahmed](AD_screens/cap11.png)<br>
*Création du compte ahmed.*

![Création du compte Haroun Rashid](AD_screens/cap14.png)<br>
*Création du compte Haroun Rashid.*

![Copie Haroun Rashid - Password never expires](AD_screens/cap15.png)<br>
*Copie du compte Haroun Rashid avec option "Password never expires".*

![Création du compte Mohamed arg](AD_screens/cap16.png)<br>
*Création du compte Mohamed arg.*

![Password never expires](AD_screens/cap17.png)<br>
*Définition du mot de passe avec option "Password never expires".*

### 4.1 – Appartenance aux groupes

**Haroun Rashid** — compte fortement privilégié :

| Groupe |
|---|
| Administrators (Builtin) |
| Domain Admins |
| Domain Users |
| Enterprise Admins |
| Group Policy Creator Owners |
| Schema Admins |

![Groupes de Haroun Rashid](config_screens/cap55.png)<br>
*Appartenance de Haroun Rashid aux groupes Domain Admins, Enterprise Admins, Schema Admins, etc.*

⚠️ Ce niveau de privilège sur un compte utilisateur standard est **la faille exploitée dans la Simulation 3 (WannaCry)** — `haroun` étant Domain Admin, le ransomware exécuté sous son contexte a pu désactiver Windows Defender sans invite UAC. Voir `reports/phase-a-baseline/simulation-3-wannacry-ransomware.md`, section 3.5 et 5.3.

**Mohamed arg** — compte standard, non privilégié :

| Groupe |
|---|
| Domain Users |

---

## 4. Service Principal Names (SPN) – candidats Kerberoasting

Un SPN a été explicitement ajouté pour le compte `SQLservice` :

```cmd
setspn -a TEKUP-DC/SQLservice.PROJET.local:60111 PROJET\SQLservice
```

![Ajout du SPN SQLservice](AD_screens/cap23.png)<br>
*Ajout du SPN `TEKUP-DC/SQLservice.PROJET.local:60111` pour le compte `PROJET\SQLservice`.*

Cela rend le compte `SQLservice` Kerberoastable (car il possède un SPN). Le mot de passe du compte ayant été défini avec Password never expires, il constitue une cible idéale pour une attaque Kerberoasting — exploitée dans la Simulation 4 (`reports/phase-a-baseline/simulation-4-active-directory.md`).

La liste complète des SPN du domaine est visible avec la commande :

```cmd
setspn -T PROJET.local -Q */*
```

![Liste des SPN du domaine](AD_screens/cap25.png)<br>
*Liste des SPN du domaine PROJET.local, incluant celui de SQLservice.*

Un autre SPN est visible dans les captures (`HYDRA-DC/SQLService.MARVEL.local:60111`) mais il s'agit d'un test antérieur (domaine MARVEL.local) ; il n'est pas actif dans PROJET.local.

---

## 5. Gestion des stratégies de groupe (GPO) — faiblesses intentionnelles

Plusieurs GPO ont été créées ou modifiées pour affaiblir la sécurité du domaine à des fins pédagogiques :

| GPO | Paramètre | Impact |
|---|---|---|
| Turn off Microsoft Defender Antivirus | Enabled | Désactive Microsoft Defender sur les postes clients, permettant l'exécution de ransomwares (Simulation 3) sans être bloqués. |

La GPO est visible dans la console Group Policy Management et liée au domaine PROJET.local.

![Group Policy Management](AD_screens/cap26.png)<br>
*Console Group Policy Management avec la GPO créée.*

![Paramètre GPO - Defender désactivé](AD_screens/cap27.png)<br>
*Paramètre de la GPO : "Turn off Microsoft Defender Antivirus" activé.*

⚠️ Cette GPO a permis au ransomware WannaCry (Simulation 3) de s'exécuter sans être bloqué par Windows Defender, car le compte haroun (Domain Admin) a pu désactiver la protection.



## 6. Récapitulatif des faiblesses intentionnelles introduites

| Faiblesse | Objectif pédagogique | Simulation liée |
|---|---|---|
| haroun cumule Domain Admins + Enterprise Admins + Schema Admins | Démontrer l'impact d'un compte utilisateur sur-privilégié | Sim 3 (WannaCry), Sim 4 (BloodHound) |
| Compte SQLservice avec SPN et mot de passe n'expirant jamais | Cible Kerberoasting | Sim 4 (AD Attack) |
| Mot de passe faible sur ADMINSTRATOR (TempPass123!) | Démontrer la faisabilité du cassage offline après capture NTLMv2 | Sim 4 (LLMNR Poisoning) |
| GPO désactivant Microsoft Defender | Permettre l'exécution de ransomware sans détection AV | Sim 3 (WannaCry) |
| Partage hackme accessible à tous | Simuler un point d'exfiltration ou de stockage de payloads | Sim 2 (pivot) |
| Comptes avec Password never expires | Faciliter le Kerberoasting et la persistance | Sim 4 (AD Attack) |
