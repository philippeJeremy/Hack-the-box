# Module 1 — Fondamentaux réseau

> ⚠️ Exercices sur labs autorisés uniquement (art. 323-1). Voir [`index`](index.md).

## Objectif
Comprendre comment deux machines se parlent, pour savoir **où écouter, où frapper, et où se cacher une trace**. Sans ce socle, tout le reste est du bruit.

## Pourquoi c'est clé pour le poste
Un pentester lit un réseau comme un plan de bâtiment : quelles portes existent, lesquelles sont ouvertes, qui parle à qui. En environnement industriel, la **segmentation** (IT/OT) est *le* sujet — tu dois savoir raisonner en couches et en zones.

## Concepts

**Les couches, version utile.** Retiens 4 niveaux, pas les 7 de l'OSI par cœur :
- **Lien (L2)** — adresses MAC, le réseau local. Attaques : ARP spoofing, VLAN hopping.
- **Réseau (L3)** — adresses IP, le routage. C'est là qu'on parle de *segmentation* et de *pivot*.
- **Transport (L4)** — **TCP** (fiable, poignée de main SYN/SYN-ACK/ACK) vs **UDP** (rapide, sans garantie). Le scan de ports vit ici.
- **Application (L7)** — HTTP, SMB, DNS, SSH… le service réel. C'est ce qu'on énumère et exploite.

**Adressage & sous-réseaux.** Une IP + un masque (`/24` = 256 adresses) définit un segment. Lire un CIDR te dit *l'étendue* d'un réseau : `10.0.0.0/8` est immense, `192.168.1.0/24` est un petit LAN. En mission, la carte des sous-réseaux = la carte des zones de confiance.

**Ports & services.** Un port ouvert = une porte avec un service derrière. Les incontournables à connaître de tête :
- 21 FTP · 22 SSH · 23 Telnet · 25 SMTP · 53 DNS · 80/443 HTTP(S)
- 88 Kerberos · 135/139/445 SMB/RPC · 389/636 LDAP(S) · 3389 RDP
- 1433 MSSQL · 3306 MySQL · 5985/5986 WinRM

Les ports 88/389/445 ensemble = tu es probablement face à un **Active Directory** (module 5).

**Le rôle du DNS.** Résolution nom→IP, mais aussi source d'infos (enregistrements, transferts de zone mal configurés). Souvent sous-estimé.

## Méthodologie
1. **Situer** : quelle IP ai-je, quel sous-réseau, quelle passerelle ?
2. **Cartographier** : quelles autres machines répondent ?
3. **Identifier les portes** : quels ports/services par machine ?
4. **Prioriser** : quel service est le plus prometteur (version connue, mal configuré) ?

Le réseau ne s'attaque pas — il se *lit* pour préparer les modules suivants.

## Outils & rôle
- **nmap** ([fiche](../outils/nmap.md)) — l'outil roi : découverte d'hôtes, scan de ports, détection de version, scripts NSE. À maîtriser à fond.
- **wireshark / tcpdump** — lire le trafic pour *comprendre* un protocole (indispensable côté Blue et pour l'OT).

## Pièges courants
- Scanner trop fort, trop vite → on rate des ports (et on se fait repérer). Un scan réfléchi > un scan bourrin.
- Oublier l'**UDP** (DNS, SNMP, TFTP vivent là) — beaucoup de découvertes s'y cachent.
- Confondre « port ouvert » et « vulnérable » : un port ouvert est une *question*, pas une réponse.

## Côté défense (Blue)
- Un scan laisse une **signature** : beaucoup de connexions vers beaucoup de ports en peu de temps → détectable par IDS (Suricata, Zeek).
- La **segmentation** (VLAN, pare-feu inter-zones) est la contre-mesure structurelle : même compromis, un attaquant ne doit pas pouvoir tout atteindre.
- Journaliser les flux inter-segments = voir un pivot anormal.

## À l'oral
- *« Différence TCP/UDP et pourquoi ça change le scan ? »* → TCP a une poignée de main (on sait si le port répond), UDP non (réponse ou silence ambigu → plus lent, moins fiable).
- *« Tu vois 88, 389, 445 ouverts, tu en déduis quoi ? »* → un contrôleur de domaine AD ; je bascule sur l'énumération d'annuaire.
- *« C'est quoi une bonne segmentation réseau ? »* → des zones de confiance séparées par des pare-feu, le moindre flux nécessaire autorisé, tout le reste bloqué et journalisé.

## Pratiquer
Toute machine HTB commence ici : un scan nmap propre. Reprends la lecture des scans de tes machines déjà faites (Lame, Blue…) : qu'est-ce que le scan te *disait* avant même l'exploit ?
