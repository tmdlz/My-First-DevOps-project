# Troubleshooting #01 — Interface réseau et accès Web UI après installation Proxmox

> **Date** : 11 février 2026
> **Contexte** : Installation Proxmox VE 9.1 sur Lenovo ThinkPad X1 Carbon 7th Gen (i5-8265U)
> **Statut** : ✅ Résolu

---

## Le problème

Après l'installation de Proxmox VE, la Web UI (`https://192.168.100.2:8006`) est inaccessible depuis le PC Windows. L'écran du serveur affiche bien le prompt de login, mais la connexion réseau ne fonctionne pas.

**Symptôme** : `ERR_CONNECTION_TIMED_OUT` dans le navigateur, `ping` en `Destination Host Unreachable` depuis Proxmox.

## Diagnostic pas à pas

### Étape 1 — Vérifier l'état des interfaces

```bash
ip a
```

**Ce que ça montre** : la liste de toutes les interfaces réseau avec leur état (UP/DOWN), leur adresse MAC et leur adresse IP.

**Résultat observé** :
- `nic0` (carte Ethernet Intel e1000e intégrée) → état **DOWN**
- `wlp82s0f3` (carte Wi-Fi) → état UP mais inutilisable par Proxmox
- `vmbr0` (bridge virtuel Proxmox) → état **UNKNOWN** car sa port slave (`nic0`) est DOWN

**Apprentissage** : le X1 Carbon 7th Gen n'a **pas de port Ethernet physique**. L'installeur Proxmox a détecté la carte réseau Intel intégrée, mais sans câble branché, l'interface est DOWN.

### Étape 2 — Ajouter un adaptateur USB Ethernet

Branché un hub USB-C avec port Ethernet sur le port **Thunderbolt** (pas le port d'alimentation). Après reboot :

```bash
ip a
```

**Nouvelle interface détectée** : `enx9405bb109d38` → état **UP**

Le nom `enxXXXXXXXXXXXX` est un nom d'interface "predictable" basé sur l'adresse MAC de l'adaptateur. C'est le standard systemd pour les interfaces réseau amovibles.

**Apprentissage** : `dmesg | tail -20` permet de voir les derniers messages du kernel, utile pour confirmer que Linux a bien reconnu un nouveau périphérique USB.

### Étape 3 — Reconfigurer le bridge Proxmox

Le bridge `vmbr0` était configuré avec `nic0` (interface interne inutilisable). Il faut le pointer vers l'adaptateur USB :

```bash
nano /etc/network/interfaces
```

**Fichier `/etc/network/interfaces`** — c'est LE fichier de configuration réseau sur Debian/Proxmox. Structure :

```
auto lo
iface lo inet loopback

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24       # IP statique du serveur
    gateway 192.168.1.1             # IP de la box internet
    bridge-ports enx9405bb109d38    # ← changé de nic0 vers l'adaptateur USB
    bridge-stp off                  # Spanning Tree Protocol désactivé (un seul bridge)
    bridge-fd 0                     # Forward Delay à 0 (pas de délai de convergence)
```

**Modification clé** : `bridge-ports nic0` → `bridge-ports enx9405bb109d38`

**Apprentissage** : le bridge (`vmbr0`) est un switch virtuel de couche 2. Il a besoin d'au moins une interface physique comme "port slave" pour communiquer avec le réseau physique. Sans interface UP en dessous, le bridge ne peut pas envoyer de trames Ethernet.

### Étape 4 — Corriger le sous-réseau

Même après le changement d'interface, `ping 192.168.100.1` reste en échec (100% packet loss).

Vérification côté PC Windows avec `ipconfig` : l'adresse IP est `192.168.1.80` avec passerelle `192.168.1.1`.

**Le problème** : pendant l'installation, Proxmox avait proposé `192.168.100.2/24` (probablement une adresse par défaut car aucun DHCP n'avait répondu à ce moment). Le réseau local réel est en `192.168.1.0/24`.

Trois fichiers à modifier :

```bash
# 1. Configuration réseau
nano /etc/network/interfaces
# Changer : address 192.168.100.2/24 → 192.168.1.100/24
# Changer : gateway 192.168.100.1   → 192.168.1.1

# 2. Résolution DNS
nano /etc/resolv.conf
# Changer : nameserver 192.168.100.1 → nameserver 192.168.1.1

# 3. Résolution de noms locale
nano /etc/hosts
# Changer : 192.168.100.2 → 192.168.1.100
```

**Apprentissage** : ces trois fichiers forment le trio de base de la configuration réseau Linux :
- `/etc/network/interfaces` → IP, masque, passerelle, interfaces
- `/etc/resolv.conf` → serveur(s) DNS
- `/etc/hosts` → résolution de noms locale (avant DNS)

### Étape 5 — Reboot et vérification

```bash
reboot
```

Après redémarrage, depuis le PC Windows : **https://192.168.1.100:8006** → Web UI Proxmox accessible ✅

## Commandes apprises

| Commande | Rôle |
|----------|------|
| `ip a` | Afficher toutes les interfaces réseau, leur état et leurs adresses IP |
| `ip link set <iface> up` | Activer manuellement une interface |
| `ip route` | Afficher la table de routage (default gateway + sous-réseaux) |
| `ping -I <iface> <ip>` | Ping en forçant une interface source spécifique |
| `dmesg \| tail -20` | Voir les 20 derniers messages du kernel (détection hardware) |
| `lsusb` | Lister les périphériques USB connectés |
| `nano <fichier>` | Éditeur de texte en terminal (Ctrl+O sauver, Ctrl+X quitter) |
| `systemctl restart networking` | Redémarrer le service réseau sans reboot |
| `ipconfig` (Windows) | Équivalent de `ip a` sur Windows |

## Fichiers Linux importants découverts

| Fichier | Rôle |
|---------|------|
| `/etc/network/interfaces` | Configuration des interfaces réseau (IP, gateway, bridges, VLANs) |
| `/etc/resolv.conf` | Configuration du serveur DNS |
| `/etc/hosts` | Table de résolution locale hostname → IP |
| `/etc/systemd/logind.conf` | Comportement système (dont action fermeture capot laptop) |

## Configuration capot fermé (bonus)

Pour utiliser le laptop comme serveur headless (capot fermé) :

```bash
nano /etc/systemd/logind.conf
# Décommenter et modifier :
HandleLidSwitch=ignore

systemctl restart systemd-logind
```

## Leçons retenues

1. **Toujours vérifier le sous-réseau réel** avec `ipconfig` (Windows) ou `ip a` (Linux) avant de configurer une IP statique.
2. **Un bridge sans interface physique UP est inutile** — le bridge est un switch virtuel, il a besoin d'un "câble" vers le monde physique.
3. **Les noms d'interface Linux sont prédictibles** : `eno*` (embarquée), `enp*` (PCI), `enx*` (MAC-based, souvent USB), `wlp*` (Wi-Fi).
4. **Thunderbolt ≠ alimentation** — sur les laptops avec deux ports USB-C, un seul transporte les données.
5. **La config réseau Linux repose sur 3 fichiers** qu'il faut garder cohérents entre eux.
