# Documentation client DSI : accéder en SSH à votre serveur

Oct 26, 2026 · @Randria

## Présentation

Ce document est remis au client DSI. Son objectif unique : vous permettre d'ouvrir une session `ssh admin@10.10.10.101` sur votre serveur depuis votre machine locale, via un tunnel WireGuard. Aucune couche applicative n'est traitée ici : l'usage du serveur (applications, configurations internes) relève de vos équipes et n'est pas encadré par ce document.

## Votre serveur

| Élément | Valeur |
| --- | --- |
| Nom de la VM | `dsi` (ID 101) |
| Système | Rocky Linux 10 |
| CPU | 4 cœurs |
| RAM | 8 Go |
| Disque | 40 Go |
| Adresse privée | `10.10.10.101` (réseau interne, non joignable depuis Internet) |
| Adresse VPN | `10.20.20.11` (attribuée par votre fichier de tunnel `dsi.conf`) |

Le serveur est isolé des autres machines du parc : toutes les connexions entrantes sont bloquées, sauf le trafic de votre tunnel WireGuard.

## Votre seul accès, pour l'instant : SSH via le VPN

```bash
ssh -i votre-cle admin@10.10.10.101
```

Cet accès ne fonctionne que **tunnel WireGuard actif**. Sans tunnel, l'adresse `10.10.10.101` est invisible : c'est la protection de base.

## Activer votre tunnel WireGuard

### 1. Télécharger l'application

Votre poste étant **Windows**, l'installez depuis l'une de ces deux sources :

| Source | Comment |
| --- | --- |
| Site officiel : wireguard.com/install | Télécharger l'installateur Windows, puis *Next → Accept → Finish* ; l'application s'installe aussi comme service Windows *WireGuardTunnel* |
| Microsoft Store | Chercher « WireGuard » et installer depuis le store |

(Les autres plateformes — macOS, Linux, Android, iOS — disposent du même logiciel ; consulter wireguard.com/install si besoin.)

### 2. Importer le fichier de configuration

Vous avez reçu un fichier `dsi.conf` ressemblant à ceci (pour le créer ou le modifier vous-même : `notepad dsi.conf`, ou WireGuard → *Edit* une fois importé) :

```
[Interface]
PrivateKey = <clé privée — ne jamais la modifier ni la partager>
Address = 10.20.20.11/32

[Peer]
PublicKey = <clé publique du serveur>
Endpoint = IP-DU-SERVEUR:51820
AllowedIPs = 10.10.10.101/32
PersistentKeepalive = 25
```

À noter : `Endpoint` contient une **adresse IP**, pas un domaine — c'est le but en l'absence d'enregistrement DNS.

- **Windows** (votre cas) : Ouvrir WireGuard → *Import tunnel(s) from file* (ou Ctrl+O) → sélectionner `dsi.conf` → bouton **Activer**. L'icône dans la barre des tâches passe en coche vert quand le tunnel est *Active*.
- **macOS / Linux / Android / iOS** : même logique (*Import from file* ou QR code) ; sous Linux en ligne de commande :

```bash
sudo cp dsi.conf /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf
sudo wg-quick up wg0        # activé
sudo wg-quick down wg0      # désactivé
# Pour l'activer automatiquement au démarrage :
sudo systemctl enable --now wg-quick@wg0
```

### 3. Vérifier la connexion

Dans **PowerShell** :

```powershell
# Le tunnel doit apparaître avec un handshake récent
wg show

# Le serveur DSI doit répondre
ping 10.10.10.101
```

Le champ `AllowedIPs` est volontairement limité à votre seule VM : même en modifiant le fichier, vous ne pouvez pas atteindre les autres machines du parc. Toute tentative d'élargissement est rejetée côté serveur.

## Première connexion SSH

