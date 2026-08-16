# OPNsense — Règles de pare-feu après durcissement (Phase B)

Jeu de règles durci, construit à partir de la baseline Phase A
([`opnsense-firewall-rules.md`](opnsense-firewall-rules.md)) selon le
principe du **moindre privilège strict** : chaque règle n'est conservée
que si elle correspond à un besoin opérationnel réel et identifiable.
Aucune règle "tout port / tout protocole" n'est conservée nulle part dans
le réseau après ce durcissement.

> ℹ️ Toutes les interfaces restent en **first-match / deny-by-default**.
> Ce document liste les règles manuelles à configurer — tout ce qui n'y
> figure pas explicitement reste bloqué.

---

## 0. Comparaison Avant / Après

| Interface | Avant (Phase A) | Après (Phase B) | Effet |
|---|---|---|---|
| **WAN** | DMZ ouverte sur tous les ports. Legacy ouverte sur tout protocole. | DMZ limitée à `80, 443`. Legacy limitée à `21, 6200` (ports réellement exploités). | Le point d'entrée de l'attaquant est réduit exactement à ce que chaque service expose réellement. |
| **userlan** | Sortie WAN illimitée (tout port/protocole). | Web (`80/443`) + DNS (`53`) uniquement. | Un poste compromis ne peut pas ouvrir un canal de sortie arbitraire. |
| **serverlan** | Règle `→ 192.168.163.164/24 (tout)` vers l'attaquant. Règle catch-all `→ securitylan (tout)`. | Les deux retirées. | Ferme un chemin de pivot pré-construit qui n'a aucun rapport avec une technique d'attaque réelle. |
| **dmz** | Règle `→ 192.168.163.164 (tout)` vers l'attaquant. ICMP sortant illimité. | Retirée / restreinte à Security_LAN. | Même raisonnement — aucun canal de retour standing vers Kali. |
| **legacy** | Aucune règle manuelle, mais entrée WAN grande ouverte. | Aucune règle manuelle, entrée WAN strictement scoping (voir Section 0bis). | Reste volontairement non instrumentée (pas d'agent), mais n'est plus grande ouverte pour autant — ce sont deux décisions différentes, traitées séparément. |
| **securitylan** | **"Allow everything to anywhere"** — la règle la plus permissive de tout le réseau, sur la zone qui héberge Wazuh/TheHive/Shuffle/MISP. | 3 flux nommés : HTTPS (443), SMTP (587), DNS (53). | La zone responsable de la détection et de la réponse était, avant durcissement, la moins restreinte du réseau. Ce n'est plus le cas. |

**Ce qui n'a pas changé :** la segmentation elle-même (6 zones, interfaces
dédiées, deny-by-default) était déjà en place en Phase A et faisait déjà
son travail — elle a bloqué le pivot dans 3 simulations sur 4. Le
durcissement ne touche pas à cette architecture de segmentation ; il
resserre uniquement ce que chaque zone est autorisée à faire à travers
les frontières déjà existantes.

---

## 0bis. Isolation stricte de l'attaquant

Objectif explicite de ce durcissement : l'attaquant ne doit avoir **aucun
moyen d'accès ou de danger au-delà de ce qui est strictement nécessaire
au scénario de test**, et aucune zone interne ne doit pouvoir lui
répondre au-delà de la connexion qu'il a lui-même initiée.

**Ce que l'attaquant peut atteindre — et rien d'autre :**

| Cible | Port | Raison |
|---|---|---|
| Hôte DMZ (DVWA) | `80, 443` | Cible de test volontaire, application web uniquement |
| Hôte Legacy (Metasploitable2) | `21, 6200` | Cible de test volontaire, ports réellement exploités dans la Simulation 2 |

Toute autre destination, tout autre port, est bloqué par défaut sur
l'interface WAN — l'attaquant n'a aucune route vers `userlan`,
`serverlan` ou `securitylan` directement.

**Ce qu'aucune zone interne ne peut renvoyer vers l'attaquant :**

Les deux règles qui autorisaient un retour de trafic illimité vers
`192.168.163.164` (depuis `serverlan` et depuis `dmz`) sont **retirées**.
Sans base légitime, un canal de sortie standing vers l'IP de l'attaquant
n'a pas de raison d'exister — s'il est nécessaire ponctuellement pour un
test manuel, il doit être ajouté temporairement puis retiré, pas laissé
en place en permanence.

**Deux points restants à vérifier avant de considérer l'isolement comme
réellement complet :**

