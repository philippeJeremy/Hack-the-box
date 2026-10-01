# Module 11 — Linux entreprise / Red Hat & FreeIPA

> ⚠️ Exercices sur ton lab FreeIPA auto-hébergé uniquement (art. 323-1). Fiche opérationnelle : [`rhel-idm`](../outils/rhel-idm.md).

## Objectif
Comprendre l'**identité centralisée côté Linux** (FreeIPA/IdM) et le durcissement Red Hat (SELinux), pour transposer tout ton savoir AD (module 5) au monde Linux d'entreprise.

## Pourquoi c'est clé pour le poste
En environnement industriel/défense, l'IT n'est pas que du Windows : les serveurs critiques tournent souvent sous RHEL, avec une identité centralisée. Savoir attaquer *et* durcir un IdM Linux est rare et différenciant. Bonne nouvelle : c'est le **même Kerberos** que l'AD.

## Concepts

**FreeIPA = l'AD du monde Linux.** Mêmes briques : **Kerberos + LDAP + DNS + autorité de certification**, un serveur IPA qui joue le rôle du contrôleur de domaine, des postes enrôlés via **SSSD**. Tout ce que tu as compris au module 5 se transpose — seul le vocabulaire change.

| AD (module 5) | Équivalent FreeIPA |
| --- | --- |
| Contrôleur de domaine | serveur IPA |
| Kerberos KDC | KDC intégré |
| LDAP (AD) | 389 Directory Server |
| GPO / délégations | **HBAC** + rôles RBAC IPA |
| Admin du domaine | groupe `admins` / rôle `admin` |

**HBAC vs sudorules.** Deux notions centrales à distinguer :
- **HBAC** (Host-Based Access Control) : *qui* peut se connecter *où*.
- **sudorules** : *qui* peut exécuter *quoi* en privilégié, défini **centralement**. C'est le `sudo -l` du module 6, mais géré depuis l'annuaire → une règle trop large s'applique à toute une flotte.

**Les mêmes attaques Kerberos.** Un compte sans pré-authentification → AS-REP ; un keytab mal protégé → usurpation du principal ; un rôle IPA permettant de modifier les règles → chemin d'escalade façon BloodHound. La logique du graphe (module 5) s'applique.

**SELinux : la spécificité Red Hat.** Une couche de contrôle d'accès obligatoire qui *confine* les processus. En mode **Enforcing**, il bloque des exploitations qui marcheraient ailleurs. Deux réflexes : en attaque, repérer un SELinux `Permissive` (vecteur ouvert) ; en défense, le garder `Enforcing` (contre-mesure structurelle).

**Le lien IT mixte.** Dans un SI réel, un **lien de confiance IPA ↔ AD** est fréquent — et devient un chemin d'attaque inter-mondes. C'est là que ta maîtrise des deux annuaires prend toute sa valeur.

## Méthodologie
1. **Se repérer** : machine enrôlée IPA ? (SSSD, tickets présents)
2. **Énumérer l'annuaire** : utilisateurs, groupes, HBAC, sudorules, rôles.
3. **Cartographier** les rôles comme un graphe (qui peut modifier quoi).
4. **Récupérer des secrets** (keytabs, creds) et rejouer façon Kerberos.
5. **Vérifier SELinux** et la posture système (croiser avec le module 6).

## Outils & rôle
- **rhel-idm** ([fiche](../outils/rhel-idm.md)) — commandes d'énum IPA, Kerberos Linux, SELinux.
- **impacket / ad-attacks** ([fiches](../outils/)) — les concepts Kerberos se rejouent côté Linux.

## Pièges courants
- Croire que « Linux = pas d'annuaire » : un IdM Linux est une cible aussi riche qu'un AD.
- Ignorer SELinux : il peut expliquer pourquoi une exploitation échoue (ou réussit).
- Oublier le lien IPA↔AD dans un SI mixte.

## Côté défense (Blue)
- Keytabs en 600/root, SELinux **Enforcing**, pas de domaine `unconfined`.
- Rôles IPA et sudorules au **moindre privilège** ; séparer admin annuaire / admin système.
- Journaliser le KDC et les modifications HBAC/sudo vers le SIEM (module 9).

## À l'oral
- *« FreeIPA, c'est quoi par rapport à l'AD ? »* → l'équivalent Linux : Kerberos + LDAP + CA. Compromettre le groupe `admins` = compromettre le domaine.
- *« HBAC vs sudorule ? »* → l'un gère qui se connecte où, l'autre qui exécute quoi en privilégié, centralement.
- *« Qu'apporte SELinux en Enforcing ? »* → il confine les processus et bloque des exploitations qui passeraient sinon.

## Pratiquer
C'est ta **phase 7** (parcours 2) : monte une VM **Rocky/Alma + FreeIPA** — le « GOAD Linux » — et déroule énum → Kerberos → escalade de rôle. Root-Me pour les épreuves Linux complémentaires.
