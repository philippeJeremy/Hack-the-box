# RHEL / FreeIPA-IdM — cheatsheet

Linux entreprise Red Hat : **identité centralisée (FreeIPA/IdM)**, SELinux, sudo/HBAC centralisé, surfaces d'admin. Prolonge [`linux-privesc`](linux-privesc.md) et [`ad-attacks`](ad-attacks.md) — même logique Kerberos que l'AD, côté Linux.

> ⚠️ **Rappel légal.** Uniquement sur ton **lab FreeIPA auto-hébergé** (Rocky/Alma) ou plateformes autorisées. Hors cadre contractuel écrit → infraction, **art. 323-1**.

**Plan** : commandes → recettes → côté défense → à savoir expliquer à l'oral.

---

## 1. Contexte — pourquoi c'est le « GOAD Linux »

FreeIPA/IdM = **Kerberos + LDAP (389-DS) + DNS + CA (Dogtag)**, exactement les briques d'un AD, côté Linux. Les postes/serveurs sont enrôlés via **SSSD**. Donc tout ce que tu as appris sur Forest (tickets, énum d'annuaire, rôles à privilèges) se transpose : le vocabulaire change, pas les concepts.

| AD | Équivalent IPA |
| --- | --- |
| Domain Controller | serveur IPA |
| Kerberos KDC | KDC intégré |
| LDAP (AD) | 389 Directory Server |
| GPO / délégations | HBAC + rôles RBAC IPA |
| `admin` du domaine | rôle `admin` / groupe `admins` |

---

## 2. Commandes — énumération (accès authentifié labo)

```bash
# Suis-je sur une machine enrôlée IPA ?
realm list ; cat /etc/sssd/sssd.conf 2>/dev/null
klist ; klist -k                      # tickets / keytabs présents

# Kerberos
kinit <user>                          # obtenir un TGT
klist                                 # vérifier le ticket

# Énumération annuaire (client ipa)
ipa user-find
ipa group-find ; ipa group-show admins
ipa hbacrule-find                     # règles d'accès (host-based)
ipa sudorule-find                     # règles sudo centralisées
ipa role-find                         # rôles RBAC IPA

# LDAP direct (si autorisé sur le lab)
ldapsearch -Y GSSAPI -b "dc=lab,dc=local" "(objectclass=person)"
```

Contexte système (à croiser avec `linux-privesc`) :
```bash
sudo -l                               # ce que sssd/IPA m'autorise
getcert list                          # certificats gérés (certmonger)
getent passwd | tail                  # comptes résolus via SSSD
```

---

## 3. Recettes (lab)

**a. Réutilisation de ticket / keytab.** Un keytab lisible (`/etc/krb5.keytab`, keytab de service mal protégé) → `kinit -k` pour se faire passer pour le principal. → à documenter : quel principal, quel accès gagné.

**b. Règle sudo/HBAC trop large.** `ipa sudorule-find` révèle une règle autorisant une commande détournable (voir [`gtfobins`](gtfobins.md)) sur un hôte → escalade locale. La logique est celle de `sudo -l`, mais **définie centralement**.

**c. Rôle IPA à privilèges.** Un compte membre d'un rôle permettant de modifier des règles sudo/HBAC = équivalent d'un chemin BloodHound → il peut s'auto-octroyer un accès. → cartographier les rôles comme un graphe.

**d. SELinux permissif.** `getenforce` = `Permissive` ou domaines `unconfined` → une exploitation qui serait bloquée passe. À noter côté remédiation.

> *À compléter avec tes captures de lab FreeIPA (énum → chaîne → preuve).*

---

## 4. Côté défense / détection

| Action | Trace / contrôle |
| --- | --- |
| Demande de TGT anormale | logs KDC (`/var/log/krb5kdc.log`) |
| Modif de règle sudo/HBAC | audit IPA (`ipa` audit / 389-DS access log) |
| Keytab lu par un compte inattendu | auditd (`auditctl -w /etc/krb5.keytab -p r`) |
| SELinux | **jamais `Permissive` en prod** ; alertes `ausearch -m avc` |
| sudo | `journalctl _COMM=sudo`, centraliser vers SIEM |

Durcissement : SELinux `Enforcing`, keytabs à 600/root, rôles IPA au moindre privilège, séparation admin annuaire / admin système.

---

## 5. À savoir expliquer à l'oral

- « FreeIPA, c'est l'AD du monde Linux : Kerberos + LDAP + CA. Compromettre le rôle `admins` = compromettre le domaine. »
- Différence **HBAC vs sudorule** (qui peut se connecter où / qui peut exécuter quoi).
- Pourquoi **SELinux en Enforcing** change la donne pour une privesc.
- Le lien avec l'AD : dans un SI mixte, un lien de confiance IPA↔AD est un chemin d'attaque à part entière.