Générez votre paire de clés personnelles **dans PowerShell**, puis transmettez la clé **publique** à l'hébergeur (jamais la privée) :

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\cle-dsi" -N ""   # sans passphrase ; ajoutez-en si vous préférez
type "$env:USERPROFILE\.ssh\cle-dsi.pub"
```

(OpenSSH Client est intégré à Windows 10 et suivants ; s'il manque : *Paramètres → Applications → Fonctionnalités facultatives → Ajouter OpenSSH Client*.)

Côté hébergeur, la clé est ajoutée au trousseau `/root/cles-clients/dsi.pub`, puis injectée dans la VM lors du provisioning.

Connexion (tunnel actif), toujours dans **PowerShell** :

```powershell
ssh -i "$env:USERPROFILE\.ssh\cle-dsi" admin@10.10.10.101
```

Compte initial fourni : `admin` (appartenant au groupe `wheel`, donc capable de `sudo`).

## Ce que vous n'avez PAS

| Non fourni | Conséquence |
| --- | --- |
| Accès à l'interface Proxmox (port 8006) | La création, l'arrêt et la restauration sont gérées par l'hébergeur |
| Accès aux autres VM du parc | L'isolation réseau bloque toute vue |
| Accès web public | Rien n'est exposé sur Internet ; l'accès SSH passe uniquement par le tunnel |
| Sauvegardes gérées par vos soins | L'hébergeur planifie quotidiennement ; précisez vos contraintes de rétention |

## Dépannage

| Symptôme | Cause probable | Solution |
| --- | --- | --- |
| `ping 10.10.10.101` échoue | Tunnel inactif ou handshake absent | `wg show` : en l'absence de handshake, vérifier que le tunnel est *Active*, puis réessayer |
| Handshake absent malgré le tunnel activé | `Endpoint` incorrect, ou firewall local bloquant UDP | Vérifier le port `51820/udp` (le cas échéant, sur le routeur de votre site) ; côté Windows, regarder l'onglet *Log* de l'application |
| `ssh` : commande introuvable / terme non reconnu | Client OpenSSH absent | *Paramètres → Applications → Fonctionnalités facultatives → Ajouter OpenSSH Client*, puis rouvrir PowerShell |
| SSH refusé après le tunnel établi | Clé privée absente ou service `sshd` arrêté | Signaler à l'hébergeur, qui intervient via sa propre route d'administration |
| Erreur `Host key verification failed` | Clé d'un ancien serveur en cache | `ssh-keygen -R 10.10.10.101`, puis se reconnecter |
| Tunnel coupé sans raison | Changement de réseau / veille de l'ordinateur | Désactiver puis réactiver le tunnel (bouton *Désactiver* / *Activer*) |

## Checklist client

- [ ] Application WireGuard installée (Windows : installateur ou Microsoft Store)
- [ ] Client OpenSSH présent dans PowerShell (`ssh -V` répond)
- [ ] Fichier `dsi.conf` importé et tunnel activé (`wg show` dans PowerShell : handshake récent)
- [ ] Paire de clés SSH personnelle créée, clé **publique** transmise
- [ ] Première connexion `ssh admin@10.10.10.101` réussie
- [ ] Mot de passe du compte `admin` référencé dans votre gestionnaire de mots de passe
- [ ] Contacts de l'hébergeur enregistrés (escalade, horaires)

## Checklist hébergeur (à faire AVANT remise du document)

- [ ] VM 101 `dsi` créée et démarrée, `qm list` la montre en running
- [ ] `qm resize 101 scsi0 40G` fait, IP `10.10.10.101/24` configurée via cloud-init
- [ ] Pare-feu de la VM : règle SSH réservée à `10.10.10.1` (l'hôte)
- [ ] Tunnel côté serveur : bloc `[Peer]` avec `AllowedIPs = 10.20.20.11/32` + ligne `PostUp` d'isolation ajoutée dans `wg0.conf`
- [ ] `wg syncconf wg0 <(wg-quick strip wg0)` exécuté sans coupure des autres peers
- [ ] `Endpoint` renseigné dans le fichier client : IP publique distante ou `192.168.88.2` si emploi en LAN
- [ ] Test d'isolement fait : sans tunnel, `ssh admin@10.10.10.101` échoue ; avec, tout marche
- [ ] Pas de domaine, pas de HAProxy, pas de port public pour cette VM en phase 1
- [ ] Fichier `dsi.conf` remis au client par un canal chiffré, jamais par email en clair
