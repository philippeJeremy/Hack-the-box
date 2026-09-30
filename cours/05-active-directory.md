# Module 5 — Active Directory

> ⚠️ Exercices sur labs autorisés uniquement (art. 323-1). Voir [`index`](index.md).

## Objectif
Comprendre **comment un domaine Windows fonctionne et où il fait confiance à tort**, pour passer d'un compte lambda à la maîtrise du domaine. C'est le module le plus important pour ton poste.

## Pourquoi c'est clé pour le poste
90 % des SI d'entreprise reposent sur Active Directory. La question d'entretien classique — *« compromettre un domaine sans identifiants »* — se joue ici. Et tout se transpose au monde Linux (FreeIPA/IdM, cf. [`rhel-idm`](../outils/rhel-idm.md)) : même Kerberos, mêmes concepts.

## Concepts

**Ce qu'est AD.** Un annuaire centralisé : utilisateurs, machines, groupes, politiques, tous gérés par des **contrôleurs de domaine (DC)**. Se connecter quelque part = demander à l'annuaire « qui es-tu et as-tu le droit ? ». Cette centralisation est sa force *et* sa surface d'attaque : compromettre le DC = compromettre tout.

**Kerberos, le cœur.** Le protocole d'authentification. Modèle mental (billets de spectacle) :
- Tu prouves ton identité une fois → tu reçois un **TGT** (ticket d'entrée, « le bracelet »).
- Pour accéder à un service, tu présentes le TGT → tu reçois un **TGS** (ticket pour *ce* service, « le billet du concert »).
- Les tickets sont chiffrés avec des clés dérivées de mots de passe. **Toute la sécurité repose là-dessus** → d'où les attaques ci-dessous, qui visent à récupérer ou forger ces tickets.

**LDAP, l'annuaire interrogeable.** AD s'énumère massivement via LDAP : qui est admin, qui appartient à quel groupe, quelles machines, quelles délégations. Un simple compte utilisateur permet déjà d'en lire énormément — c'est pour ça que « accès faible ≠ accès inoffensif ».

**Les familles d'attaques (le *quoi* et le *pourquoi*, pas le *comment* armé) :**
- **AS-REP roasting** — certains comptes n'exigent pas la pré-authentification Kerberos ; on récupère alors un élément chiffré avec leur mot de passe, cassable hors-ligne. *Cause : une option de compte mal réglée.*
- **Kerberoasting** — n'importe quel utilisateur peut demander un TGS pour un compte de service ; ce ticket est chiffré avec le mot de passe du service → cassable hors-ligne. *Cause : des comptes de service à mots de passe faibles.*
- **Pass-the-Hash / Pass-the-Ticket** — en NTLM/Kerberos, on peut s'authentifier avec le *hash* ou le *ticket* sans connaître le mot de passe en clair. *Cause : la réutilisation de secrets et le SSO.*
- **DCSync** — un compte avec les bons droits peut *demander au DC de répliquer* les secrets du domaine, comme le ferait un autre DC → récupération des hashes de tous les comptes. *Cause : des droits de réplication accordés trop largement.*
- **Délégation (RBCD…)** — mécanisme légitime (un service agit au nom d'un utilisateur) détourné pour usurper une identité. *Cause : des délégations mal cadrées.*
- **Failles de configuration** : mots de passe en clair dans des attributs/partages (GPP, descriptions), groupes à privilèges oubliés (Backup Operators, DnsAdmins).

**Le graphe.** La clé mentale d'AD : ce n'est pas une liste, c'est un **graphe de relations**. « Qui peut agir sur qui ? » Un chemin part de ton compte faible et mène, de relation en relation, jusqu'à admin du domaine. **BloodHound** dessine ce graphe — c'est ta carte.

## Méthodologie
1. **Se repérer** : suis-je dans le domaine ? quel est le DC (ports 88/389/445) ?
2. **Énumérer largement** via LDAP (utilisateurs, groupes, délégations) — même avec un accès minimal.
3. **Cartographier** avec BloodHound : quels chemins vers un compte à privilèges ?
4. **Récupérer des secrets** (roasting, creds en clair) et les **casser** hors-ligne.
5. **Rejouer** les secrets (PtH/PtT) pour avancer sur le chemin.
6. **Atteindre** la maîtrise du domaine (DCSync), **documenter** chaque saut.

## Outils & rôle
- **bloodhound** ([fiche](../outils/bloodhound.md)) — cartographie du graphe, chemins d'attaque.
- **ad-attacks** ([fiche](../outils/ad-attacks.md)) — roasting, DCSync, délégations + remédiations.
- **impacket** ([fiche](../outils/impacket.md)) — exécution distante, secretsdump, Kerberos.
- **smb / NetExec** ([fiche](../outils/smb.md)) — énumération, spraying, PtH.
- **hashcat** ([fiche](../outils/hashcat.md)) — casser ce qu'on récupère (roasting).

## Pièges courants
- Sous-estimer un **compte faible** : il ouvre déjà toute l'énumération LDAP.
- Foncer sans **BloodHound** : sans la carte, on attaque à l'aveugle.
- Oublier que **casser un hash** peut prendre du temps / échouer : privilégier les chemins qui ne dépendent pas d'un mot de passe faible.

## Côté défense (Blue)
Chaque attaque a une trace (repris du tableau de ton [`index` outils](../outils/index.md)) :

| Attaque | Trace clé |
| --- | --- |
| Kerberoasting | Event 4769 (chiffrement RC4) |
| AS-REP roasting | Event 4768 sans pré-auth |
| Password spraying | 4625 en rafale |
| Pass-the-Hash | 4624 type 3 (NTLM) |
| DCSync | 4662 depuis un hôte non-DC |

Remédiations structurelles : mots de passe de service longs (ou gMSA), désactiver RC4, tiering administratif (comptes admin cloisonnés), moindre privilège sur les droits de réplication, surveiller les délégations.

## À l'oral
- *« Compromettre un domaine sans identifiants, comment tu abordes ça ? »* → énumération de ce qui est accessible sans auth (AS-REP roasting sur comptes sans pré-auth, spraying prudent), puis cartographie BloodHound dès le premier accès.
- *« Explique Kerberoasting simplement. »* → tout utilisateur peut demander un ticket pour un compte de service ; ce ticket est chiffré avec le mot de passe du service, donc cassable hors-ligne. Correctif : mots de passe de service forts / gMSA.
- *« C'est quoi DCSync et pourquoi c'est grave ? »* → se faire passer pour un DC et demander la réplication des secrets → on obtient les hashes de tout le domaine. Correctif : restreindre les droits de réplication.
- *« Pourquoi BloodHound ? »* → AD est un graphe de relations ; BloodHound révèle le chemin le plus court d'un compte faible vers admin, qu'on ne verrait pas à l'œil.

## Pratiquer
C'est ta **phase 4** : Forest → Sauna → Active → Support, puis **GOAD** (lab local multi-machines) comme livrable. Sur chaque box, remplis le tableau attaque → trace → détection : c'est ce qui fait la valeur Red + Blue de ton profil.
