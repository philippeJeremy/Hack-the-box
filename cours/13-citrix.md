# Module 13 — Citrix / accès distant

> ⚠️ Exercices sur ton lab auto-hébergé (versions d'évaluation) uniquement. Les cas historiques s'étudient **en compréhension**, pas sur une cible (art. 323-1). Fiche : [`citrix`](../outils/citrix.md).

## Objectif
Comprendre comment fonctionne un accès distant d'entreprise (gateway + bureaux/applications publiés) et où il cède : la **gateway** exposée et le **breakout** d'une session verrouillée.

## Pourquoi c'est clé pour le poste
Citrix est omniprésent pour donner accès à distance à des applications et bureaux, y compris sur postes durcis en environnement industriel. C'est souvent le point d'entrée externe *ou* le poste verrouillé qu'on te demandera de tester.

## Concepts

**Les composants.**
- **NetScaler / ADC** : la gateway (reverse-proxy d'accès). Surface externe la plus exposée.
- **StoreFront** : le portail qui liste les applications/bureaux publiés.
- **Virtual Apps & Desktops** : les applications et bureaux eux-mêmes.
- **VDA** : l'agent sur la machine qui héberge la session utilisateur.

**Deux angles d'attaque distincts :**

**1. La gateway (surface externe).** Historiquement la cible des vulnérabilités graves. À étudier **comme cas d'école** pour comprendre *la classe de faille*, pas pour rejouer :
- Les grandes failles Citrix ont touché le **parsing** des requêtes, la **gestion de session** et les **jetons**.
- Leçon défensive majeure : après un correctif, il faut **invalider les sessions** existantes — un jeton volé avant le patch survit sinon. Patcher ne suffit pas.

**2. Le breakout VDI (poste verrouillé).** Quand seule une application est publiée (pas un bureau complet), l'enjeu est de **sortir du bac à sable** :
- détourner une boîte de dialogue (Ouvrir/Enregistrer) vers un explorateur ou un shell,
- abuser de raccourcis/menus lançant un processus tiers,
- exploiter le **mapping de lecteurs** client↔session pour déposer un outil ou exfiltrer.
Une fois « sorti », on retombe sur du **Windows privesc** (module 6) et du **latéral** (module 7) classiques. C'est particulièrement pertinent sur poste industriel durci : le test valide que le kiosque tient *vraiment*.

**Citrix n'est qu'une porte.** Comme le web ou K8s, l'accès distant est un **tremplin** vers le SI (l'annuaire, les applications internes), pas une fin en soi.

## Méthodologie
1. **Recon** : identifier la version de la gateway, cartographier StoreFront.
2. **Gateway** : évaluer l'exposition (versions, configuration) — en compréhension.
3. **Session publiée** : tester le confinement (peut-on sortir de l'app ?).
4. **Post-breakout** : privesc + latéral comme sur un poste Windows normal.
5. **Recommander** : patch + invalidation de sessions, verrouillage applicatif.

## Outils & rôle
- **citrix** ([fiche](../outils/citrix.md)) — recon, cas d'école, techniques de breakout.
- **windows-privesc** ([fiche](../outils/windows-privesc.md)) — la suite une fois sorti de la session.

## Pièges courants
- Rejouer un exploit de gateway sur autre chose que ton lab : hors sujet et illégal.
- Croire qu'un **patch** suffit après un vol de jeton : il faut invalider les sessions.
- Oublier que le breakout n'est que le début : la vraie valeur est dans le latéral derrière.

## Côté défense (Blue)
- **Patch rapide** de la gateway **+ invalidation des sessions** post-correctif.
- **Jetons** à durée de vie courte, liaison à l'IP, MFA robuste.
- **Verrouillage applicatif** (AppLocker/WDAC) sur les postes/kiosques, mapping de lecteurs désactivé si inutile.
- **Segmentation** : un compte de session au moindre privilège, pour limiter le latéral.

## À l'oral
- *« Deux mondes Citrix à tester ? »* → la gateway (surface externe : patch + gestion de session) et le VDI (le kiosque tient-il vraiment ?).
- *« Pourquoi patcher ne suffit pas après un vol de jeton ? »* → le jeton volé reste valide ; il faut invalider les sessions.
- *« Un breakout d'app publiée, et après ? »* → on retombe sur du Windows privesc + latéral ; Citrix n'est qu'une porte.

## Pratiquer
C'est ta **phase 9** : monte un lab avec les versions d'évaluation (ADC VPX + un VDA) et teste surtout le **confinement** et le **durcissement**. Pas de CTF gratuit clé en main — d'où le lab perso.
