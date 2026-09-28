# Cheatsheet — Burp Suite

Le proxy d'interception, outil central de tout test web. On observe, on modifie, on rejoue chaque requête entre le navigateur et le serveur.

> ⚠️ **Cadre.** Labs et périmètres autorisés uniquement (PortSwigger Academy, Juice Shop en local, cible sous convention d'audit). Intercepter et rejouer du trafic hors périmètre écrit est une infraction (art. 323-1).

---

## 1. Rappels

- **Burp s'intercale en proxy** entre ton navigateur et la cible : tout le HTTP(S) passe par lui, tu le lis et le modifies.
- **Édition Community (gratuite)** : Proxy, Repeater, Decoder, Comparer, Intruder **bridé** (throttle). L'édition Pro lève le bridage et ajoute le Scanner. En apprentissage, Community + travail manuel suffit (et c'est ce qu'on attend à l'OSCP/PNPT).
- **Le réflexe** : tout se comprend d'abord **à la main dans Repeater**, on n'automatise (Intruder) qu'ensuite.

---

## 2. Mise en route

```
1. Proxy → Proxy settings : écoute sur 127.0.0.1:8080 (défaut).
2. Navigateur : proxy HTTP → 127.0.0.1:8080 (extension FoxyProxy pratique).
3. HTTPS : installer le certificat CA de Burp → http://burp → "CA Certificate"
   → importer dans le navigateur (autorités de confiance), sinon erreurs TLS.
4. Définir le périmètre : Target → Scope → ajouter le domaine cible.
   Puis "show only in-scope items" pour ne pas polluer avec le bruit.
```

> Sans le certificat CA importé, le HTTPS casse (le navigateur voit un certif Burp non fiable). C'est l'étape qu'on oublie.

---

## 3. Proxy — intercepter et modifier

| Onglet | Rôle |
| --- | --- |
| **Intercept** | Met en pause chaque requête ; on l'édite avant de la transmettre (`Forward`) |
| **HTTP history** | Journal de tout ce qui est passé (c'est là qu'on cherche après coup) |
| **WebSockets history** | Idem pour les WebSockets |

- `Intercept is on/off` : laisser **off** pour naviguer normalement, puis relire dans HTTP history. On active l'intercept seulement pour capturer une requête précise.
- Clic droit sur une requête → **Send to Repeater** (`Ctrl+R`) / **Send to Intruder** (`Ctrl+I`).

---

## 4. Repeater — rejouer à la main

Le cœur du travail. On modifie **un paramètre à la fois** et on observe la réponse.

- `Ctrl+R` pour y envoyer une requête, `Ctrl+Espace` (ou le bouton **Send**) pour l'envoyer.
- Onglets multiples : un par requête étudiée.
- Vue **Request / Response** côte à côte ; l'onglet **Render** affiche le rendu.
- Recherche dans la réponse (barre de recherche en bas) : longueur, mot-clé, message d'erreur.

Recettes typiques dans Repeater :
```
# Tester une IDOR : changer l'id et regarder si on accède à la donnée d'autrui
GET /api/account?id=1017   →   id=1018   (réponse 200 avec les données ? = faille)

# Tester une injection SQL : casser la syntaxe et lire l'erreur / la différence
username=admin'          → erreur 500 / message SQL ?
username=admin' -- -      → contournement d'auth ?

# Tester le contrôle d'accès : forcer une URL d'admin avec un compte lambda
GET /admin  avec le cookie de session d'un user standard → 200 ou 403 ?
```

---

## 5. Intruder — automatiser (fuzzing)

Sur une requête envoyée à Intruder, on marque les **positions** (`§valeur§`) à faire varier.

| Type d'attaque | Positions | Usage |
| --- | --- | --- |
| **Sniper** | 1 payload, 1 position à la fois | Fuzzing d'un seul champ |
| **Battering ram** | 1 payload, toutes les positions | Même valeur partout |
| **Pitchfork** | N payloads en parallèle | Couples user+pass alignés |
| **Cluster bomb** | Produit cartésien | Toutes les combinaisons user × pass |

Payloads : **Simple list** (dictionnaire), **Numbers** (plage, pour les IDOR), **Runtime file** (gros fichier).

Lecture des résultats : trier par **Status** et surtout par **Length** — une réponse de longueur différente trahit souvent le cas intéressant (login réussi, id valide, injection qui passe).

> ⚠️ Intruder est **throttlé en Community** (lent). Pour du brute-force réel on préfère `ffuf`/`hydra` ; Intruder sert au fuzzing ciblé et à la démonstration.

---

## 6. Decoder / Comparer / Sequencer

- **Decoder** : encode/décode URL, Base64, HTML, hex ; calcule des hash. Pratique pour lire un JWT ou un paramètre encodé.
- **Comparer** : diff visuel entre deux réponses (utile après une injection booléenne aveugle : vrai vs faux).
- **Sequencer** : analyse l'entropie des jetons de session (prédictibles ?).

---

## 7. Options qui font gagner du temps

- **Match and replace** (Proxy settings) : réécrire automatiquement un en-tête (ex. forcer `User-Agent`, injecter un `X-Forwarded-For`).
- **Target → Site map** : arborescence du site découverte passivement en naviguant.
- **Save project** : sauvegarder la session (Pro) pour reprendre l'audit.
- Extensions via **BApp Store** : *Autorize* (test d'autorisation/IDOR automatique), *JWT Editor*, *Logger++*.

---

## 8. Correspondance OWASP Top 10 → où chercher dans Burp

| Faille | Manœuvre Burp |
| --- | --- |
| Contrôle d'accès (IDOR) | Repeater : modifier `id`, rejouer avec une autre session ; extension Autorize |
| Injection SQL | Repeater : casser la requête, comparer les réponses ; relais vers sqlmap |
| XSS | Injecter un payload dans un champ réfléchi, lire dans Response/Render |
| CSRF | Vérifier l'absence de jeton anti-CSRF dans une requête sensible |
| SSRF | Repeater : pointer un paramètre URL vers un service interne / Burp Collaborator |
| Auth | Intruder pitchfork/cluster bomb sur le login (démonstration) |

---

## 9. Côté défense / détection

- Une utilisation de Burp = **beaucoup de requêtes depuis une même IP**, souvent avec des valeurs anormales (quotes, `../`, `<script>`), des codes 400/500 en rafale.
- Détection côté serveur : **WAF** (ModSecurity), corrélation de pics de requêtes, alertes sur motifs d'injection dans les logs applicatifs.
- Remédiations transverses : requêtes paramétrées (SQLi), échappement en sortie + CSP (XSS), contrôle d'accès **côté serveur** (IDOR), jeton anti-CSRF + `SameSite`, validation stricte des URL serveur (SSRF).

---

## 10. À savoir expliquer à l'oral

- **Pourquoi un proxy d'interception ?** Le navigateur applique la logique client (JS, validations) ; Burp permet de la contourner en agissant sur la requête réelle envoyée au serveur — d'où la règle « ne jamais faire confiance au client ».
- **Repeater vs Intruder** : Repeater = compréhension manuelle, une requête ; Intruder = automatisation d'un même schéma sur beaucoup de valeurs.
- **Pourquoi le certificat CA ?** Pour déchiffrer le HTTPS, Burp présente son propre certificat ; il faut donc le déclarer de confiance dans le navigateur.
- **Différence des 4 modes Intruder** et quand utiliser cluster bomb (toutes combinaisons) vs pitchfork (couples alignés).
