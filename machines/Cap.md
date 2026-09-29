# Cap — HackTheBox

| | |
| --- | --- |
| **Plateforme** | HackTheBox |
| **OS** | Linux |
| **Difficulté** | 🟢 Easy |
| **Date** | 2026-__-__ |
| **Vecteur** | <foothold : à compléter> → <privesc : à compléter> |
| **CVE** | <à compléter — probablement aucune : misconfig> |
| **Tags** | web · idor · pcap · linux-privesc · capabilities |

> `<IP_CIBLE>` = IP de session. tun0 = `10.10.14.x` · cible = `10.129.x.x`.

> 🎯 **Phase 3 — retour Linux, web + privesc par capability.** Deux nouveautés vs Shocker/Optimum :
> une faille **web logique** (pas un CVE), et une privesc Linux que tu n'as pas encore vue.
> Déroule ta fiche `lire-peas` côté Linux et pense **au-delà de sudo/SUID**.

---

## TL;DR

<À remplir en fin de box.>

---

## 1. Reconnaissance

```bash
nmap -sC -sV -p- -oA nmap/cap <IP_CIBLE>
```

| Port | Service | Version | Piste |
| --- | --- | --- | --- |
| | | | |

> Note bien **tous** les services ouverts : l'un d'eux resservira une fois des identifiants récupérés.

---

## 2. Énumération web

Application web à explorer **à la main** (clique, observe les URL, regarde ce que fait chaque page).

- Sections de l'appli : `______`
- Une page génère/affiche des données par **identifiant dans l'URL**. Question : que se passe-t-il
  si tu **changes cet identifiant** ? (pense **IDOR** — contrôle d'accès défaillant, OWASP A01)
- Type de fichier récupérable : `______`

> Réflexe : une ressource numérotée dans l'URL (`/x/1`, `?id=1`) → teste les autres valeurs,
> surtout **0**. Regarde aussi ce que tu télécharges : le format t'oriente sur l'outil pour l'analyser.

---

## 3. Accès initial (foothold)

**Méthode :** exploiter la faille web → obtenir un fichier → l'**analyser** pour en extraire des identifiants → les rejouer sur un service d'accès distant repéré à la recon.

- Fichier obtenu et outil d'analyse : `______` (indice : quel outil lit une capture réseau ?)
- Ce que j'y trouve : `______` (attention aux **protocoles en clair** — quels ports transportent des identifiants sans chiffrement ?)
- Service où je rejoue les identifiants : `______`

```bash
# à compléter : analyse du fichier + connexion au service
```

- **Utilisateur obtenu :** `______` (⚠️ pas root)

**Flag user :** home de l'utilisateur (non publié).

> Leçon de méthode : une **fuite d'info** (fichier lisible) alimente l'étape suivante ;
> un identifiant capturé sur un protocole **non chiffré** est réutilisable ailleurs (réutilisation de creds).

---

## 4. Énumération privesc (Linux)

Déroule `linux-privesc` / `lire-peas`. Ici, ne t'arrête PAS à `sudo -l` et aux SUID :

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null     # SUID
getcap -r / 2>/dev/null                     # ← la famille à regarder de près ici
```

- Piste retenue : `______`

> Rappel `lire-peas` (Linux, famille 3) : les **capabilities** donnent à un binaire un droit fin
> sans SUID complet. Une capability mal posée sur un interpréteur = escalade.

---

## 5. Élévation de privilèges

**Méthode :** le binaire porteur d'une capability dangereuse peut, appelé directement, changer d'UID.

- Binaire + capability trouvés : `______`
- Va sur **[GTFOBins](https://gtfobins.github.io/)** → ce binaire → section **`Capabilities`**.

```bash
# à compléter (cf. GTFOBins, section Capabilities)
```

- **Résultat :** `uid=0(root)`
- Pourquoi ça marche (pour l'oral) : `______`

**Flag root :** `/root/` (non publié).

---

## 6. Remédiation

- <correctif web — comment empêcher l'IDOR>
- <correctif privesc — la capability>
- <durcissement transverse : protocoles chiffrés, etc.>

---

## 7. Côté Blue Team — détection

| Étape | Trace / observation | Détection |
| --- | --- | --- |
| IDOR | accès à des ressources d'autres id, séquence d'URL incrémentées | logs applicatifs : un compte accède à des objets qui ne sont pas les siens |
| creds capturés | (côté défense : pourquoi un protocole en clair est un risque) | |
| privesc capability | | |

---

## 8. Leçons

- <faille logique web (IDOR) vs faille technique — ce que ça change>
- <la capability : nouvelle famille de privesc pour moi>
- <piège / temps passé>

---

## Références

- OWASP A01 (Broken Access Control) — <https://owasp.org/Top10/A01_2021-Broken_Access_Control/>
- GTFOBins (Capabilities) — <https://gtfobins.github.io/>
- Fiches liées : [nmap](../outils/nmap.md) · [burp](../outils/burp.md) · [linux-privesc](../outils/linux-privesc.md) · [gtfobins](../outils/gtfobins.md) · [lire-peas](../outils/lire-peas.md)
