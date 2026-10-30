# Cheatsheet — Évasion de kiosque (kiosk breakout)

Sortir d'une appli plein écran verrouillée (borne, kiosque Windows, session RDP bridée) pour atteindre un **shell** sur le Windows sous-jacent.

> ⚠️ Labs / périmètres autorisés uniquement. Le principe : un kiosque n'est qu'une **appli au-dessus d'un Windows complet** — toute fenêtre système qui s'ouvre est une porte.

---

## 1. Le réflexe : faire apparaître une boîte de dialogue

Une boîte **Ouvrir / Enregistrer sous / Imprimer / Importer** est une **fenêtre de l'Explorateur** → on peut y naviguer, taper un chemin, lancer un exe.

Comment la faire surgir :
- Bouton « Ouvrir un fichier », « Joindre », « Importer », « Parcourir ».
- **Provoquer une erreur** qui ouvre un dialogue (ex. désactiver une fonction attendue côté serveur → le client affiche une fenêtre système). *(méthode Touch : scan désactivé via le web → erreur → dialogue)*
- `Ctrl + O` (Ouvrir), `Ctrl + S` / `Ctrl + Shift + S` (Enregistrer), `Ctrl + P` (Imprimer → « Imprimer dans un fichier » → chemin).

---

## 2. Une fois la boîte « Ouvrir » affichée

```
1. Barre d'adresse / champ "Nom" -> taper :  C:\Windows\System32
2. Afficher *.* (tous les fichiers)
3. Repérer cmd.exe  (ou powershell.exe)
4. Clic DROIT -> "Ouvrir"            <- lance un shell hors du kiosque
```
Variantes utiles dans ce champ :
- Taper directement `cmd.exe`, `powershell`, `C:\Windows\System32\cmd.exe`.
- Taper `file://C:/` pour naviguer.
- Clic droit sur un fichier → « Ouvrir avec », « Exécuter en tant que… ».

---

## 3. Autres portes classiques

- **Raccourcis Windows** : `Win+E` (Explorateur), `Win+R` (Exécuter), `Ctrl+Shift+Esc` (Gestionnaire des tâches → « Exécuter une nouvelle tâche » → `cmd`), `Win+D`, `Shift×5` (Sticky Keys).
- **Liens hypertextes** dans l'appli → navigateur → `file:///C:/` ou téléchargement d'un exe.
- **Dialogue d'impression** → imprimante « Microsoft Print to PDF » → chemin de sortie = navigation disque.
- **Menus d'aide** (`?` / F1) qui ouvrent un navigateur embarqué.

---

## 4. Après l'évasion (RDP)

Clavier souvent en **QWERTY** → repasser en AZERTY :
```cmd
powershell -c "Set-WinUserLanguageList -LanguageList fr-FR,en-US -Force"
```
Au lancement FreeRDP 3.x : `/kbd:layout:0x0000040C` (AZERTY FR).

Puis énumérer : `whoami /priv`, `netstat -ano`, services, fichiers de conf/`.bat` avec creds, bases locales (MySQL 3306, etc.).

---

## 5. Côté défense

- Désactiver les boîtes de dialogue fichier et `Ctrl+O`/`Ctrl+P` dans l'appli kiosque.
- **Assigned Access** durci / Shell restreint, AppLocker interdisant `cmd.exe`, `powershell.exe`, `taskmgr.exe`.
- Désactiver Sticky Keys et les raccourcis système, masquer la barre des tâches.
- Compte kiosque à **privilèges minimaux**, pas d'accès disque hors dossier applicatif.

## Fiches liées
- [windows-privesc](windows-privesc.md) · [commandes-utiles](commandes-utiles.md) · Machine : [Touch](../machines/Touch.md)
