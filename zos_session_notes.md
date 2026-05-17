# Notes de session Hercules z/OS — Problème réseau CTCI
**Date :** 14 mai 2026  
**Statut :** En cours — réseau z/OS non fonctionnel

---

## Environnement

| Élément | Valeur |
|---------|--------|
| **Distribution** | ADCD z/OS 1.10 Summer (Application Development Controlled Distribution) |
| **Émulateur** | Hercules (mode z/Arch) |
| **OS hôte** | Linux (Ubuntu) |
| **Utilisateur TSO** | IBMUSER (mot de passe par défaut : SYS1) |
| **Répertoire Hercules** | `~/Documents/MAINFRAME/Z110SA/` |
| **Fichier de config** | `~/Documents/MAINFRAME/Z110SA/images/Z110/ADCD_LINUX.CONF` |
| **LPARNAME** | DUZA |
| **Hostname z/OS** | DUZA.DUZA.NET |
| **Accès console** | `c3270 192.168.0.211:23` (port 23, pas 3270 !) |

---

## Configuration réseau

### Côté Linux (hôte)
| Élément | Valeur |
|---------|--------|
| Interface physique | `enp10s0` |
| IP Linux | `192.168.0.9/24` |
| Gateway | `192.168.0.1` |
| Interface TUN Hercules | `tun0` |
| IP TUN (côté Linux) | `192.168.0.211` |
| IP TUN (côté z/OS) | `192.168.0.210` |

### Directive Hercules dans ADCD_LINUX.CONF (ligne 130)
```
0E20.2 3088 CTCI /dev/net/tun 1500 192.168.0.210 192.168.0.211 255.255.255.255
```
> Ancienne config LCS (`0E20 LCS 10.0.1.20`) remplacée par CTCI.

### Commandes Linux à exécuter avant de démarrer Hercules
```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.conf.enp10s0.proxy_arp=1
sudo ip route add 192.168.0.210/32 dev tun0
```
> ⚠️ Ces commandes ne sont pas persistantes — à relancer à chaque reboot Linux.

---

## Datasets importants

| Dataset | Rôle |
|---------|------|
| `ADCD.Z110S.PROCLIB(TCPIP)` | Proc de démarrage TCPIP |
| `ADCD.Z110S.TCPPARMS(PROFILE)` | Config principale TCPIP (contient `DATASETPREFIX`) |
| `ADCD.Z110S.TCPPARMS(PROF1)` | **Config réseau active** (HOME, ROUTES, DEVICE) |
| `ADCD.Z110S.TCPPARMS(HOSTS)` | Fichier hosts z/OS (créé/modifié en session) |
| `DUZA.TCPPARMS` | Ancienne tentative de config réseau — **NON utilisée par TCPIP** |
| `TCPIP.FTP.DATA` | Config serveur FTP |
| `/etc/resolve.conf` | Config résolution noms USS (sans le `v` !) |
| `ADCD.Z110S.PROCLIB(FTPD)` | Proc de démarrage FTP |

---

## Compilateur COBOL

| Élément | Valeur |
|---------|--------|
| Produit | IBM COBOL for z/OS V4R1 |
| Compilateur | `IGY410.SIGYCOMP` (volume JARES2, device 0A81) |
| Macros | `IGY410.SIGYMAC` (volume JAPRD2) |
| Procédures | `IGY410.SIGYPROC` (volume JAPRD2, contient PROC `IGYWCL`) |

---

## Problème réseau — État actuel

### Symptômes
- `ping 192.168.0.210` depuis Linux → **100% packet loss**
- `ping 192.168.0.9` depuis z/OS USS → **timeout**
- `ping 192.168.0.1` depuis z/OS USS → **timeout**
- `tcpdump -i tun0` sur Linux pendant ping z/OS → **0 packets** (les paquets ne quittent pas z/OS)
- Device Hercules `0E20` : statut `open` ✅
- Interface z/OS `CTC1` : adresse `192.168.0.210`, flag `P` (Primary) ✅
- Default gateway z/OS : pointait vers `192.168.0.5` (incorrect) → corrigé dans `PROF1`

