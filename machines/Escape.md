# Escape — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows (contrôleur de domaine) |
| **Difficulté** | 🟡 Medium |
| **Date** | 2026-__-__ |
| **Vecteur** | <à compléter moi-même> |
| **CVE** | — (abus de config ADCS) |
| **Tags** | active-directory · mssql · adcs · esc1 · certipy |

> 🎯 **Nouveautés à apprendre — ma 1re box ADCS (je cherche les commandes seul)** :
> 1) **MSSQL** accessible : récupérer un hash d'authentification (pense coercition / `xp_` ) et le casser, ou lire des logs qui fuient des creds.
> 2) **ADCS — ESC1** : un **template de certificat** mal configuré autorise à demander un certif **au nom de n'importe qui** (Administrator). Outil clé : **Certipy** (`find` pour repérer le template vulnérable, `req` pour demander le certif).
> 3) Un certificat d'un compte = **authentification Kerberos** → récupérer son hash (PKINIT / UnPAC-the-hash) → PtH.

> 💡 Installe **certipy-ad** (`pipx install certipy-ad`). Concepts à googler : « ADCS ESC1 », « Certipy find / req / auth ».

---

## TL;DR
<à remplir en fin de box>

## 1. Reconnaissance
<ports — MSSQL (1433) ? partages lisibles ?>

## 2. Foothold
<comment obtenir un 1er compte via MSSQL ? où fuit le mot de passe ?>

## 3. Privesc — ADCS ESC1
<Certipy find → quel template est vulnérable ? Certipy req -upn administrator → certif → auth → hash>

## 4. Remédiation
<durcir les templates de certificats : enrôlement, EKU, "enrollee supplies subject">

## 5. Leçons
- <ADCS = nouvelle surface énorme ; ESC1 = template qui laisse choisir le sujet (SAN)>
- <un certificat vaut une authentification → PKINIT → hash NT>

## Références
- Fiches liées : [ad-memo](../outils/ad-memo.md) · [bloodhound](../outils/bloodhound.md) · [hashcat](../outils/hashcat.md)
