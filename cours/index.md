# Cours — révision pentest (lecture mobile)

Cours de fond à lire pendant les temps libres, pensé pour le **téléphone** : concepts, schémas mentaux et méthodo — le *pourquoi*. Le *comment* opérationnel (commandes) reste dans les fiches [`outils/`](../outils/). Chaque module se lit en une ou deux sessions.

> ⚠️ **Rappel légal.** Ces connaissances s'exercent **uniquement** sur tes labs et plateformes autorisées (HackTheBox, TryHackMe, Root-Me, PortSwigger, labs auto-hébergés). Hors cadre contractuel écrit, un test d'intrusion est une infraction — **art. 323-1 du Code pénal**. Un pentester se distingue d'un attaquant par **l'autorisation, le périmètre et la traçabilité**, jamais par la technique.

---

## Comment lire ce cours

Chaque module suit le même plan, pour ancrer les réflexes :

1. **Objectif** — ce que tu dois savoir faire à la fin
2. **Pourquoi c'est clé pour le poste** — le lien métier
3. **Concepts** — le fond, en morceaux digestes
4. **Méthodologie** — le déroulé mental, dans l'ordre
5. **Outils & rôle** — ce que fait chaque outil (renvoi aux fiches)
6. **Pièges courants** — les erreurs qui coûtent cher
7. **Côté défense (Blue)** — comment ça se détecte / se corrige
8. **À l'oral** — les questions d'entretien et les bonnes réponses
9. **Pratiquer** — les machines/labs pour valider

Le fil rouge de tout le cursus : **entrer → comprendre → élever ses droits → se propager → prouver → détecter.** Chaque module est une étape de cette chaîne.

---

## Les modules

| # | Module | Étape de la chaîne | Fiches liées | Phase parcours |
| --- | --- | --- | --- | --- |
| 1 | [Fondamentaux réseau](01-fondamentaux-reseau.md) | socle | nmap | 1 |
| 2 | [Reconnaissance](02-reconnaissance.md) | entrer | nmap, smb, gobuster | 1-2 |
| 3 | [Sécurité web](03-web.md) | entrer | burp | 3 |
| 4 | [Exploitation](04-exploitation.md) | entrer | searchsploit, metasploit, reverse-shells | 4 |
| 5 | [Active Directory](05-active-directory.md) | comprendre / se propager | bloodhound, ad-attacks, impacket, hashcat | 4 |
| 6 | [Élévation de privilèges](06-privesc.md) | élever | linux-privesc, windows-privesc, gtfobins, lire-peas | 6 |
| 7 | [Post-exploitation](07-post-exploitation.md) | se propager | impacket, file-transfer, reverse-shells | — |
| 8 | [Reporting](08-reporting.md) | prouver | — | 8 |
| 9 | [Blue Team / détection](09-blue-team.md) | détecter | wazuh-sysmon | 9 |
| 10 | [Systèmes industriels / OT](10-ot-ics.md) | contexte indus | (ot-ics à venir) | 10 |

**Bloc entreprise / cloud** (parcours 2 — à lire après le module 5, car ils réutilisent Kerberos/l'annuaire) :

| # | Module | Étape de la chaîne | Fiches liées | Phase parcours |
| --- | --- | --- | --- | --- |
| 11 | [Linux entreprise / RHEL & FreeIPA](11-rhel-idm.md) | comprendre / élever | rhel-idm | 7 |
| 12 | [Conteneurs & Kubernetes](12-kubernetes.md) | entrer / se propager | kubernetes | 8 |
| 13 | [Citrix / accès distant](13-citrix.md) | entrer | citrix, windows-privesc | 9 |

---

## L'état d'esprit (à lire en premier)

- **Énumérer avant d'exploiter.** 80 % du travail est de la compréhension. On ne lance pas un exploit « pour voir » : on sait *pourquoi* on le lance.
- **Comprendre > mémoriser.** Un outil change, un concept reste. Sache expliquer *pourquoi* une attaque marche, tu sauras l'adapter.
- **Red + Blue.** Pour chaque attaque apprise, connais sa trace et sa remédiation. C'est ce qui fait un bon pentester (et un bon responsable pentest) : tu ne casses pas pour casser, tu aides à corriger.
- **Méthode = reproductibilité.** Un test qu'on ne peut pas rejouer et documenter n'a pas de valeur en mission.
