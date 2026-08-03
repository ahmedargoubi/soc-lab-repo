# Installation & Configuration — Active Directory

Configuration du contrôleur de domaine et des objets Active Directory pour le lab, sur la zone **Server_LAN** (`192.168.7.0/24`).

---

## 1. Contrôleur de domaine

| Paramètre | Valeur |
|---|---|
| Hostname | `TEKUP-DC` |
| OS | Windows Server 2022 |
| Domaine | `PROJET.local` |
| IP |`192.168.7.139` |

### 1.1 – Promotion en Domain Controller

Le rôle **Active Directory Domain Services** a été installé et configuré via l'assistant AD DS Configuration Wizard. La promotion s'est terminée avec succès (`This server was successfully configured as a domain controller`), avec un redémarrage automatique du serveur.

**Avertissement noté pendant l'installation :** Windows Server 2022 applique par défaut le paramètre de sécurité *"Allow cryptography algorithms compatible with Windows NT 4.0"*, qui désactive les algorithmes de chiffrement faibles lors de l'établissement des canaux sécurisés — comportement par défaut, aucune action requise.

### 1.2 – DNS

Le rôle DNS Server a été installé conjointement à la promotion AD DS. La zone `TEKUP-DC.PROJET.local` est créée automatiquement et visible dans la console **DNS Manager**.

Deux avertissements récurrents (Event ID 4013, `Microsoft-Windows-DNS-Server-Service`) ont été observés dans les logs DNS — à documenter/investiguer si le comportement persiste après durcissement.

---

## 2. Essais de jonction de domaine (avant configuration DNS correcte)

Avant que le contrôleur de domaine soit pleinement fonctionnel, plusieurs tentatives de jonction ont échoué avec le message *"That domain couldn't be found. Check the domain name and try again."* :

| Nom de domaine testé | Résultat |
|---|---|
| `PROJET.local` | ❌ Domaine introuvable |
| `TEKUP.local` | ❌ Domaine introuvable |

Ces échecs correspondent à une phase où le DNS des clients ne pointait probablement pas encore vers `TEKUP-DC`, ou le contrôleur n'était pas encore pleinement promu. La jonction a fini par réussir une fois `PROJET.local` pleinement opérationnel (voir section 3).

---

## 3. Postes clients joints au domaine

| Nom de PC (avant jonction) | Nom final | Type |
|---|---|---|
| `DESKTOP-A9LLDAQ` | `AHMED` | Computer (Win10-Client) |
| `DESKTOP-A9LLDAQ` | `HAROUN` | Computer (Win10-Client) |

Les deux machines apparaissent dans **Active Directory Users and Computers → PROJET.local → Computers** une fois la jonction réussie.

---

## 4. Comptes utilisateurs créés

Plusieurs comptes ont été créés par copie d'utilisateurs existants (`Copy Object - User`), avec l'option **"Password never expires"** activée systématiquement.

| Utilisateur créé | Copié depuis | Login | Notes |
|---|---|---|---|
| **SQL service** (`SQLservice`) | Haroun Rashid | `SQLservice@PROJET.local` | Compte de service — candidat probable pour Kerberoasting (SPN attendu) |
| **Mohamed arg** (`Mohamed`) | Ahmed arg | `Mohamed@PROJET.local` | Copié depuis un compte standard |

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

⚠️ Ce niveau de privilège sur un compte utilisateur standard est **la faille exploitée dans la Simulation 3 (WannaCry)** — `haroun` étant Domain Admin, le ransomware exécuté sous son contexte a pu désactiver Windows Defender sans invite UAC. Voir `reports/phase-a-baseline/simulation-3-wannacry-ransomware.md`, section 3.5 et 5.3.

**mohamed arg** — compte standard, non privilégié :

| Groupe |
|---|
| Domain Users |

---

## 5. Récapitulatif des faiblesses intentionnelles introduites

| Faiblesse | Objectif pédagogique |
|---|---|
| `haroun` cumule Domain Admins + Enterprise Admins + Schema Admins | Démontrer l'impact d'un compte utilisateur sur-privilégié (exploité en Sim 3 et lié aux chemins BloodHound de la Sim 4) |
| Compte `SQLservice` créé avec mot de passe n'expirant jamais | Cible probable pour Kerberoasting (SPN à confirmer) |
| Mot de passe faible sur `ahmed` (`TempPass123!`, cf. Simulation 4) | Démontrer la faisabilité du cassage offline après capture NTLMv2 |

---

## 6. Captures associées

| Étape | Référence |
|---|---|
| Renommage des postes clients (AHMED, HAROUN) | Captures `Rename your PC` |
| Échecs de jonction (`PROJET.local`, `TEKUP.local`) | Captures `Join a domain` |
| Jonction réussie — postes visibles dans ADUC | `Active Directory Users and Computers` |
| DNS Manager — zone `TEKUP-DC.PROJET.local` | `DNS Manager` |
| Server Manager — IPs et événements DNS du DC | `Server Manager → DNS` |
| Création `SQLservice` (copie depuis Haroun Rashid) | `Copy Object - User` |
| Création `Mohamed arg` (copie depuis Ahmed arg) | `Copy Object - User` |
| Groupes de `Haroun Rashid` | `Haroun Rashid Properties → Member Of` |
| Groupes de `mohamed arg` | `mohamed arg Properties → Member Of` |
| Promotion DC terminée | `Active Directory Domain Services Configuration Wizard → Results` |
