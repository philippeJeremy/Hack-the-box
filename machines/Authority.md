# Authority — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows (contrôleur de domaine) |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-__-__ |
| **Vecteur** | <à compléter moi-même> |
| **CVE** | — |
| **Tags** | active-directory · ansible-vault · pwm · adcs · esc1 |

> 🎯 **Nouveautés à apprendre (je cherche les commandes seul)** :
> 1) Un partage expose des fichiers **Ansible** : un secret est protégé par **Ansible Vault** → à casser (`ansible2john`) puis déchiffrer.
> 2) Ces creds déverrouillent **PWM** (appli de gestion de mdp) → récupérer un compte du domaine (pense capture LDAP / config en mode debug).
> 3) **ADCS ESC1** à nouveau, mais avec une subtilité : le compte utilisé pour demander le certif (souvent un **compte machine**). Réutilise ce que tu as appris sur Escape.

> 💡 Concepts à googler : « Ansible Vault crack », « PWM LDAP config capture », « Certipy ESC1 machine account ».

---

## TL;DR
<à remplir en fin de box>

## 1. Reconnaissance
<ports — PWM (8443) ? partages lisibles en anonyme ?>

## 2. Foothold
<où sont les fichiers Ansible ? comment casser le Vault ? que donnent ces creds dans PWM ?>

## 3. Privesc — ADCS ESC1
<quel template vulnérable ? quel compte pour demander le certif ? auth → hash admin>

## 4. Remédiation
<ne pas versionner de Vault faible ; durcir les templates ADCS>

## 5. Leçons
- <Ansible Vault : secret chiffré crackable hors ligne>
- <PWM : une appli de gestion de mdp mal configurée fuit des creds>
- <ESC1 confirmé — la 2e fois pour ancrer le réflexe ADCS>

## Références
- Fiches liées : [ad-memo](../outils/ad-memo.md) · [bloodhound](../outils/bloodhound.md) · [hashcat](../outils/hashcat.md)
