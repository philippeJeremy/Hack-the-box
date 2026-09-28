# Lab Blue Team — Wazuh + Sysmon

Monter un mini-SOC maison : un SIEM (Wazuh), un poste Windows instrumenté (Sysmon + audit), et Kali comme attaquant. Le but n'est pas d'administrer Wazuh, c'est de **voir quelles traces tes propres attaques laissent** et d'écrire les règles qui les détectent. C'est le travail pratique du module 9 (Purple Team) et le livrable « tableau attaque → détection » de ta checklist de candidature.

> ⚠️ **Cadre.** Tout se passe dans TON lab, sur TES VM. On rejoue ici les attaques déjà vues (modules 5–7) uniquement pour les détecter.

---

## 1. Architecture du lab

```
        ┌──────────────┐         logs (chiffrés 1514/udp)
        │  Kali        │ ───────────────────────────────┐
        │  (attaquant) │                                 │
        └──────┬───────┘                                 ▼
               │ attaque (SMB, WinRM…)         ┌───────────────────┐
               ▼                               │  Wazuh server     │
        ┌──────────────┐   agent Wazuh         │  (Ubuntu)         │
        │  Windows 10  │ ─────────────────────►│  SIEM + dashboard │
        │  + Sysmon    │                       │  https://<IP>     │
        └──────────────┘                       └───────────────────┘
```

Trois VM sur le même réseau privé (host-only ou NAT interne VirtualBox) :

| VM | Rôle | Ressources mini | IP exemple |
| --- | --- | --- | --- |
| **Ubuntu Server 22.04** | Wazuh (SIEM + dashboard) | 4 vCPU, 8 Go RAM, 50 Go | `192.168.56.10` |
| **Windows 10/11** | Cible instrumentée (agent + Sysmon) | 2 vCPU, 4 Go RAM | `192.168.56.20` |
| **Kali** | Attaquant | 2 vCPU, 4 Go RAM | `192.168.56.30` |

Wazuh est gourmand : les 8 Go pour l'Ubuntu ne sont pas négociables (il embarque un Indexer type Elasticsearch). Si ton hôte a 16 Go, allume l'Ubuntu + une seule autre VM à la fois.

> Tu peux réutiliser un des Windows clients de ton lab AD (module 5). Pas besoin du DC pour commencer.

---

## 2. Installer le serveur Wazuh (VM Ubuntu)

Wazuh 4.14, méthode « quickstart » (tout-en-un : Indexer + Server + Dashboard) :

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

L'installation prend 10–15 min. À la fin, elle affiche l'utilisateur `admin` et son mot de passe. Pour le retrouver :

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Accès au tableau de bord depuis Kali ou l'hôte : `https://192.168.56.10`
(certificat auto-signé → avertissement navigateur normal, on passe outre.)

**Vérifier** que le service tourne :
```bash
sudo systemctl status wazuh-manager
```

---

## 3. Déployer l'agent sur Windows

Dans le dashboard : **Agents → Deploy new agent** génère la commande exacte, avec l'IP du manager déjà remplie. Elle ressemble à (PowerShell **en administrateur**) :

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.8-1.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER="192.168.56.10"
Start-Service WazuhSvc
```

Au bout d'une minute, l'agent apparaît **Active** dans **Agents** sur le dashboard.

---

## 4. Installer et configurer Sysmon

Sysmon (Sysinternals) est ce qui donne la télémétrie fine : création de processus avec ligne de commande complète, accès inter-processus (utile pour LSASS), connexions réseau, chargement de DLL. Les journaux Windows par défaut ne descendent pas à ce niveau.

On utilise la config de référence **SwiftOnSecurity** (bon rapport signal/bruit) :

```powershell
# Dans un dossier de travail, en administrateur
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile Sysmon.zip
Expand-Archive Sysmon.zip
Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile sysmonconfig.xml
.\Sysmon\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Sysmon écrit dans le journal `Microsoft-Windows-Sysmon/Operational`. Il faut dire à l'agent Wazuh de le remonter. Éditer `C:\Program Files (x86)\ossec-agent\ossec.conf`, ajouter dans `<ossec_config>` :

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Activer aussi l'audit Windows qui manque par défaut (sinon pas de 4688 avec ligne de commande, pas de 4769 exploitable) — `gpedit.msc` ou en ligne de commande :

```powershell
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Service Ticket Operations" /success:enable
# afficher la ligne de commande dans le 4688
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```

Redémarrer l'agent : `Restart-Service WazuhSvc`.

---

## 5. Écrire des règles de détection (côté serveur Wazuh)

Les règles custom vont dans `/var/ossec/etc/rules/local_rules.xml` sur l'Ubuntu. Wazuh décode déjà Sysmon et range chaque Event ID dans un groupe (`sysmon_event1`, `sysmon_event_10`…). On s'accroche à ces groupes avec `if_group`.