1. ⚠️ **Risque résiduel assumé — pivot DMZ → Security_LAN.** Si
   l'attaquant compromet l'hôte DMZ via `80/443`, cet hôte conserve ses
   propres règles sortantes légitimes vers Security_LAN (`8081` MISP,
   `8000` Velociraptor — nécessaires pour que l'hôte DMZ reste supervisé).
   Un attaquant qui prend ce host hérite donc de cette portée réseau
   limitée mais réelle. Ce n'est pas éliminable sans supprimer la
   supervision de la DMZ elle-même — c'est documenté ici comme un risque
   résiduel **assumé**, pas comme un oubli (cohérent avec le principe de
   transparence de [`security-principles.md`](../docs/security-principles.md),
   section 6), plutôt que de prétendre que l'isolement est absolu alors
   qu'il ne l'est pas tout à fait.
2. ⚠️ **À vérifier : règle auto-générée sur `legacy`.** La baseline note
   "aucune règle manuelle" sur cette interface, mais ne confirme pas
   explicitement que la règle auto-générée **"Default allow LAN to
   any"** — celle-là même qui rendait `securitylan` dangereusement
   permissive avant durcissement — a bien été supprimée sur `legacy`
   également. Si elle est toujours active, un hôte Legacy compromis
   aurait un accès sortant total malgré l'absence de règle manuelle
   visible. **À confirmer et supprimer explicitement dans
   `Firewall → Rules → legacy` avant de considérer cette zone comme
   isolée.**

---

## 1. WAN

| Protocole | Source | Destination | Port |
|---|---|---|---|
| TCP | WAN net | dmz net | `80, 443` |
| TCP | WAN net | `192.168.11.197/32`* | `21, 6200` |

\* *Hôte Legacy (Metasploitable2) — ports limités à ceux réellement
exploités dans la Simulation 2 (`21` = déclenchement du backdoor vsftpd,
`6200` = shell backdoor résultant). À confirmer contre
[`simulation-2-metasploitable-pivot.md`](../reports/phase-a-baseline/simulation-2-metasploitable-pivot.md)
avant application.*

---

## 2. userlan (User_LAN) ↔ serverlan (Server_LAN)

La baseline contenait **deux niveaux de règles qui font la même chose** :
une règle vers l'hôte AD-DC nommément (`192.168.7.139/32`) et une règle
plus large vers `serverlan net` en entier. Tant que Server_LAN n'héberge
qu'un seul serveur (l'AD-DC), la règle "zone entière" n'ajoute aucune
capacité légitime — elle ne fait qu'élargir inutilement la portée si
d'autres machines venaient à rejoindre `serverlan` plus tard sans qu'on
y pense. Conservée uniquement la version nommée, la plus étroite :

| Protocole | Source | Destination | Port |
|---|---|---|---|
| TCP/UDP | userlan net | `192.168.7.139/32` | AD_Ports |
| ICMP | userlan net | `192.168.7.139/32` | * |
| TCP | userlan net | `192.168.9.133/32` | Velociraptor_Port |
| TCP/UDP | userlan net | securitylan net | `8081` |
| ICMP | userlan net | securitylan net | * |
| TCP/UDP | userlan net | `192.168.9.133` | Wazuh_Ports |
| ICMP | userlan net | userlan net | * |
| TCP | userlan net | WAN net | `80, 443` |
| TCP/UDP | userlan net | WAN net | `53` |

Retiré : `userlan net → serverlan net (AD_Ports)` et
`userlan net → serverlan net (ICMP)` — redondants avec les règles vers
`192.168.7.139/32` ci-dessus. **Si un second serveur est ajouté à
Server_LAN**, il doit recevoir sa propre règle nommée plutôt que de
rouvrir une confiance de zone à zone entière — c'est le principe même du
moindre privilège appliqué à la relation userlan ↔ serverlan.

Retiré : `userlan net → WAN net (tout protocole)` — remplacé par
web + DNS uniquement (voir Section 0).

**Aucune règle `serverlan → userlan` n'existait dans la baseline, et
aucune n'est ajoutée ici** — l'authentification AD est initiée par le
client, pas par le serveur ; il n'y a pas de besoin identifié pour que
Server_LAN initie une connexion vers User_LAN.

---

## 3. serverlan (Server_LAN)

| Protocole | Source | Destination | Port |
|---|---|---|---|
| TCP | serverlan net | `192.168.9.133/32` | Velociraptor_Port |
| ICMP | serverlan net | `192.168.7.139` | * |
| TCP/UDP | serverlan net | `192.168.7.139` | AD_Ports |
| TCP | serverlan net | securitylan net | `8081` |
| ICMP | serverlan net | securitylan net | * |
| TCP/UDP | serverlan net | `192.168.9.133` | Wazuh_Ports |

