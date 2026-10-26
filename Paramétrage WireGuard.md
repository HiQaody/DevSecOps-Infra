# Documentation Complète : WireGuard sur Proxmox

**Version** : 6.0 — Procédure complète avec IP publique `41.188.46.76`  
**Date** : 27 octobre 2026  
**Auteur** : @Randria

---

## 📋 Table des matières
1. [Installation de WireGuard](#1-installation-de-wireguard)
2. [Génération des secrets du serveur](#2-génération-des-secrets-du-serveur)
3. [Configuration du serveur](#3-configuration-du-serveur)
4. [Script d'automatisation `wg-manager.sh`](#4-script-dautomatisation-wg-managersh)
5. [Tâches effectuées par l'administrateur](#5-tâches-effectuées-par-ladministrateur)
6. [Note à envoyer au client (Aldice)](#6-note-à-envoyer-au-client-aldice)
7. [Vérifications et tests](#7-vérifications-et-tests)
8. [Dépannage](#8-dépannage)

---

## 1. Installation de WireGuard

### 1.1 Installer les paquets nécessaires

```bash
apt update && apt install wireguard qrencode -y
```

**Vérification** :
```bash
wg --version
# Doit afficher : wireguard-tools v1.0.x
```

### 1.2 Activer le routage IP

```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-vpn.conf
sysctl -p /etc/sysctl.d/99-vpn.conf
```

**Vérification** :
```bash
sysctl net.ipv4.ip_forward
# Doit afficher : net.ipv4.ip_forward = 1
```

---

## 2. Génération des secrets du serveur

### 2.1 Créer le dossier de configuration

```bash
cd /etc/wireguard
mkdir -p clients
chmod 700 clients
```

### 2.2 Générer les clés du serveur

⚠️ **IMPORTANT** : Le `umask 077` est obligatoire pour protéger les clés privées.

```bash
umask 077
wg genkey | tee server.key | wg pubkey > server.pub
```

**Vérification** :
```bash
ls -l server.*
# Doit afficher :
# -rw------- 1 root root 45 ... server.key
# -rw------- 1 root root 45 ... server.pub

cat server.pub
# Exemple : aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890abcde=
```

### 2.3 Sauvegarde critique

**Conservez `server.key` en DEUX endroits** :
1. Sur l'hôte Proxmox : `/etc/wireguard/server.key`
2. Dans votre gestionnaire de mots de passe (Bitwarden, KeePass, etc.)

⚠️ **Sans cette clé, tous les clients devront être re-provisionnés en cas de perte du serveur.**

---

## 3. Configuration du serveur

### 3.1 Créer le fichier `/etc/wireguard/wg0.conf`

```bash
nano /etc/wireguard/wg0.conf
```

**Contenu initial** (sans clients) :

```ini
[Interface]
Address = 10.20.20.1/24
ListenPort = 51820
PrivateKey = <REMPLACER_PAR_LE_CONTENU_DE_server.key>

# Règle de base : tout bloquer par défaut sur wg0
PostUp = iptables -A FORWARD -i wg0 -j DROP
PostDown = iptables -D FORWARD -i wg0 -j DROP

# ⚠️ ROUTAGE RETOUR : Décommentez les 2 lignes ci-dessous SI la passerelle par défaut
# de la VM n'est PAS l'hôte Proxmox (nécessaire si la VM ne sait pas comment répondre
# au réseau 10.20.20.0/24).
# PostUp = iptables -t nat -A POSTROUTING -s 10.20.20.0/24 -o vmbr1 -j MASQUERADE
# PostDown = iptables -t nat -D POSTROUTING -s 10.20.20.0/24 -o vmbr1 -j MASQUERADE
```

**Remplacez** `<REMPLACER_PAR_LE_CONTENU_DE_server.key>` par :
```bash
cat /etc/wireguard/server.key
```

### 3.2 Appliquer les permissions

```bash
chmod 600 /etc/wireguard/wg0.conf
ls -l /etc/wireguard/wg0.conf
# Doit afficher : -rw------- 1 root root ...
```

### 3.3 Activer le service

```bash
systemctl enable --now wg-quick@wg0
```

**Vérification** :
```bash
wg show
# Doit afficher :
# interface: wg0
#   public key: <clé publique du serveur>
#   private key: (hidden)
#   listening port: 51820

ip -br a | grep wg0
# Doit afficher : wg0 UNKNOWN/UP 10.20.20.1/24
```

### 3.4 Vérifier l'accessibilité du port

**Test local** :
```bash
ss -ulnp | grep 51820
# Doit afficher une ligne avec 51820
```

**Test externe** (depuis un autre ordinateur ou smartphone en 4G) :
```bash
nc -zuv 41.188.46.76 51820
# Doit afficher : Connection to 41.188.46.76 51820 port [udp/*] succeeded!
```

⚠️ **Si le port n'est pas accessible** : Vérifiez que le pare-feu du fournisseur cloud/FAI autorise le port 51820/UDP entrant.

---

## 4. Script d'automatisation `wg-manager.sh`

### 4.1 Créer le script

```bash
nano /usr/local/bin/wg-manager.sh
```

**Contenu complet** :

```bash
#!/bin/bash
set -e

# Configuration
WG_CONF="/etc/wireguard/wg0.conf"
WG_DEV="wg0"
WG_DIR="/etc/wireguard"
CLIENTS_DIR="$WG_DIR/clients"
SERVER_ENDPOINT="41.188.46.76:51820"

# Vérification root
if [ "$EUID" -ne 0 ]; then
  echo "❌ Ce script doit être exécuté en tant que root (sudo)."
  exit 1
fi

# Création du dossier clients
mkdir -p "$CLIENTS_DIR"
chmod 700 "$CLIENTS_DIR"

usage() {
  echo "Usage:"
  echo "  $0 add-pub <nom_client> <ip_client_wg> <ip_vm_cible> <cle_publique_client>"
  echo "  $0 revoke <nom_client> <ip_client_wg> <ip_vm_cible>"
  echo ""
  echo "Exemple ajout   : $0 add-pub aldice 10.20.20.12 10.10.10.101 'abcd1234...='"
  echo "Exemple révocation: $0 revoke aldice 10.20.20.12 10.10.10.101"
  exit 1
}

if [ "$1" == "add-pub" ] && [ "$#" -eq 5 ]; then
  CLIENT_NAME=$2
  CLIENT_WG_IP=$3
  VM_IP=$4
  CLIENT_PUB=$5

  echo "📝 Mise à jour de $WG_CONF pour $CLIENT_NAME..."
  
  # 1. Nettoyer d'éventuelles anciennes règles pour ce client
  sed -i "/# RULE $CLIENT_NAME/d" "$WG_CONF"
  
  # 2. Insérer les règles iptables AVANT le premier [Peer]
  sed -i "0,/\[Peer\]/s//PostUp = iptables -I FORWARD -i $WG_DEV -s $CLIENT_WG_IP -d $VM_IP -j ACCEPT # RULE $CLIENT_NAME\nPostDown = iptables -D FORWARD -i $WG_DEV -s $CLIENT_WG_IP -d $VM_IP -j ACCEPT # RULE $CLIENT_NAME\n\n[Peer]/" "$WG_CONF"

  # 3. Ajouter le bloc Peer à la fin
  cat <<EOF >> "$WG_CONF"

# Client: $CLIENT_NAME
[Peer]
PublicKey = $CLIENT_PUB
AllowedIPs = $CLIENT_WG_IP/32
EOF

  echo "🛡️ Application des règles iptables en live..."
  iptables -I FORWARD -i $WG_DEV -s $CLIENT_WG_IP -d $VM_IP -j ACCEPT

  echo "🔄 Synchronisation de l'interface $WG_DEV (sans coupure)..."
  wg syncconf $WG_DEV <(wg-quick strip $WG_CONF)

  # 4. Générer le modèle à envoyer au client
  SERVER_PUB=$(cat "$WG_DIR/server.pub")

  cat <<EOF > "$CLIENTS_DIR/modele_$CLIENT_NAME.conf"
# --- INSTRUCTIONS POUR LE CLIENT ---
# 1. Ouvrez ce fichier avec un éditeur de texte.
# 2. Remplacez "<COLLER_VOTRE_CLE_PRIVEE_WIREGUARD_ICI>" par le contenu de votre fichier wg_client_private.key
# 3. Importez ce fichier dans votre application WireGuard.
# -----------------------------------

[Interface]
PrivateKey = <COLLER_VOTRE_CLE_PRIVEE_WIREGUARD_ICI>
Address = $CLIENT_WG_IP/32

[Peer]
PublicKey = $SERVER_PUB
Endpoint = $SERVER_ENDPOINT
AllowedIPs = $VM_IP/32
PersistentKeepalive = 25
EOF
  chmod 600 "$CLIENTS_DIR/modele_$CLIENT_NAME.conf"

  echo "✅ Client $CLIENT_NAME ajouté avec succès."
  echo "📄 Modèle prêt à être envoyé au client : $CLIENTS_DIR/modele_$CLIENT_NAME.conf"
  echo "💡 N'oubliez pas d'ajouter la clé publique SSH du client dans ~/.ssh/authorized_keys sur la VM $VM_IP !"

elif [ "$1" == "revoke" ] && [ "$#" -eq 4 ]; then
  CLIENT_NAME=$2
  CLIENT_WG_IP=$3
  VM_IP=$4

  echo "🗑️ Révocation de l'accès pour $CLIENT_NAME..."
  sed -i "/# RULE $CLIENT_NAME/d" "$WG_CONF"
  
  awk -v name="$CLIENT_NAME" '
    BEGIN { skip=0 }
    $0 == "# Client: " name { skip=1; next }
    /^\[Peer\]/ && skip==1 { skip=0; next }
    { if (!skip) print }
  ' "$WG_CONF" > "$WG_CONF.tmp" && mv "$WG_CONF.tmp" "$WG_CONF"

  iptables -D FORWARD -i $WG_DEV -s $CLIENT_WG_IP -d $VM_IP -j ACCEPT 2>/dev/null || true
  wg syncconf $WG_DEV <(wg-quick strip $WG_CONF)
  rm -f "$CLIENTS_DIR/modele_$CLIENT_NAME.conf"

  echo "✅ Client $CLIENT_NAME révoqué avec succès."
else
  usage
fi
```

### 4.2 Rendre le script exécutable

```bash
chmod +x /usr/local/bin/wg-manager.sh
```

### 4.3 Utilisation

**Ajouter un client** :
```bash
wg-manager.sh add-pub aldice 10.20.20.12 10.10.10.101 "0r4Hg/8AZhPES2N51Rzr+O+6tJVeqymyLRZLwpagrT0="
```

**Révoquer un client** :
```bash
wg-manager.sh revoke aldice 10.20.20.12 10.10.10.101
```

---

## 5. Tâches effectuées par l'administrateur

### 5.1 Checklist complète

- [x] **Installation** de WireGuard et qrencode
- [x] **Activation** du routage IP (`net.ipv4.ip_forward=1`)
- [x] **Génération** des clés serveur (`server.key` et `server.pub`)
- [x] **Sauvegarde** de `server.key` dans le gestionnaire de mots de passe
- [x] **Création** du fichier `/etc/wireguard/wg0.conf` avec les permissions 600
- [x] **Activation** du service `wg-quick@wg0`
- [x] **Vérification** de l'accessibilité du port 51820/UDP
- [x] **Installation** du script `wg-manager.sh`
- [x] **Création** du dossier `/etc/wireguard/clients/`

### 5.2 Pour chaque nouveau client (exemple : Aldice)

**Étape 1 : Demander les clés au client**

Envoyez cette note au client (voir section 6).

**Étape 2 : Ajouter le client WireGuard**

Une fois que le client vous a envoyé sa clé publique WireGuard :

```bash
wg-manager.sh add-pub aldice 10.20.20.12 10.10.10.101 "<clé publique WireGuard du client>"
```

**Étape 3 : Configurer l'accès SSH sur la VM**

Connectez-vous à la VM `10.10.10.101` et ajoutez la clé SSH publique du client :

```bash
ssh admin@10.10.10.101

# Une fois sur la VM :
mkdir -p ~/.ssh
chmod 700 ~/.ssh

echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAXLLr2gjNTFf009Icv1Q81bd3EBFx9+BSUnBvKxz1GS aldice-universite" >> ~/.ssh/authorized_keys

chmod 600 ~/.ssh/authorized_keys
```

**Étape 4 : Envoyer le fichier de configuration au client**

Le fichier `/etc/wireguard/clients/modele_aldice.conf` a été créé automatiquement par le script.

Envoyez ce fichier au client avec les instructions (voir section 6).

**Étape 5 : Mettre à jour l'inventaire**

Ajoutez cette entrée dans `/etc/wireguard/inventaire.csv` :

```csv
nom,ip_wg,ip_vm,date_ajout,cle_publique_wg,cle_publique_ssh
aldice,10.20.20.12,10.10.10.101,2026-10-27,0r4Hg/8AZhPES2N51Rzr+O+6tJVeqymyLRZLwpagrT0=,ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAXLLr2gjNTFf009Icv1Q81bd3EBFx9+BSUnBvKxz1GS
```

---

## 6. Note à envoyer au client (Aldice)

### 6.1 Premier email : Demande de génération des clés

**Objet** : Configuration de votre accès VPN - Génération des clés requise

```
Bonjour Aldice,

Pour configurer votre accès VPN sécurisé à votre machine virtuelle, nous avons besoin que vous génériez vos clés de chiffrement sur votre propre ordinateur. Cela garantit que votre clé privée ne transite jamais par nos serveurs.

=== ÉTAPE 1 : Génération des clés WireGuard ===

Sur votre ordinateur, ouvrez un terminal et exécutez :

**Linux / macOS :**
cd ~
umask 077
wg genkey | tee wg_client_private.key | wg pubkey > wg_client_public.key

**Windows (PowerShell avec WireGuard installé) :**
cd $env:USERPROFILE
wg genkey | tee wg_client_private.key | wg pubkey > wg_client_public.key

Si la commande "wg" n'est pas trouvée sur Windows, installez d'abord WireGuard depuis :
https://www.wireguard.com/install/

**Action requise :**
Envoyez-moi UNIQUEMENT le contenu du fichier wg_client_public.key (une ligne de 44 caractères se terminant par "=").

⚠️ NE M'ENVOYEZ PAS le fichier wg_client_private.key ! Gardez-le précieusement sur votre ordinateur.

=== ÉTAPE 2 : Génération de la clé SSH (si ce n'est pas déjà fait) ===

Si vous n'avez pas encore de clé SSH pour accéder à votre VM :

**Linux / macOS :**
ssh-keygen -t ed25519 -f ~/.ssh/cle_vm -N ""

**Windows (PowerShell) :**
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\cle_vm -N ""

**Action requise :**
Envoyez-moi le contenu du fichier cle_vm.pub (ou cle_vm.pub sur Windows).

⚠️ NE M'ENVOYEZ PAS le fichier cle_vm (sans .pub) ! C'est votre clé privée SSH.

=== Prochaines étapes ===

Une fois que j'aurai reçu vos deux clés publiques (WireGuard et SSH), je vous enverrai :
1. Votre fichier de configuration VPN (modele_aldice.conf)
2. Les instructions pour l'importer dans WireGuard
3. Les instructions pour vous connecter à votre VM

Cordialement,
@Randria
```

### 6.2 Deuxième email : Envoi de la configuration VPN

**Objet** : Votre configuration VPN est prête - Instructions d'installation

**Pièce jointe** : `modele_aldice.conf`

```
Bonjour Aldice,

Votre configuration VPN est prête. Veuillez suivre ces instructions étape par étape.

=== ÉTAPE 1 : Compléter le fichier de configuration ===

1. Ouvrez le fichier joint "modele_aldice.conf" avec un éditeur de texte (Bloc-notes, VS Code, nano, etc.)

2. Repérez cette ligne :
   PrivateKey = <COLLER_VOTRE_CLE_PRIVEE_WIREGUARD_ICI>

3. Remplacez "<COLLER_VOTRE_CLE_PRIVEE_WIREGUARD_ICI>" par le contenu de votre fichier wg_client_private.key (généré à l'étape précédente).

   Exemple :
   Avant : PrivateKey = <COLLER_VOTRE_CLE_PRIVEE_WIREGUARD_ICI>
   Après : PrivateKey = aB3cD4eF5gH6iJ7kL8mN9oP0qR1sT2uV3wX4yZ5aB6c=

4. Sauvegardez le fichier sous le nom "aldice.conf" (gardez l'extension .conf)

=== ÉTAPE 2 : Installer WireGuard ===

**Windows :**
1. Téléchargez depuis : https://www.wireguard.com/install/
2. Installez l'application
3. Lancez WireGuard

**macOS :**
brew install --cask wireguard

**Linux (Ubuntu/Debian) :**
sudo apt install wireguard-tools

**Mobile (iOS/Android) :**
Téléchargez l'application "WireGuard" depuis l'App Store ou Google Play.

=== ÉTAPE 3 : Importer le tunnel ===

**Windows :**
1. Ouvrez l'application WireGuard
2. Cliquez sur "Import tunnel(s) from file" (ou Ctrl+O)
3. Sélectionnez le fichier "aldice.conf"
4. Le tunnel apparaît dans la liste

**macOS / Linux :**
1. Ouvrez un terminal
2. Copiez le fichier : sudo cp aldice.conf /etc/wireguard/wg0.conf
3. Appliquez les permissions : sudo chmod 600 /etc/wireguard/wg0.conf
4. Activez le tunnel : sudo wg-quick up wg0

**Mobile :**
1. Ouvrez l'application WireGuard
2. Cliquez sur "+" ou "Add Tunnel"
3. Choisissez "Create from file or archive"
4. Sélectionnez le fichier "aldice.conf"

=== ÉTAPE 4 : Activer le tunnel ===

1. Dans l'application WireGuard, sélectionnez votre tunnel
2. Cliquez sur "Activate" ou "Activer"
3. L'icône doit devenir verte (ou bleue sur mobile)
4. Vérifiez que "Latest handshake" affiche un timestamp récent (quelques secondes)

=== ÉTAPE 5 : Tester la connexion ===

Ouvrez un terminal (ou PowerShell sur Windows) et exécutez :

ssh admin@10.10.10.101

Si c'est la première connexion, vous verrez un message demandant de confirmer l'empreinte de la clé. Tapez "yes" et appuyez sur Entrée.

Vous devriez maintenant être connecté à votre VM !

=== Dépannage ===

**Erreur "Connection timed out" :**
- Vérifiez que le tunnel est bien activé (icône verte)
- Vérifiez que votre pare-feu local ne bloque pas WireGuard

**Erreur "Network is unreachable" :**
- Le tunnel n'est pas activé. Cliquez sur "Activate"

**Erreur "Permission denied (publickey)" :**
- La clé SSH n'est pas correctement configurée sur la VM. Contactez-moi.

**Le tunnel se déconnecte fréquemment :**
- Vérifiez votre connexion Internet
- Le paramètre "PersistentKeepalive = 25" dans le fichier de config aide à maintenir la connexion

Cordialement,
@Randria
```

---

## 7. Vérifications et tests

### 7.1 Vérification côté serveur (Proxmox)

```bash
# État du tunnel
wg show

# Doit afficher pour chaque client actif :
# peer: <clé publique du client>
#   endpoint: <IP du client>:<port>
#   allowed ips: 10.20.20.XX/32
#   latest handshake: X seconds ago
#   transfer: X.XX KiB received, X.XX KiB sent

# Règles iptables
iptables -L FORWARD -n -v | grep wg0

# Doit afficher :
#   0     0 ACCEPT  all  --  wg0    *       10.20.20.12          10.10.10.101
#   0     0 DROP    all  --  wg0    *       0.0.0.0/0            0.0.0.0/0
```

### 7.2 Test d'isolement (obligatoire)

**Test 1 : Sans VPN**
```bash
# Depuis la machine du client (VPN désactivé) :
ssh admin@10.10.10.101
# Doit ÉCHOUER (timeout) : la VM est invisible depuis Internet
```

**Test 2 : Avec VPN actif**
```bash
# Depuis la machine du client (VPN activé) :
ssh admin@10.10.10.101
# Doit fonctionner
```

**Test 3 : Isolation entre clients**
```bash
# Depuis la machine du client (VPN activé) :
ping 10.10.10.102   # Autre VM
# Doit ÉCHOUER (timeout) : isolation stricte
```

### 7.3 Vérification de l'accessibilité publique

Depuis un ordinateur **extérieur** au réseau (ex: smartphone en 4G) :

```bash
nc -zuv 41.188.46.76 51820
# Doit afficher : Connection to 41.188.46.76 51820 port [udp/*] succeeded!
```

---

## 8. Dépannage

### 8.1 Handshake absent malgré le tunnel activé

**Causes possibles** :
- Port 51820/UDP bloqué par le pare-feu du fournisseur cloud/FAI
- Pare-feu local du client (Windows Defender, etc.)
- CGNAT chez le FAI du client

**Diagnostic** :
```bash
# Sur Proxmox
wg show   # Vérifier "latest handshake"

# Test du port depuis l'extérieur
nc -zuv 41.188.46.76 51820
```

### 8.2 Le client ne peut pas joindre sa VM

**Vérifications** :
1. Le tunnel est-il actif ? (`wg show` côté client)
2. La règle `iptables` existe-t-elle ? (`iptables -L FORWARD -n -v | grep wg0`)
3. La VM répond-elle au ping depuis l'hôte Proxmox ? (`ping 10.10.10.101`)
4. Le pare-feu de la VM autorise-t-il le SSH depuis `10.20.20.0/24` ?

### 8.3 SSH échoue même si le ping fonctionne

**Causes possibles** :
- Clé SSH non ajoutée dans `~/.ssh/authorized_keys`
- Permissions incorrectes sur `~/.ssh` ou `authorized_keys`
- SSH daemon configuré pour refuser les connexions

**Diagnostic** :
```bash
ssh -vvv -i ~/.ssh/cle_vm admin@10.10.10.101   # Mode verbeux
```

### 8.4 Le port 51820 n'est pas accessible depuis l'extérieur

**Solutions** :
1. Vérifiez la console de votre fournisseur cloud et ouvrez le port 51820/UDP
2. Vérifiez le pare-feu Proxmox : `pve-firewall status`
3. Vérifiez que WireGuard écoute : `ss -ulnp | grep 51820`

---

## 📎 Annexes

### A. Commandes utiles

```bash
# Voir l'état de tous les tunnels
wg show

# Voir les règles iptables liées à wg0
iptables -L FORWARD -n -v | grep wg0

# Redémarrer le service (coupe brièvement tous les tunnels)
systemctl restart wg-quick@wg0

# Voir les logs du service
journalctl -u wg-quick@wg0 -f

# Tester l'accessibilité du port depuis l'extérieur
nc -zuv 41.188.46.76 51820
```

### B. Inventaire des clients

Maintenez un fichier `/etc/wireguard/inventaire.csv` :

```csv
nom,ip_wg,ip_vm,date_ajout,cle_publique_wg,cle_publique_ssh
aldice,10.20.20.12,10.10.10.101,2026-10-27,0r4Hg/8AZhPES2N51Rzr+O+6tJVeqymyLRZLwpagrT0=,ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAXLLr2gjNTFf009Icv1Q81bd3EBFx9+BSUnBvKxz1GS
```

### C. Structure des fichiers

```
/etc/wireguard/
├── wg0.conf              # Configuration principale (permissions 600)
├── server.key            # Clé privée du serveur (permissions 600)
├── server.pub            # Clé publique du serveur (permissions 600)
├── inventaire.csv        # Inventaire des clients
└── clients/
    ├── modele_aldice.conf    # Fichier à envoyer au client
    ├── modele_jacquinot.conf # Fichier à envoyer au client
    └── ...
```

---

**Fin de la documentation.** Pour toute question ou support, contactez @Randria.