**Exemple 1 — accès à LSASS (signature d'un dump type Mimikatz)**
Sysmon Event 10 = un processus ouvre un handle sur un autre. Si la cible est `lsass.exe`, c'est suspect.

```xml
<group name="local,sysmon,">
  <rule id="100010" level="12">
    <if_group>sysmon_event_10</if_group>
    <field name="win.eventdata.targetImage">lsass\.exe</field>
    <description>Accès au processus LSASS — possible vol d'identifiants (T1003.001)</description>
    <mitre>
      <id>T1003.001</id>
    </mitre>
  </rule>
</group>
```

**Exemple 2 — web shell : un serveur web lance un shell**
Sysmon Event 1 = création de processus. Si `w3wp.exe` ou `apache` a pour parent-enfant un `cmd.exe`/`powershell.exe`, c'est un web shell.

```xml
<group name="local,sysmon,">
  <rule id="100011" level="12">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.parentImage">w3wp\.exe|httpd\.exe|nginx\.exe</field>
    <field name="win.eventdata.image">cmd\.exe|powershell\.exe</field>
    <description>Serveur web lançant un interpréteur — possible web shell (T1505.003)</description>
    <mitre>
      <id>T1505.003</id>
    </mitre>
  </rule>
</group>
```

Après édition : `sudo systemctl restart wazuh-manager`.
Le champ exact d'un événement se lit dans le dashboard (**Threat Hunting**, on déplie un log Sysmon et on copie le nom `win.eventdata.xxx`).

---

## 6. La boucle Purple Team (le cœur de l'exercice)

Pour chaque technique : **attaquer → observer → écrire la règle → rejouer → confirmer**.

| # | Attaque (depuis Kali/Windows) | Trace attendue | Où la voir dans Wazuh |
| --- | --- | --- | --- |
| 1 | `nxc smb` brute force / spray | salve de **4625** | filtre `data.win.system.eventID:4625` |
| 2 | `impacket-psexec` | **7045** (service) + **4624** type 3 | recherche `7045` |
| 3 | Kerberoasting (`GetUserSPNs`) | **4769** chiffrement `0x17` (RC4) | `4769` + `ticketEncryptionType:0x17` |
| 4 | Dump LSASS (Mimikatz) | **Sysmon 10** vers `lsass.exe` | ta règle `100010` |
| 5 | Web shell (module 3) | **Sysmon 1** parent web → shell | ta règle `100011` |
| 6 | Création compte de persistance | **4720** / **4732** | recherche `4720` |

**Atomic Red Team** automatise ce rejeu proprement (une commande par technique ATT&CK) :

```powershell
# sur le Windows, en administrateur
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics
Invoke-AtomicTest T1003.001   # dump LSASS → doit déclencher ta règle 100010
```

---

## 7. Livrable module 9

1. Screenshot du dashboard avec une alerte de chaque technique rejouée.
2. Le tableau **attaque → trace → règle Sigma/Wazuh → remédiation** (10 lignes).
3. Deux règles écrites de ta main dans `local_rules.xml`, commentées et taguées MITRE.

C'est exactement ce qui te démarque en entretien : tu ne dis pas seulement « j'ai eu Domain Admin », tu dis « et voici ce qui aurait dû le détecter ».

---

## 8. Traduire en Sigma (portable, multi-SIEM)

Une règle **Sigma** est indépendante du SIEM : on l'écrit une fois, on la convertit (Splunk, Sentinel, Elastic, Wazuh) avec `sigma-cli`. Exemple, accès LSASS :

```yaml
title: Acces LSASS - vol d'identifiants
status: experimental
logsource:
  product: windows
  category: process_access
detection:
  selection:
    TargetImage|endswith: '\lsass.exe'
    GrantedAccess: '0x1410'
  condition: selection
level: high
tags:
  - attack.credential_access
  - attack.t1003.001
```

Savoir passer de Sigma → règle Wazuh est un plus concret à montrer.

---

## À savoir expliquer à l'oral (module 9)

- **Pourquoi Sysmon en plus des journaux Windows ?** Les logs natifs ne donnent ni la ligne de commande complète, ni les accès inter-processus, ni le hash des binaires. Sysmon comble ce manque.
- **Différence EDR / SIEM ?** L'EDR est l'agent sur le poste (voit et bloque localement) ; le SIEM centralise et corrèle les logs de tout le parc. Wazuh fait un peu des deux.
- **Qu'est-ce qu'un faux positif et comment le réduire ?** Une alerte légitime (admin qui fait vraiment du PsExec) : on affine la règle (exclusion de comptes/hôtes connus) plutôt que de la supprimer.
- **Purple Team vs Red/Blue séparées ?** Red et Blue travaillent ensemble, attaque par attaque, pour valider que chaque technique est bien détectée — c'est ta valeur de profil mixte.
