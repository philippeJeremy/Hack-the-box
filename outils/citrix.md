# Citrix / accès distant — cheatsheet

Gateway et VDI d'entreprise : **NetScaler/ADC**, Virtual Apps & Desktops, StoreFront. Deux angles : la **gateway** (surface externe, cas historiques) et le **breakout VDI** (échappement d'une session/app publiée verrouillée). Prolonge [`windows-privesc`](windows-privesc.md).

> ⚠️ **Rappel légal.** Uniquement sur ton **lab auto-hébergé** (versions d'évaluation ADC VPX + VDA dans tes VM). Pas de CTF gratuit clé en main — donc pas de test « sauvage ». Hors cadre contractuel écrit → infraction, **art. 323-1**. Les cas historiques ci-dessous sont étudiés **en compréhension**, pas déroulés sur une cible.

**Plan** : composants → recon → cas d'école → breakout VDI → côté défense → à l'oral.

---

## 1. Composants

| Élément | Rôle | Intérêt pentest |
| --- | --- | --- |
| **NetScaler / ADC** | reverse-proxy / gateway (Citrix Gateway) | surface externe n°1 |
| **StoreFront** | portail des apps/desktops publiés | énum, auth |
| **Virtual Apps & Desktops** (ex-XenApp/XenDesktop) | apps/bureaux publiés | breakout, latéral |
| **VDA** | l'agent sur la machine qui héberge la session | cible du breakout |

---

## 2. Recon (lab)

```bash
# Fingerprint version de la gateway (lab)
nmap -p 443,80 --script http-title,ssl-cert <ip_lab>
curl -sk https://<ip_lab>/vpn/index.html -I      # bannières / chemins Citrix
```

L'objectif en recon : **identifier la version** de l'ADC (les versions déterminent l'exposition connue) et cartographier StoreFront (points d'auth, apps publiées visibles).

---

## 3. Cas d'école (compréhension, pas exploitation)

À étudier pour savoir *pourquoi* ça a cassé — utile en entretien et pour la remédiation :

| Cas | Classe de faille | Leçon |
| --- | --- | --- |
| **« Shitrix »** (2019) | traversée de chemin → exécution | validation d'entrée insuffisante sur la gateway |
| **CitrixBleed** (2023) | fuite mémoire → vol de jetons de session | un jeton volé = contournement d'auth/MFA |

Ce qu'on retient : sur une gateway, les failles graves touchent le **parsing**, la **gestion de session** et les **jetons**. Côté défenseur : patcher vite + **invalider les sessions** après patch (un jeton volé survit au correctif sinon).

---

## 4. Breakout VDI / application publiée (lab)

Quand seule une app est publiée (pas un bureau complet), l'enjeu est de **sortir du bac à sable** :
- boîtes de dialogue (Ouvrir/Enregistrer) → navigation vers un explorateur ou un shell
- raccourcis clavier / menus d'aide lançant un processus tiers
- **mapping de lecteurs** client→session (exfiltration / dépôt d'outil)
- depuis la session, retomber sur du **Windows privesc classique** (voir la fiche) et du latéral vers l'annuaire

C'est très pertinent sur **poste industriel durci** : le kiosque est censé être verrouillé, le test valide qu'il l'est vraiment.

> *À compléter avec tes captures de lab (config publiée → technique de breakout → preuve → correctif).*

---

## 5. Côté défense / détection

| Vecteur | Contrôle / trace |
| --- | --- |
| Gateway non patchée | veille CVE + patch rapide + **invalidation des sessions** post-patch |
| Jeton de session volé | durée de vie courte, liaison IP, MFA robuste |
| Breakout d'app publiée | verrouillage kiosque (pas d'accès explorateur/dialogues), AppLocker/WDAC |
| Mapping de lecteurs | désactiver le mapping client si non nécessaire |
| Latéral post-session | segmentation, comptes de session au moindre privilège |

---

## 6. À savoir expliquer à l'oral

- « Sur Citrix, deux mondes : la **gateway** (surface externe, patch + gestion de session) et le **VDI** (est-ce que le kiosque tient vraiment ?). »
- Pourquoi **patcher ne suffit pas** après un vol de jeton : il faut invalider les sessions.
- Un **breakout d'app publiée** ramène à du Windows privesc + latéral classique : Citrix n'est qu'une porte.
- Réflexe : verrouillage applicatif (AppLocker/WDAC) + moindre privilège sur les comptes de session.