### Ce qui a été modifié dans `ADCD.Z110S.TCPPARMS(PROF1)`
```
* DEVICE ADM1ETP MPCIPA NONROUTER      ← commenté
* LINK OSDL IPAQENET ADM1ETP           ← commenté

DEVICE CTCA1 CTC E20                   ← ajouté
LINK CTC1 CTC 1 CTCA1                  ← ajouté

HOME
    192.168.0.210 CTC1                 ← modifié (était 192.168.252.167)

BEGINRoutes
ROUTE 192.168.0.0 255.255.255.0 = CTC1 MTU 1492    ← modifié
ROUTE DEFAULT 192.168.0.1 CTC1 MTU 1492            ← modifié (était 192.168.252.2 via OSDL)
ENDRoutes
```

### Ce qui reste à investiguer
1. **Vérifier que PROF1 est bien lu par TCPIP** — la proc `ADCD.Z110S.PROCLIB(TCPIP)` référence `ADCD.Z110S.TCPPARMS(PROFILE)` qui contient `DATASETPREFIX ADCD.Z110S.TCPPARMS` — vérifier comment PROF1 est inclus/appelé
2. **Vérifier l'état de l'interface CTC1** avec `/D TCPIP,,NETSTAT` depuis SDSF après redémarrage TCPIP
3. **Vérifier si d'autres membres de TCPPARMS** référencent encore `OSDL` ou `192.168.252.x`
4. **Tester** `SRCHFOR '252'` dans `ADCD.Z110S.TCPPARMS` pour trouver d'autres références à l'ancienne config

---

## Problème FTP — État actuel

### Symptômes
- FTPD1 démarre mais `gethostbyname` warning persiste
- Port 21 en écoute sur `0.0.0.0` ✅
- Pas de connexion possible depuis Linux (réseau non fonctionnel)

### Config `TCPIP.FTP.DATA`
```
BANNER
  Welcome to z/OS FTP Server
ENDBANNER
PORT 21
INACTIVE 900
MAXUSERS 10
JESINTERFACELEVEL 2
NOREVERSE
HOSTNAME MAINFRAME
```

### `/etc/resolve.conf` USS (modifié)
```
TCPIPJobname TCPIP
DomainOrigin DUZA.NET
Domain DUZA.NET
DatasetPrefix ADCD.Z110S.TCPPARMS
Hostname DUZA
MESSAGECASE MIXED
ALWAYSWTO YES
```
> `NSINTERADDR` supprimé car DNS `9.24.112.254` (IBM interne) et `10.0.1.1` (ancienne config) non joignables.

### `ADCD.Z110S.TCPPARMS(HOSTS)`
```
127.0.0.1        LOOPBACK
192.168.0.210    DUZA  DUZA.DUZA.NET  MAINFRAME
```

---

## Commandes utiles rappel

| Action | Commande | Contexte |
|--------|----------|----------|
| Voir adresses IP actives | `/D TCPIP,,NETSTAT,HOME` | SDSF |
| Voir routes actives | `/D TCPIP,,NETSTAT,GATE` | SDSF |
| Voir connexions | `/D TCPIP,,NETSTAT,CONN` | SDSF |
| Redémarrer TCPIP | `/P TCPIP` puis `/S TCPIP` | SDSF |
| Démarrer FTP | `/S FTPD` | SDSF |
| Voir tâches actives | `DA` | SDSF |
| Voir logs système | `LOG` | SDSF |
| Shell Unix z/OS | Option `O` depuis menu ISPF | ISPF |
| Voir device Hercules | `devlist` | Console Hercules |
| Voir trafic réseau | `sudo tcpdump -i tun0 -n` | Terminal Linux |