Retiré : `serverlan net → securitylan net (tout protocole)` — catch-all
redondant avec les règles nommées déjà présentes. Retiré :
`serverlan net → 192.168.163.164/24 (tout protocole)` — voir Section 0bis.

---

## 4. dmz (DMZ_LAN)

| Protocole | Source | Destination | Port |
|---|---|---|---|
| TCP | dmz net | `192.168.9.133/32` | Wazuh_Ports |
| ICMP | dmz net | `192.168.9.133/32` | * |
| TCP | dmz net | securitylan net | `8081` |
| TCP | dmz net | securitylan net | `8000` |
| ICMP | WAN net | dmz net | * |

Retiré : `dmz net → * (ICMP illimité)` — restreint à Security_LAN
uniquement. Retiré : `dmz net → 192.168.163.164 (tout protocole)` — voir
Section 0bis.

⚠️ Si `192.168.11.0/24` correspond en réalité à **Legacy** et non à
**DMZ** (incohérence non résolue entre les schémas Phase A et Phase B —
voir [`network-topology.md`](../docs/network-topology.md)), la règle WAN
correspondante en Section 1 doit être déplacée en conséquence.

---

## 5. legacy (Legacy_LAN)

Aucune règle sortante manuelle. Seule l'entrée WAN définie en Section 1
(`21, 6200` uniquement) atteint cette zone.

⚠️ **Voir Section 0bis, point 2** — la suppression explicite de la règle
auto-générée "Default allow LAN to any" sur cette interface doit être
confirmée manuellement dans OPNsense avant de considérer cette zone
comme réellement isolée.

---

## 6. securitylan (Security_LAN)

| Protocole | Source | Destination | Port |
|---|---|---|---|
| TCP | securitylan net | WAN net | `443` |
| TCP | securitylan net | WAN net | `587` |
| TCP/UDP | securitylan net | WAN net | `53` |
| TCP | `192.168.11.0/24` | `192.168.9.133/32` | Wazuh_Ports |
| TCP | dmz/userlan/serverlan net | `192.168.9.133/32` | Velociraptor_Port |
| TCP/UDP | serverlan/userlan net | `192.168.9.133/32` | Wazuh_Ports |

Retiré : **"Default allow LAN to any"** (IPv4 et IPv6). Remplacée par
exactement les trois flux sortants réellement utilisés :
- `443` — VirusTotal, flux MISP externes, appels HTTPS de Shuffle/TheHive
- `587` — notification email SOC (SMTP)
- `53` — résolution DNS nécessaire aux flux ci-dessus

Les échanges internes entre Wazuh, Shuffle, TheHive et MISP (tous sur
`192.168.9.0/24`) restent couverts par le trafic intra-zone.

---

## 7. Alias — définition stricte requise

| Alias | Ports |
|---|---|
| `AD_Ports` | TCP/UDP `53`, TCP/UDP `88`, TCP `135`, TCP `389`, TCP `636`, TCP `445`, TCP `3268`, TCP `3269` |
| `Wazuh_Ports` | TCP `1514`, TCP `1515` |
| `Velociraptor_Port` | TCP `8000` |

`AD_Ports` doit être vérifié contre la capture réelle de l'alias dans
OPNsense (`Firewall → Aliases`) avant application.

---

## 8. Résumé

| Interface | Règles retirées | Règles restreintes |
|---|---|---|
| WAN | 1 (`legacy` ouvert) | 1 (`dmz` limité à 80/443), 1 (`legacy` limité à 21/6200) |
| userlan ↔ serverlan | 2 (règles zone-large redondantes) | 1 (sortie WAN limitée à web+DNS) |
| serverlan | 2 | 0 |
| dmz | 1 | 1 (ICMP restreint) |
| legacy | — | 1 (à confirmer manuellement — règle auto-générée) |
| securitylan | 2 | 0 (remplacées par 3 règles nommées) |

Aucune zone ne conserve de règle "tout protocole" ou "tout port" vers une
destination large. Chaque flux restant est nommé, justifié, et limité au
port strictement nécessaire. Deux points restent explicitement ouverts
(Section 0bis) plutôt que présentés comme résolus : le risque résiduel de
pivot DMZ → Security_LAN, et la confirmation manuelle de la règle
auto-générée sur `legacy`.
