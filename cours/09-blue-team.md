# Module 9 — Blue Team / détection

> ⚠️ Voir [`index`](index.md).

## Objectif
Savoir **détecter** les attaques des modules précédents : quelle trace laisse chaque technique, où la chercher, comment corréler. C'est la moitié « défense » d'un profil Purple.

## Pourquoi c'est clé pour le poste
Un pentester qui connaît la détection produit de bien meilleures remédiations et sait dire au client *« voilà ce que vous auriez dû voir »*. Pour un poste en environnement industriel/défense, la culture détection/SOC est souvent aussi valorisée que l'offensif. C'est ton différenciateur.

## Concepts

**Le renversement de perspective.** Jusqu'ici tu apprenais à *faire* ; ici tu apprends à *voir*. Chaque action offensive produit un signal quelque part (journal système, flux réseau, comportement de processus). Le Blue, c'est transformer ce signal en alerte.

**Les sources de traces :**
- **Journaux système** (Windows Event Log, auditd/syslog Linux) : authentifications, créations de services, exécutions.
- **Télémétrie de processus** (**Sysmon** sous Windows) : qui a lancé quoi, quel processus parent, quelles connexions. Beaucoup plus riche que les logs par défaut.
- **Réseau** (IDS type Suricata/Zeek) : scans, connexions sortantes anormales, exfiltration.
- **EDR** : corrélation comportementale sur les postes.

**La centralisation : le SIEM.** Les traces éparpillées ne servent à rien. Un **SIEM** (ex. **Wazuh**) les collecte, les normalise, et permet d'écrire des **règles de détection** et de corréler des événements de sources différentes. C'est le cœur d'un SOC.

**MITRE ATT&CK : le langage commun.** Un référentiel qui nomme chaque technique d'attaque (ex. T1059 = exécution par interpréteur). Il sert à **cartographier** ce qu'on sait détecter et à repérer les angles morts. Parler ATT&CK, c'est parler le langage du SOC.

**La boucle Purple Team.** Le plus formateur : tu **attaques** ton propre lab, puis tu vérifies **ce que la détection a vu** (ou raté), et tu améliores les règles. Attaque → trace → détection → ajustement. C'est exactement ce que fait ta fiche `wazuh-sysmon`.

**Le tableau de correspondance (ton livrable module 9) :**

| Attaque | Module | Trace clé |
| --- | --- | --- |
| Scan de ports | 1-2 | rafale de connexions (IDS) |
| Kerberoasting | 5 | Event 4769 (RC4) |
| AS-REP roasting | 5 | Event 4768 sans pré-auth |
| Password spraying | 5 | 4625 en rafale |
| Pass-the-Hash | 5-7 | 4624 type 3 (NTLM) |
| DCSync | 5 | 4662 depuis un non-DC |
| Exécution distante (psexec) | 7 | Event 7045 (création de service) |
| Reverse shell | 4 | interpréteur → connexion sortante (T1059) |

## Méthodologie (Purple)
1. **Monter le lab de détection** : Sysmon sur les postes, collecte vers un SIEM (Wazuh).
2. **Rejouer une attaque** apprise (roasting, PtH…).
3. **Chercher la trace** correspondante dans le SIEM.
4. **Écrire/affiner** la règle de détection.
5. **Mapper** sur MITRE ATT&CK pour visualiser la couverture.

## Outils & rôle
- **wazuh-sysmon** ([fiche](../outils/wazuh-sysmon.md)) — le lab SIEM maison : détecter ses propres attaques.
- **MITRE ATT&CK Navigator** — cartographier la couverture de détection.

## Pièges courants
- Croire qu'une attaque est « invisible » : presque tout laisse une trace, la question est *si quelqu'un regarde*.
- Générer des règles trop bruyantes (faux positifs) → alertes ignorées. La qualité prime sur la quantité.
- Oublier les **angles morts** : ce que le lab ne journalise pas ne sera jamais détecté.

## À l'oral
- *« Comment on détecte un Kerberoasting ? »* → Event 4769 avec chiffrement RC4 en volume anormal depuis un compte ; corrélé dans le SIEM.
- *« C'est quoi MITRE ATT&CK ? »* → un référentiel qui nomme les techniques d'attaque ; il sert à cartographier ce qu'on détecte et à trouver les angles morts.
- *« Pourquoi Sysmon plutôt que les logs par défaut ? »* → il donne la généalogie des processus et les connexions, là où les logs natifs sont trop pauvres.
- *« C'est quoi une démarche Purple Team ? »* → attaquer pour valider la détection et l'améliorer, plutôt qu'opposer Red et Blue.

## Pratiquer
Ton lab **Wazuh + Sysmon** : pour chaque attaque de tes machines, va vérifier la trace et écris la règle. C'est le livrable qui matérialise ton profil Red **+** Blue.
