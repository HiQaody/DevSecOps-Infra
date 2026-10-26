# Documentation Proxmox VE : héberger des clients isolés

Oct 9, 2026 · @Randria

## Présentation

Proxmox VE permet de transformer un serveur physique en plusieurs serveurs virtuels, chacun isolé des autres, comme un VPS vendu par un hébergeur.

Ce document présente d'abord le cas d'utilisation, les avantages et les limites de Proxmox VE. Il décrit ensuite pas à pas comment héberger des clients isolés sur un seul serveur.

Le principe est la **virtualisation** : un hyperviseur découpe le matériel (CPU, RAM, disque, réseau) en machines virtuelles (VM). Chaque client reçoit sa propre VM, avec son système, ses utilisateurs et ses ressources, sans voir celles des autres.

## Cas d'utilisation

Proxmox VE convient surtout quand un serveur unique doit servir plusieurs utilisateurs qui ne doivent pas se voir.

| Cas | Description | Type recommandé |
| --- | --- | --- |
| Revente de VPS | Chaque client reçoit un serveur virtuel avec accès root. | VM (KVM) |
| Hébergement web mutualisé avancé | Un espace par client pour ses sites et bases de données. | Conteneur (LXC) ou VM |
| Environnements de test et de développement | Plusieurs équipes ou projets isolés sur un même serveur. | VM ou conteneur |
| Serveurs internes d'entreprise | Messagerie, fichiers, ERP et applications métier séparés. | VM |
| Laboratoire et formation | Étudiants ou stagiaires avec chacun leur machine à tester. | VM ou conteneur |
| Consolidation de serveurs | Remplacer plusieurs vieux serveurs par un seul plus puissant. | VM |

Pour votre besoin (donner un espace à chaque client sans qu'il voie les autres), le scénario de référence est la **revente de VPS** avec une VM KVM par client.

## Avantages

Proxmox VE est gratuit, complet et simple à administrer, ce qui en fait le meilleur point de départ pour vendre des VPS.

| Avantage | Détail |
| --- | --- |
| Gratuit et open source | Licence AGPL, sans coût de licence. L'abonnement payant est optionnel (support et dépôt stable). |
| Deux technologies en une | VM complètes (KVM) et conteneurs légers (LXC) gérés dans la même interface. |
| Isolation forte | Chaque VM a son propre noyau, ses ressources et son pare-feu. Le client ne voit pas les autres. |
| Interface web complète | Création, console, snapshots, sauvegardes et supervision depuis le navigateur, sans outil tiers. |
| Gestion fine des ressources | Limites par VM : CPU, RAM, disque, débit réseau, IOPS. |
| Stockage moderne | Prise en charge de ZFS, LVM, Ceph, NFS, avec snapshots et RAID logiciel. |
| Sauvegardes intégrées | Sauvegardes planifiées, et Proxmox Backup Server pour l'incrémental, la déduplication et le chiffrement. |
| Cloud-init et modèles | Un client est créé en quelques secondes en clonant un modèle (template). |
| Droits par utilisateur | Rôles et pools : un client peut gérer sa seule VM depuis l'interface. |
| API et automatisation | API REST, compatible avec WHMCS, Terraform, Ansible et vos propres scripts. |
| Évolutif | Passage possible à un cluster de plusieurs serveurs avec migration à chaud. |
| Grande communauté | Documentation officielle abondante et forum actif. |

## Limites

Proxmox VE n'est pas une plateforme d'hébergement clé en main : le portail client, la facturation et la sécurité restent à votre charge.

| Limite | Conséquence | Piste de solution |
| --- | --- | --- |
| Pas de portail client ni de facturation | Il faut un outil externe pour vendre et gérer les abonnements. | WHMCS, Virtualizor ou scripts via l'API |
| Conteneurs moins isolés | Les conteneurs LXC partagent le noyau de l'hôte, donc le risque est plus élevé. | Utiliser des VM KVM pour les clients externes |
| Serveur unique = point de défaillance | Une panne matérielle coupe tous les clients. | Sauvegardes externes, cluster à plusieurs nœuds |
| Courbe d'apprentissage | Réseau, stockage et pare-feu demandent des compétences Linux. | Tester sur un serveur de laboratoire d'abord |
| Support officiel payant | Sans abonnement, l'aide passe par le forum et la communauté. | Abonnement Community ou Basic si l'activité devient critique |
| Dépôt gratuit moins testé | Le dépôt « No-Subscription » reçoit les mises à jour plus tôt, avec un peu plus de risque de régression. | Mettre à jour hors heures de pointe, avec sauvegarde récente |
| Sécurité à gérer soi-même | Mises à jour, pare-feu, accès SSH et surveillance dépendent de vous. | Voir la section « Sécurité de l'hôte » |
| Ressources partagées | Une VM trop gourmande peut ralentir les autres sans limites. | Quotas CPU, RAM, disque, réseau et IOPS par VM |
| Haute disponibilité exigeante | Elle demande au moins 3 serveurs et un stockage partagé ou répliqué. | Prévoir le budget matériel avant de la promettre |
| Pas de GPU ni matériel spécifique simple | Le passage direct de matériel (passthrough) est possible mais délicat. | À étudier au cas par cas |

## Architecture cible et bonnes pratiques

Pour un débutant, la bonne pratique professionnelle tient en cinq règles : des disques redondants, des sauvegardes séparées du serveur, un réseau isolé, des accès administrateur protégés et une infrastructure documentée.

| Domaine | Choix recommandé | Pourquoi |
| --- | --- | --- |
| Stockage | LVM-thin sur sda, second pool vmdata sur sdb (option C) | Capacité cumulée, mais aucune redondance : une panne de disque impose une restauration depuis la sauvegarde externe. |
| Sauvegardes | Proxmox Backup Server sur une autre machine, plus une copie hors site | Le miroir ne protège ni d'une erreur, ni d'un vol, ni d'un incendie. |
| Réseau | `vmbr0` pour l'administration et l'IP publique, `vmbr1` privé (NAT) pour les VM | Les VM ne sont jamais sur le réseau d'administration. |
| Entrée web | Une seule VM HAProxy reçoit les ports 80 et 443 | Un seul point d'entrée à surveiller et à protéger. |
| Accès des clients | VPN WireGuard, une clé par client, limité à sa VM | Un client ne voit ni les autres VM ni l'hôte. |
| Accès administrateur | Compte nominatif, clés SSH, 2FA, interface 8006 jamais publique | Réduit fortement le risque de piratage. |
| Ressources | RAM non sur-allouée, quotas CPU, disque et réseau par VM | Un client gourmand ne pénalise pas les autres. |
| Mises à jour | Fenêtre planifiée, snapshot avant chaque mise à jour | Retour arrière rapide en cas de problème. |
| Supervision | Alertes par email : état du pool ZFS, SMART, espace disque | Les pannes sont vues avant les clients. |
| Documentation | Inventaire des VM, IP, domaines, clés et procédures | Une autre personne peut reprendre l'exploitation. |

### La règle 3-2-1 pour les sauvegardes

Gardez **3 copies** de vos données, sur **2 supports différents**, dont **1 hors du site**. Un serveur avec deux disques en miroir compte pour une seule copie : il faut donc au moins une machine de sauvegarde séparée et une copie à l'extérieur.

### Erreurs fréquentes des débutants

- Travailler tous les jours avec le compte `root` au lieu d'un compte nominatif.
- Exposer l'interface d'administration (port 8006) à Internet.
- Confondre snapshot, miroir de disques et sauvegarde : seule une copie sur une autre machine est une sauvegarde.
- Ne jamais tester une restauration.
- Sur-allouer la RAM : le serveur se met alors à swapper et toutes les VM ralentissent.
- Remplir le stockage ZFS à plus de 80 %.

## Choix du stockage (disques durs)

**Décision : option C retenue.** `sdb` complète `sda` pour les VM : `sda` garde Proxmox et le pool `local-lvm`, `sdb` devient un second pool `vmdata`. Il n'y a donc plus de disque de sauvegarde local : les sauvegardes vont sur une autre machine. L'installation ZFS RAID1 reste documentée pour une migration ultérieure.

### Situation actuelle

| Disque | Modèle | Taille | État | Usage actuel |
| --- | --- | --- | --- | --- |
| `/dev/sda` | SSDSCKJB240G7R | 240 Go | SMART PASSED, usure 0 % | Proxmox (partitions EFI et LVM) |
| `/dev/sdb` | SSDSCKJB240G7R | 240 Go | SMART PASSED, usure 0 % | Libre, non utilisé |

Proxmox voit chaque disque individuellement (SMART lisible), sans contrôleur RAID entre les deux : la configuration convient parfaitement à ZFS.

### Les trois options

| Option | Principe | Capacité utile | Si un disque tombe | Verdict |
| --- | --- | --- | --- | --- |
| **A. ZFS RAID1** | Les deux SSD en miroir | environ 220 Go | Le serveur continue de fonctionner | Recommandé |
| **B. Garder l'installation actuelle** | `sda` pour les VM, `sdb` pour des sauvegardes locales | environ 194 Go | Si `sda` tombe : arrêt et restauration | Solution de secours |
| **C. Cumuler les deux disques** | Les VM réparties sur `sda` et `sdb` | environ 430 Go | Perte possible de toutes les VM | Retenue : sauvegardes externes obligatoires |

L'option C cumule la capacité des deux disques, mais retire toute redondance et tout disque de sauvegarde local : les sauvegardes doivent aller sur une autre machine.

### Pourquoi ZFS RAID1

- **Tolérance de panne** : un disque peut tomber sans coupure, vous le remplacez à chaud ou lors d'un arrêt planifié.
- **Intégrité** : ZFS vérifie chaque bloc avec une somme de contrôle et détecte la corruption silencieuse.
- **Snapshots instantanés** : un retour arrière en quelques secondes avant chaque mise à jour.
- **Compression** : elle économise de l'espace sans effort.
- **Intégré à Proxmox** : installation, supervision et gestion depuis l'interface.

Le mot « RAID1 » dans l'installeur de Proxmox désigne un miroir ZFS : c'est le bon choix pour deux disques identiques.

### Réinstallation en ZFS RAID1

1. Démarrez sur la clé USB (F11, puis la clé en mode UEFI) et lancez *Install Proxmox VE (Graphical)*.
2. À l'étape *Target Harddisk*, cliquez **Options**.
3. Filesystem : **zfs (RAID1)**. Cochez `/dev/sda` et `/dev/sdb` ; les autres lignes doivent rester sur « Do not use ».
4. Onglet *Advanced Options* : `ashift` = 12, `compress` = on, `checksum` = on, `copies` = 1. Laissez la taille maximale du cache (ARC) proposée par l'installeur.
5. Terminez l'installation (pays, mot de passe root, email, réseau), puis redémarrez.

Attention : cette opération **efface les deux disques**. Elle est sans risque ici, car aucune VM n'a encore été créée.

### Vérifier après l'installation

```bash
zpool status                                   # rpool : les deux disques en ONLINE
zfs list                                       # espace disponible
pvesm status                                   # le stockage local-zfs est actif
cat /sys/module/zfs/parameters/zfs_arc_max     # limite du cache ZFS en octets
cat /etc/pve/storage.cfg                       # local-zfs doit porter l'option sparse
```

Si `zfs_arc_max` vaut 0, limitez le cache à 16 Go pour garder la RAM aux VM :

```bash
echo "options zfs zfs_arc_max=17179869184" > /etc/modprobe.d/zfs.conf
update-initramfs -u -k all
reboot
```

### Règles d'exploitation du stockage ZFS

- Ne dépassez pas **80 % de remplissage** du pool : au-delà, les performances chutent. Surveillez avec `zpool list`.
- Activez les alertes par email (*Datacenter → Notifications*) pour l'état du pool et SMART.
- Un contrôle complet (*scrub*) est lancé chaque mois par défaut : vérifiez son résultat avec `zpool status`.
- N'ajoutez jamais de RAID matériel sous ZFS.
- Un snapshot ne remplace pas une sauvegarde.
- Pour remplacer un disque défaillant : installez un SSD de taille égale ou supérieure, puis suivez la procédure `zpool replace` de la documentation Proxmox.

### Capacité à planifier

| Poste | Taille |
| --- | --- |
| Système Proxmox et ISO | environ 12 Go |
| Template cloud-init | 10 Go |
| VM HAProxy | 10 Go |
| VM DSI | 40 Go |
| VM tsenamilay et UF2 | 120 Go |
| **Total prévu** | **environ 192 Go** |
| Limite recommandée (80 % du pool) | environ 176 Go |

Ces chiffres valent pour ZFS RAID1 ; le total dépasse la limite recommandée de 80 %, il faut donc réduire l'allocation ou retenir l'option C. Avec l'option C, la capacité est répartie : environ 60 Go sur `sda` (limite 155 Go) et 120 Go sur `sdb` (limite environ 175 Go), soit une marge confortable sur chaque pool.

### Option C retenue : sdb complète sda pour les VM

Cette option utilise les deux SSD pour les VM : `sda` garde Proxmox et son pool `local-lvm` (environ 194 Go), `sdb` devient un second pool `vmdata` (environ 220 Go). La capacité totale pour les VM atteint environ 410 Go. En contrepartie, elle n'offre aucune redondance, et il n'y a plus de disque de sauvegarde local.

| Risque | Conséquence | Mesure de compensation |
| --- | --- | --- |
| Panne de `sda` | L'hôte et les VM du pool `local-lvm` sont perdus ; les services s'arrêtent | SSD de rechange, sauvegarde externe, plan de reprise écrit |
| Panne de `sdb` | Les VM du pool `vmdata` sont perdues ; l'hôte continue de fonctionner | Sauvegarde externe, restauration sur un SSD de remplacement |
| Aucune sauvegarde locale | Tout repose sur la machine de sauvegarde externe | Sauvegarde nocturne externe et test de restauration chaque mois |
| Pool plein | Les VM peuvent se corrompre | Surveillance de l'occupation, alerte à 80 % |

Deux disques sans redondance doublent à peu près le risque de panne qui vous touche : la sauvegarde externe n'est plus un confort, c'est la seule protection de vos données.

### Étape 1 : vérifier le disque sdb

Vérifiez d'abord que `sdb` est bien vide : toutes ses données seront effacées.

```bash
lsblk -o NAME,SIZE,MODEL,MOUNTPOINT /dev/sdb
wipefs -a /dev/sdb      # efface les anciennes signatures, seulement si le disque est vide
```

### Étape 2 : créer le pool vmdata

Par l'interface : *app → Disks → LVM-Thin → Create: Thinpool*, disque `/dev/sdb`, nom `vmdata`, avec *Add Storage* coché. Ou en ligne de commande :

```bash
pvcreate /dev/sdb
vgcreate vmdata /dev/sdb
lvcreate --type thin-pool -l 95%FREE -n data vmdata
pvesm add lvmthin vmdata --vgname vmdata --thinpool data --content rootdir,images
```

Ne rattachez jamais `sdb` au groupe de volumes `pve` de `sda` : un volume réparti sur deux disques perdrait toutes les VM à la panne de l'un d'eux. Gardez deux pools séparés.

Vérifiez avec `pvesm status` : les stockages `local-lvm` et `vmdata` doivent apparaître.

### Étape 3 : répartir les VM entre les deux disques

| VM | Stockage | Disque | Raison |
| --- | --- | --- | --- |
| Template 9000, HAProxy (100), DSI (101) | `local-lvm` | `sda` | Services d'infrastructure, avec l'hôte |
| tsenamilay (102), UF2 (103), futurs clients (110+) | `vmdata` | `sdb` | Une panne de `sdb` n'arrête ni HAProxy ni le VPN |

Pour créer une VM sur `sdb`, ajoutez `--storage vmdata` à la commande `qm clone`. Une VM peut aussi être déplacée plus tard : `qm move-disk <ID> scsi0 vmdata --delete 1`.

### Étape 4 : sauvegarder sur une autre machine

Ajoutez un stockage de sauvegarde externe dans *Datacenter → Storage → Add*. Choisissez selon votre matériel :

| Destination | Avantage | Limite |
| --- | --- | --- |
| Proxmox Backup Server sur une autre machine | Incrémental, dédupliqué, chiffré | Une machine à dédier |
| Partage NFS ou SMB (NAS ou autre PC) | Simple et rapide | Ne protège pas d'un sinistre sur le site |
| Disque USB externe branché au serveur | Peu coûteux, démarrage immédiat | À faire tourner et à stocker hors du site |
| Serveur distant ou stockage cloud chiffré | Protège d'un incendie ou d'un vol | Coût et débit à prévoir |

Exemple avec un partage NFS (adaptez l'adresse et le dossier) :

```bash
pvesm add nfs sauvegarde-ext --server 192.168.88.10 --export /export/proxmox --content backup
```

Puis planifiez les sauvegardes dans *Datacenter → Backup → Add* :

| Champ | Valeur |
| --- | --- |
| Storage | `sauvegarde-ext` (votre stockage externe) |
| Schedule | tous les jours à 02:00 |
| Selection mode | All |
| Mode | Snapshot |
| Compression | ZSTD |
| Retention | keep-daily 7, keep-weekly 4, keep-monthly 3 |
| Notification | votre email |

Sauvegardez aussi la configuration de l'hôte sur le même stockage :

```bash
tar czf /mnt/pve/sauvegarde-ext/config-hote-$(date +%F).tar.gz /etc/pve /etc/network/interfaces /etc/hosts /etc/fstab /etc/wireguard
```

### Étape 5 : surveiller les disques et les pools

```bash
lvs -o vg_name,lv_name,lv_size,data_percent,metadata_percent   # occupation des deux pools
smartctl -H /dev/sda                                           # santé du SSD système
smartctl -H /dev/sdb                                           # santé du SSD vmdata
```

Répétez ces contrôles chaque semaine, ou automatisez-les avec vos alertes email. Réagissez dès que `data_percent` dépasse 80 sur l'un des pools.

### Plan de reprise

**Si `sdb` tombe** (l'hôte continue de fonctionner) :

1. Remplacez le SSD par un modèle de taille égale ou supérieure.
2. Recréez le pool `vmdata` (étapes 1 et 2).
3. Restaurez les VM depuis la sauvegarde externe, par exemple `qmrestore /mnt/pve/sauvegarde-ext/dump/<fichier> 102 --storage vmdata`.

**Si `sda` tombe** (l'hôte est perdu) :

1. Remplacez le SSD par un modèle de taille égale ou supérieure.
2. Réinstallez la même version de Proxmox VE avec le même nom d'hôte et la même IP, **sans toucher à `sdb`**.
3. Rajoutez le stockage externe, puis restaurez la VM HAProxy en premier, et les autres ensuite. Les fichiers de configuration des VM étaient sur `sda` : restaurez-les depuis les sauvegardes plutôt que de réutiliser les disques de `sdb`.
4. Vérifiez les domaines, le VPN et les pare-feu.

Notez cette procédure hors du serveur et testez-la une fois avant d'accueillir des clients.

### Migrer vers ZFS RAID1 plus tard

Ce choix n'est pas définitif. Quand vous aurez un budget disques ou un cluster, vous pourrez réinstaller en ZFS RAID1 (section « Installation de Proxmox VE en ZFS RAID1 »), puis restaurer les VM depuis les sauvegardes externes.

## Guide détaillé : matériel et installation

Votre serveur, un Dell PowerEdge R640 (72 threads, environ 188 Go de RAM, deux SSD de 240 Go), convient à cet usage. Proxmox VE 9 est déjà installé sur `sda` (pool LVM-thin `local-lvm`) et `sdb` forme un second pool `vmdata` pour les VM (option C), comme détaillé dans « Choix du stockage ».

### Matériel recommandé

| Élément | Recommandation |
| --- | --- |
| CPU | Intel VT-x ou AMD-V activé dans le BIOS (obligatoire). Plus de cœurs permet plus de clients. |
| RAM | 64 Go minimum. Évitez de sur-allouer la RAM. |
| Disques | SSD ou NVMe en ZFS mirror ou RAID1. Pas de RAID matériel avec ZFS. |
| Réseau | 1 Gbit/s minimum, avec au moins une IP publique. |
| Électricité | Onduleur (UPS), surtout si le courant est instable. |

### Installation

1. Dans le BIOS (F2) : *Boot Mode* = UEFI et *Processor Settings → Virtualization Technology* = Enabled. Passez *Generic USB Boot* sur Enabled si la clé n'apparaît pas.
2. Téléchargez l'ISO sur proxmox.com/downloads et créez une clé USB (4 Go minimum, 8 Go conseillés) avec Rufus en mode DD ou balenaEtcher.
3. Démarrez avec F11, choisissez la clé en mode UEFI, puis *Install Proxmox VE (Graphical)*.
4. À l'étape *Target Harddisk*, choisissez **zfs (RAID1)** avec `sda` et `sdb` (voir « Choix du stockage »).
5. Renseignez le pays, le fuseau horaire (Indian/Antananarivo), un mot de passe root solide et un email.
6. Configurez le réseau : IP fixe, passerelle, DNS et nom d'hôte complet, par exemple `pve1.mondomaine.com`.
7. Terminez, retirez la clé, redémarrez, puis ouvrez `https://IP-DU-SERVEUR:8006`.

## Installation de Proxmox VE en ZFS RAID1 (pas à pas)

Cette procédure n'est pas nécessaire avec l'option B retenue : gardez-la pour une migration future. Elle installe Proxmox VE 9 sur les deux SSD de 240 Go en miroir ZFS, efface les deux disques et dure environ 15 minutes.

### 1. Avant de commencer

- [ ] Aucune donnée à conserver sur `sda` et `sdb` : les deux disques seront effacés.
- [ ] Une clé USB de 8 Go avec l'ISO de Proxmox VE 9 (voir étape 2).
- [ ] Un écran et un clavier branchés sur le serveur, ou la console virtuelle de l'iDRAC.
- [ ] Le câble réseau branché sur le port principal.
- [ ] Les informations réseau ci-dessous, notées à l'avance.

| Information | Exemple | À renseigner |
| --- | --- | --- |
| Nom d'hôte complet | `pve1.mondomaine.com` | Définitif : il ne se change plus après l'entrée dans un cluster |
| Adresse IP et masque | `192.168.88.2/24` | IP fixe, hors de la plage DHCP |
| Passerelle | `192.168.88.1` | À vérifier sur votre routeur |
| Serveur DNS | `1.1.1.1` | Ou le DNS de votre fournisseur |
| Mot de passe root | 12 caractères minimum | À conserver dans un gestionnaire de mots de passe |
| Email | adresse valide | Reçoit les alertes du serveur |

### 2. Créer la clé USB

Téléchargez la dernière ISO Proxmox VE 9 sur proxmox.com/downloads, puis vérifiez son empreinte SHA256 avec celle de la page de téléchargement :

```bash
# Windows (PowerShell ou invite de commandes)
certutil -hashfile proxmox-ve_9.x-1.iso SHA256

# Linux ou macOS
sha256sum proxmox-ve_9.x-1.iso
```

Puis écrivez l'ISO sur la clé :

- **Rufus** (Windows) : choisissez l'ISO, schéma de partition GPT, puis acceptez le mode **DD Image** quand Rufus le propose.
- **balenaEtcher** (Windows, macOS, Linux) : choisissez l'ISO et la clé, puis *Flash*.
- **Linux en ligne de commande** (vérifiez bien le nom de la clé avec `lsblk`, une erreur efface un autre disque) :

```bash
sudo dd if=proxmox-ve_9.x-1.iso of=/dev/sdX bs=1M status=progress conv=fsync
```

### 3. Régler le BIOS

Au démarrage, appuyez sur **F2** (System Setup) et vérifiez :

| Réglage | Valeur | Chemin |
| --- | --- | --- |
| Boot Mode | UEFI | System BIOS → Boot Settings |
| Virtualization Technology | Enabled | System BIOS → Processor Settings |
| Embedded SATA | AHCI Mode | System BIOS → SATA Settings |
| Generic USB Boot | Enabled (si la clé n'est pas détectée) | System BIOS → Boot Settings |

Enregistrez avec *Save Changes and Exit*.

### 4. Démarrer l'installeur

1. Branchez la clé sur un port USB, de préférence à l'arrière du serveur.
2. Au démarrage, appuyez sur **F11** (Boot Manager), puis choisissez *One-shot UEFI Boot Menu* et la clé USB.
3. Dans le menu bleu de Proxmox, choisissez **Install Proxmox VE (Graphical)**.
4. Lisez et acceptez le contrat de licence (*I agree*).

### 5. Choisir ZFS RAID1

À l'écran *Target Harddisk*, cliquez sur **Options** :

| Option | Valeur à choisir |
| --- | --- |
| Filesystem | **zfs (RAID1)** |
| Harddisk 0 | `/dev/sda` |
| Harddisk 1 | `/dev/sdb` |
| Autres disques | `-- do not use --` |
| compress | on |
| checksum | on |
| ashift | 12 |
| copies | 1 |
| ARC max size | valeur proposée par l'installeur |
| hdsize | taille maximale (voir remarque) |

Validez avec **OK**, puis **Next**. Si l'installeur affiche un avertissement sur un contrôleur RAID matériel, arrêtez-vous : ZFS doit voir les disques directement, ce qui est le cas ici puisque Proxmox lit leur SMART.

Remarque : si vous voulez pouvoir remplacer un jour un disque par un modèle légèrement plus petit, réduisez `hdsize` de quelques Go. Ce n'est pas obligatoire avec deux disques identiques.

### 6. Localisation, compte et réseau

1. *Country* : Madagascar. *Time zone* : Indian/Antananarivo. *Keyboard Layout* : celui de votre clavier (par exemple French).
2. Saisissez le mot de passe root et l'adresse email préparés à l'étape 1.
3. *Management Network Configuration* : interface réseau principale, nom d'hôte complet, adresse IP avec masque, passerelle et DNS.
4. Sur l'écran *Summary*, relisez tout. Laissez cochée l'option de redémarrage automatique, puis cliquez sur **Install**.

Retirez la clé USB quand le serveur redémarre.

### 7. Première connexion

Depuis un autre ordinateur du même réseau, ouvrez `https://IP-DU-SERVEUR:8006`.

- Le navigateur affiche « Non sécurisé » : c'est normal, le certificat est auto-signé.
- Utilisateur `root`, domaine *Linux PAM*, avec le mot de passe défini à l'installation.
- Le message « No valid subscription » est normal : fermez-le.

### 8. Vérifications immédiates

Dans *app → Shell* :

```bash
zpool status                  # rpool : état ONLINE, un miroir avec les deux disques
zpool list                    # taille et taux d'occupation du pool
zfs list                      # jeux de données et espace libre
pvesm status                  # stockages local et local-zfs actifs
proxmox-boot-tool status      # le chargeur de démarrage est installé sur les deux disques
```

Résultat attendu de `zpool status` (les noms des disques varient) :

```
  pool: rpool
 state: ONLINE
config:
        NAME                     STATE     READ WRITE CKSUM
        rpool                    ONLINE       0     0     0
          mirror-0               ONLINE       0     0     0
            ata-SSD...-part3     ONLINE       0     0     0
            ata-SSD...-part3     ONLINE       0     0     0
```

Si un des deux disques manque, ou si l'état n'est pas `ONLINE`, recommencez l'installation avant d'aller plus loin. La limite du cache ZFS se vérifie dans la section « Choix du stockage ».

### 9. Configuration de base après l'installation

1. Passez sur le dépôt No-Subscription et mettez à jour : voir la section « Dépôts et mises à jour ».
2. Créez un compte administrateur nominatif, pour ne plus travailler avec `root` :

```bash
pveum user add admin@pve --password 'MotDePasseSolide'
pveum acl modify / --users admin@pve --roles Administrator
```

3. Activez la double authentification sur ce compte : *Datacenter → Permissions → Two Factor*.
4. Configurez les alertes email : *Datacenter → Notifications*, et vérifiez que le serveur peut envoyer des emails.
5. Redémarrez une fois pour confirmer que le serveur repart seul, puis revérifiez `zpool status`.

### 10. Tester la tolérance de panne

Faites ce test avant d'accueillir le premier client, quand le serveur ne contient encore rien d'important. Il simule la panne d'un disque sans rien débrancher :

```bash
zpool status                                  # notez le nom exact d'un disque (partition 3)
zpool offline rpool <nom-du-disque-part3>     # simule la panne
zpool status                                  # état DEGRADED : le serveur fonctionne toujours
zpool online rpool <nom-du-disque-part3>      # remet le disque
zpool status                                  # attendez la fin de la resynchronisation, puis ONLINE
```

Si l'interface web reste accessible pendant l'état DEGRADED, le miroir fonctionne.

### 11. Remplacer un disque défaillant

Un disque en panne se reconnaît à l'état `DEGRADED` ou `FAULTED` dans `zpool status`, ou à une alerte SMART. Installez un SSD de taille égale ou supérieure, puis, avec le nom du disque sain et celui du nouveau (`ls -l /dev/disk/by-id`) :

```bash
# 1. Copier la table de partitions du disque sain vers le nouveau, puis régénérer ses identifiants
sgdisk <disque-sain> -R <nouveau-disque>
sgdisk -G <nouveau-disque>

# 2. Remplacer le disque dans le pool (partition 3)
zpool replace -f rpool <ancienne-partition-3> <nouvelle-partition-3>

# 3. Rendre le nouveau disque capable de démarrer (partition 2)
proxmox-boot-tool format <nouvelle-partition-2>
proxmox-boot-tool init <nouvelle-partition-2>
```

Suivez ensuite la resynchronisation avec `zpool status`. Comptez quelques minutes pour un disque presque vide.

### 12. Problèmes courants

| Symptôme | Cause probable | Solution |
| --- | --- | --- |
| La clé n'apparaît pas au démarrage | USB boot désactivé ou clé mal écrite | Activer *Generic USB Boot*, port arrière, recréer la clé en mode DD |
| « No Hard Disk found » | Disques non vus par le contrôleur | Ouvrir un terminal avec Ctrl+Alt+F2 et lancer `lsblk` ; vérifier le mode AHCI ou HBA |
| Écran noir après le menu | Problème de carte graphique | Dans le menu, choisir le mode *Terminal UI* ou ajouter `nomodeset` avec la touche E |
| Impossible d'ouvrir l'interface web | Mauvaise IP, câble ou pare-feu | Sur le serveur, vérifier `ip -br a` et que `https://IP:8006` est bien utilisé |
| Le serveur redémarre sur l'installeur | La clé USB est restée en premier | Retirer la clé, ou régler l'ordre dans *UEFI Boot Settings* |
| Le serveur ne démarre pas après l'installation | Entrée UEFI manquante | Dans *UEFI Boot Settings → UEFI Boot Sequence*, placer l'entrée Proxmox en premier |

Une fois cette section terminée, passez au réseau privé `vmbr1`, puis à la création du template et des VM.

## Guide détaillé : configuration, réseau et isolation

L'isolation entre clients repose sur trois éléments : un réseau dédié aux VM, le pare-feu Proxmox et les filtres MAC et IP.

### Dépôts et mises à jour

Sans abonnement payant, remplacez le dépôt « enterprise » par le dépôt gratuit « No-Subscription ». C'est l'erreur `apt-get update` affichée en rouge dans les tâches après l'installation. Soit par l'interface (*app → Updates → Repositories* : *Disable* sur les lignes enterprise, puis *Add* → No-Subscription), soit en ligne de commande :

```bash
mv /etc/apt/sources.list.d/pve-enterprise.sources /root/ 2>/dev/null
mv /etc/apt/sources.list.d/ceph.sources /root/ 2>/dev/null

cat > /etc/apt/sources.list.d/proxmox.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

apt update && apt full-upgrade -y
reboot
```

### Option A : réseau privé avec NAT

Cette option économise les adresses IP publiques. Créez d'abord le fichier : `nano /etc/network/interfaces`, puis ajoutez dans `/etc/network/interfaces` :

```
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up   iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE
```

Appliquez avec `ifreload -a`. Pour donner un accès SSH à un client, redirigez un port de l'hôte vers sa VM :

```bash
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 22101 -j DNAT --to 10.10.10.101:22
```

### Option B : une IP publique par VM

Attribuez une IP publique à chaque client sur `vmbr0`. Selon votre hébergeur, une adresse MAC virtuelle peut être nécessaire.

### Isolation entre clients

Activez le pare-feu Proxmox à trois niveaux :

1. *Datacenter → Firewall → Options* : Firewall = Yes.
2. Sur chaque VM, *Firewall → Options* : Firewall = Yes, **MAC filter = Yes**, **IP filter = Yes**, politique d'entrée (Input policy) = **DROP**.
3. Sur chaque VM, ajoutez uniquement les règles utiles au client : SSH 22, HTTP 80, HTTPS 443, etc.

Avec la politique DROP, une VM ne peut pas joindre les autres. Les filtres MAC et IP empêchent un client d'usurper l'adresse d'un autre.

## Guide détaillé : ajouter un client

Le template cloud-init (ID 9000) est créé dans la section « Création des VM : HAProxy, DSI et UF ». Une fois ce modèle prêt, ajouter un client prend quelques secondes.

### Créer la VM d'un client

```bash
# Clonage complet depuis le template 9000
qm clone 9000 110 --name dsi-uf --full

# Ressources, IP et accès
qm set 110 --cores 2 --memory 2048 \
  --ipconfig0 ip=10.10.10.110/24,gw=10.10.10.1 \
  --nameserver 1.1.1.1 \
  --ciuser client --sshkeys /root/dsi-uf.pub

# Disque de 40 Go
qm resize 110 scsi0 40G

# Réseau limité à 12,5 Mo/s (environ 100 Mbit/s), pare-feu actif
qm set 110 --net0 virtio,bridge=vmbr1,rate=12.5,firewall=1

qm start 110
```

Limitez aussi les entrées/sorties disque (`mbps_rd`, `mbps_wr`, `iops_rd`, `iops_wr`) pour qu'un client gourmand ne ralentisse pas les autres.

### Numérotation conseillée

| Élément | Convention |
| --- | --- |
| Template | ID 9000 |
| HAProxy | ID 100, IP 10.10.10.10 |
| DSI et UF | ID 101 à 103 |
| Futurs clients | ID 110, 111, 112… |
| IP privée | 10.10.10.ID (ex. VM 110 → 10.10.10.110) |
| Port SSH externe | 22 suivi de l'ID, via HAProxy (ex. 22110) |

## Guide détaillé : accès client, sauvegardes, sécurité et automatisation

Le plus sûr est de ne donner au client que l'accès SSH ou RDP à sa VM et de garder l'interface Proxmox pour vous.

### Accès du client à sa seule VM

Si le client doit gérer sa VM (démarrer, arrêter, console) depuis l'interface web :

```bash
pveum pool add client-dupont
pveum pool modify client-dupont --vms 110
pveum user add dupont@pve --password 'MotDePasseSolide'
pveum acl modify /pool/client-dupont --users dupont@pve --roles PVEVMUser
```

Il ne verra que sa VM.

### Sauvegardes

- **Plus de disque de sauvegarde local** : les deux SSD portent des VM. Toute sauvegarde doit donc aller sur une autre machine.
- **Proxmox Backup Server (PBS)**, gratuit, sur une machine séparée : sauvegardes incrémentales, dédupliquées et chiffrées. À défaut, un partage NFS ou SMB, ou un disque USB externe de secours.
- **Copie hors site** des sauvegardes (second PBS ou stockage distant) pour respecter la règle 3-2-1.
- Planification dans *Datacenter → Backup* : chaque nuit, mode *Snapshot*, compression *ZSTD*, rétention de 7 jours, 4 semaines et 3 mois.
- **Testez une restauration chaque mois** : une sauvegarde jamais testée n'est pas une garantie.
- Sauvegardez aussi la configuration de l'hôte (`/etc/pve`, `/etc/network/interfaces`) et gardez les clés et mots de passe en lieu sûr, hors du serveur.
- Faites un snapshot avant chaque mise à jour importante : le retour arrière est rapide, mais ce n'est pas une sauvegarde.

### Sécurité de l'hôte

- N'exposez pas l'interface 8006 ni le SSH de l'hôte à Internet : restreignez par IP ou passez par un VPN WireGuard.
- Authentification SSH par clés uniquement, avec `PermitRootLogin prohibit-password`.
- Activez la 2FA sur le compte root@pam (*Datacenter → Permissions → Two Factor*).
- Installez **fail2ban**.
- Mettez en place une supervision : Zabbix, Netdata ou Prometheus.

### Automatisation et facturation

Avec plusieurs clients, évitez de tout faire à la main :

| Outil | Usage |
| --- | --- |
| WHMCS avec module Proxmox | Création automatique, facturation, suspension en cas d'impayé. |
| Virtualizor | Panneau client et gestion de VPS clé en main. |
| API Proxmox VE | Scripts personnalisés (création, suppression, quotas). |
| OpenStack | Plateforme cloud complète, pour un projet plus ambitieux. |

## Création des VM : HAProxy, DSI et clients

Cette section crée quatre VM à partir du template cloud-init : HAProxy (reverse proxy), DSI, le client tsenamilay et UF2. Les noms, tailles et IP se changent dans le tableau et dans les commandes.

### Plan des VM

| ID | Nom | Rôle | CPU | RAM | Disque | IP privée |
| --- | --- | --- | --- | --- | --- | --- |
| 9000 | rock-tpl | Template (modèle) | 2 | 2 Go | 10 Go | aucune |
| 100 | haproxy | Reverse proxy des noms de domaine | 2 | 2 Go | 10 Go | 10.10.10.10 |
| 101 | dsi | Serveur DSI | 4 | 8 Go | 40 Go | 10.10.10.101 |
| 102 | tsenamilay | Serveur client | 4 | 8 Go | 80 Go | 10.10.10.102 |
| 103 | uf2 | Serveur UF n°2 | 4 | 8 Go | 40 Go | 10.10.10.103 |

Le template, HAProxy et DSI (60 Go) vont dans `local-lvm` (`sda`, 194 Go). tsenamilay et UF2 (120 Go) vont dans `vmdata` (`sdb`, environ 220 Go). Les deux pools restent bien sous la limite de 80 %.

### Étape 0 : préparer l'hôte

Dans *app → Shell* :

```bash
# Les stockages des VM : local-lvm (sda) et vmdata (sdb)
pvesm status

# Le réseau privé vmbr1 doit exister
ip -br a | grep vmbr1
```

Résultat attendu : `local-lvm` et `vmdata` actifs, et une ligne `vmbr1` avec l'adresse 10.10.10.1/24.

Vérifiez aussi que `sdb` est bien vierge avant d'en faire le pool `vmdata` :

```bash
lsblk -f /dev/sdb       # aucune FSTYPE ni MOUNTPOINT ne doit apparaître
pvs                     # seul /dev/sda3 (VG pve) doit être listé
```

Les ISOs utiles (Rocky 10.2 DVD, Debian 13 netinst, etc.) se placent dans **/var/lib/vz/template/iso** (ou via *local → ISO Images → Upload*) : elles restent disponibles sur le stockage `local` pour les installations manuelles. Le template cloud-init (étape 1) n'utilise pas l'ISO, mais l'image qcow2 Rocky GenericCloud, seule à contenir cloud-init préinstallé ; le DVD Rocky convient pour une installation manuelle classique.

Si `vmbr1` n'existe pas, créez-le : *app → Network → Create → Linux Bridge*, nom `vmbr1`, adresse `10.10.10.1/24`, aucun port, puis *Apply Configuration*. Ajoutez ensuite le NAT décrit dans la section « configuration, réseau et isolation ».

Dans les commandes ci-dessous, `local-lvm` est le pool de `sda` et `vmdata` celui de `sdb` (voir « Option C retenue »). Si `pvesm status` affiche d'autres noms, remplacez-les partout.

### Étape 1 : créer le template (une seule fois)

```bash
cd /var/lib/vz/template
wget https://download.rockylinux.org/pub/rocky/10/images/x86_64/Rocky-10-GenericCloud.latest.x86_64.qcow2

qm create 9000 --name rock-tpl --memory 2048 --cores 2 --cpu host --net0 virtio,bridge=vmbr1
qm importdisk 9000 Rocky-10-GenericCloud.latest.x86_64.qcow2 local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot order=scsi0 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1
qm template 9000
```

Attention au piège : si la VM 9000 existait déjà (essai précédent), `qm create` échoue, et `qm importdisk` crée le volume sous le nom `vm-9000-disk-1`, resté en `unused1`, pendant que `scsi0` pointe sur un volume inexistant. Dans ce cas, rattachez le vrai disque :

```bash
qm config 9000                     # repérez la ligne unused1: local-lvm:vm-9000-disk-1
qm set 9000 --delete scsi0
qm set 9000 --scsi0 local-lvm:vm-9000-disk-1
```

Vérification du template avant tout clone — `qm config 9000` doit montrer exactement :

```
scsi0: local-lvm:vm-9000-disk-1     (ou -disk-0, selon l'import)
ide2: local-lvm:vm-9000-cloudinit,media=cdrom
boot: order=scsi0
template: 1
```

Si `boot` reste sur `order=net0` ou si `scsi0`/`ide2` manquent, les clones n'auront ni disque ni source d'installation : corrigez d'abord le template.

Créez aussi la clé SSH d'administration, à utiliser pour toutes les VM :

```bash
ssh-keygen -t ed25519 -f /root/cle-admin -N ""
```

### Étape 2 : créer la VM HAProxy (ID 100)

```bash
qm destroy 100 --purge 2>/dev/null    # nettoie un éventuel essai précédent
qm clone 9000 100 --name haproxy --full
```

Le déroulé du clone doit lister **deux** volumes (le disque système et le cloudinit). Si seul le cloudinit apparaît, le template a perdu son disque : revenez à l'étape 1.

```bash
qm clone 9000 100 --name haproxy --full
qm set 100 --cores 2 --memory 2048 \
  --ipconfig0 ip=10.10.10.10/24,gw=10.10.10.1 \
  --nameserver 1.1.1.1 \
  --ciuser admin --sshkeys /root/cle-admin.pub
qm resize 100 scsi0 10G
qm set 100 --net0 virtio,bridge=vmbr1,firewall=1
qm set 100 --onboot 1
qm start 100
```

`--onboot 1` relance la VM automatiquement au redémarrage du serveur. Pour HAProxy c'est indispensable : sans elle, tous les sites sont coupés.

### Étape 3 : créer la VM DSI (ID 101)

```bash
qm clone 9000 101 --name dsi --full
qm set 101 --cores 4 --memory 8192 \
  --ipconfig0 ip=10.10.10.101/24,gw=10.10.10.1 \
  --nameserver 1.1.1.1 \
  --ciuser admin --sshkeys /root/cle-admin.pub
qm resize 101 scsi0 40G
qm set 101 --net0 virtio,bridge=vmbr1,firewall=1
qm set 101 --onboot 1
qm start 101
```

### Étape 4 : créer tsenamilay (ID 102) et UF2 (ID 103)

Créer la VM client `tsenamilay` :

```bash
qm clone 9000 102 --name tsenamilay --full --storage vmdata
qm set 102 --cores 4 --memory 8192 \
  --ipconfig0 ip=10.10.10.102/24,gw=10.10.10.1 \
  --nameserver 1.1.1.1 \
  --ciuser admin --sshkeys /root/cle-admin.pub
qm resize 102 scsi0 80G
qm set 102 --net0 virtio,bridge=vmbr1,firewall=1
qm set 102 --onboot 1
qm start 102
```

Créer la VM UF2 :

```bash
qm clone 9000 103 --name uf2 --full --storage vmdata
qm set 103 --cores 4 --memory 8192 \
  --ipconfig0 ip=10.10.10.103/24,gw=10.10.10.1 \
  --nameserver 1.1.1.1 \
  --ciuser admin --sshkeys /root/cle-admin.pub
qm resize 103 scsi0 40G
qm set 103 --net0 virtio,bridge=vmbr1,firewall=1
qm set 103 --onboot 1
qm start 103
```

### Étape 5 : pare-feu des VM

Sur Proxmox, activez d'abord le pare-feu au niveau Datacenter (*Datacenter → Firewall → Options → Firewall = Yes*). Puis créez un fichier par VM, avec l'éditeur de texte : `nano /etc/pve/firewall/100.fw` (et de la même façon pour les autres IDs).

Fichier à ouvrir avec `nano /etc/pve/firewall/100.fw` (HAProxy : web ouvert, SSH réservé à l'administration) :

```
[OPTIONS]
enable: 1
policy_in: DROP
macfilter: 1
ipfilter: 1

[RULES]
IN ACCEPT -p tcp -dport 80,443
IN ACCEPT -p tcp -dport 22 -source 10.10.10.1
```

Puis, fichiers `/etc/pve/firewall/101.fw`, `102.fw` et `103.fw` (DSI, tsenamilay et UF2 : visibles seulement par HAProxy et l'administration), chacun à créer avec `nano /etc/pve/firewall/101.fw` (102, 103) :

```
[OPTIONS]
enable: 1
policy_in: DROP
macfilter: 1
ipfilter: 1

[RULES]
IN ACCEPT -source 10.10.10.10 -p tcp -dport 80,443
IN ACCEPT -source 10.10.10.1 -p tcp -dport 22
```

La règle SSH autorise l'hôte Proxmox (10.10.10.1) seulement. Chaque VM refuse tout le reste, y compris les autres VM : DSI, tsenamilay et UF2 ne se voient donc pas entre elles. Si deux de ces VM doivent communiquer, ajoutez une règle précise sur la VM concernée, par exemple `IN ACCEPT -source 10.10.10.102 -p tcp -dport 5432`.

### Étape 6 : vérifier

```bash
qm list                      # les 4 VM doivent être en running
ssh -i /root/cle-admin admin@10.10.10.10   # connexion à HAProxy
ssh -i /root/cle-admin admin@10.10.10.101  # connexion à DSI
```

Depuis la VM `tsenamilay`, testez que l'isolation fonctionne : `ping 10.10.10.103` ne doit pas répondre.

### Étape 7 : installer HAProxy et brancher les domaines

Dans la VM HAProxy :

```bash
sudo dnf update -y && sudo dnf install -y haproxy
```

Puis configurez HAProxy avec le fichier de la section « Reverse proxy HAProxy » ci-dessous, adapté à ce plan :

| Domaine (exemple) | Backend | Serveur |
| --- | --- | --- |
| dsi.exemple.mg | be\_dsi | 10.10.10.101 |
| tsenamilay.exemple.mg | be\_tsenamilay | 10.10.10.102 |
| uf2.exemple.mg | be\_uf2 | 10.10.10.103 |

Remplacez les domaines d'exemple par les vôtres, créez les enregistrements DNS de type A vers votre IP publique, et redirigez les ports 80 et 443 de l'hôte vers 10.10.10.10 (voir la section HAProxy).

### Dépannage rapide

| Symptôme | Cause probable | Solution |
| --- | --- | --- |
| `storage 'local-lvm' does not exist` | Le stockage porte un autre nom | Lancer `pvesm status` et remplacer le nom |
| `disk 'scsi0' does not exist` au clone/resize | Le template a perdu son disque (`unused1` vu dans `qm config 9000`) | Rattacher : `qm set 9000 --delete scsi0` puis `qm set 9000 --scsi0 local-lvm:vm-9000-disk-1` |
| `bridge 'vmbr1' does not exist` au démarrage | Le bridge privé n'est pas créé | Créer `vmbr1` (voir étape 0) puis `ifreload -a` |
| Le clone ne liste que le volume cloudinit | Disque système absent du template | Corriger le template à l'étape 1 avant de re-cloner |
| La VM démarre mais pas de réseau | `vmbr1` absent ou NAT manquant | Vérifier `ip -br a` et la règle MASQUERADE |
| SSH refusé | Mauvaise clé ou pare-feu | Utiliser `-i /root/cle-admin` et vérifier le fichier `.fw` |
| `kvm: no such file` ou lenteur extrême | Virtualisation désactivée | Activer Virtualization Technology dans le BIOS |

## Reverse proxy HAProxy

HAProxy reçoit tout le trafic web de votre IP publique et l'envoie vers la bonne VM selon le nom de domaine demandé.

```
Internet → IP publique (ports 80 et 443) → hôte Proxmox (redirection) → VM HAProxy 10.10.10.10 → DSI / tsenamilay / UF2
```

### 1. Rediriger les ports 80 et 443 vers HAProxy

Sur l'hôte Proxmox, ouvrez `nano /etc/network/interfaces`, puis ajoutez ces lignes dans le bloc `vmbr1` pour qu'elles survivent au redémarrage :

```
    post-up   iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 80  -j DNAT --to 10.10.10.10:80
    post-up   iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to 10.10.10.10:443
    post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 80  -j DNAT --to 10.10.10.10:80
    post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to 10.10.10.10:443
```

Appliquez avec `ifreload -a`. Si votre IP publique est sur un routeur ou une box, redirigez d'abord les ports 80 et 443 de ce routeur vers l'IP de l'hôte (par exemple 192.168.88.2). Vérifiez aussi que votre fournisseur ne bloque pas ces ports et que vous n'êtes pas derrière un CGNAT, sinon les sites seront injoignables depuis Internet.

### 2. Installer HAProxy

```bash
sudo dnf update -y && sudo dnf install -y haproxy
```

### 3. Configurer HAProxy

Fichier à éditer avec `nano /etc/haproxy/haproxy.cfg`. Le port 80 est routé par nom de domaine, le port 443 par SNI sans déchiffrer le trafic : chaque VM garde son propre certificat.

```
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    timeout connect 5s
    timeout client  50s
    timeout server  50s

# ---------- HTTP (port 80) : routage par nom de domaine ----------
frontend http_in
    bind *:80
    acl host_dsi hdr(host) -i dsi.exemple.mg
    acl host_tsenamilay hdr(host) -i tsenamilay.exemple.mg
    acl host_uf2 hdr(host) -i uf2.exemple.mg
    use_backend be_dsi_http if host_dsi
    use_backend be_tsenamilay_http if host_tsenamilay
    use_backend be_uf2_http if host_uf2
    default_backend be_none

# ---------- HTTPS (port 443) : routage par SNI, sans déchiffrer ----------
frontend https_in
    bind *:443
    mode tcp
    option tcplog
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    acl sni_dsi req_ssl_sni -i dsi.exemple.mg
    acl sni_tsenamilay req_ssl_sni -i tsenamilay.exemple.mg
    acl sni_uf2 req_ssl_sni -i uf2.exemple.mg
    use_backend be_dsi_https if sni_dsi
    use_backend be_tsenamilay_https if sni_tsenamilay
    use_backend be_uf2_https if sni_uf2

# ---------- Backends ----------
backend be_none
    # aucun serveur : réponse 503 pour un domaine inconnu

backend be_dsi_http
    server dsi 10.10.10.101:80 check
backend be_dsi_https
    mode tcp
    server dsi 10.10.10.101:443 check

backend be_tsenamilay_http
    server tsenamilay 10.10.10.102:80 check
backend be_tsenamilay_https
    mode tcp
    server tsenamilay 10.10.10.102:443 check

backend be_uf2_http
    server uf2 10.10.10.103:80 check
backend be_uf2_https
    mode tcp
    server uf2 10.10.10.103:443 check
```

Vérifiez la syntaxe, puis appliquez :

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl enable --now haproxy
sudo systemctl reload haproxy
```

### 4. Configurer le DNS

Chez le registrar de chaque domaine, créez un enregistrement de type A qui pointe vers votre IP publique. La propagation prend de quelques minutes à quelques heures.

### 5. Tester

```bash
# Avant même le DNS, depuis l'extérieur
curl -I -H "Host: dsi.exemple.mg" http://VOTRE-IP-PUBLIQUE

# Après le DNS
curl -I https://dsi.exemple.mg
```

Un domaine inconnu doit retourner une erreur 503. Les journaux se lisent avec `journalctl -u haproxy -f`.

### Certificats HTTPS : deux approches

| Approche | Principe | Avantage | Limite |
| --- | --- | --- | --- |
| **Passthrough** (configuration ci-dessus) | HAProxy transmet le trafic chiffré ; chaque VM gère son certificat (certbot) | Confidentialité : HAProxy ne déchiffre rien | La VM voit l'IP de HAProxy, pas celle du visiteur |
| **Terminaison sur HAProxy** | HAProxy déchiffre et centralise les certificats Let's Encrypt | Un seul endroit à gérer, IP réelle du visiteur transmise | HAProxy voit tout le trafic ; un incident l'affecte plus fortement |

Pour débuter, gardez le passthrough. Passez à la terminaison sur HAProxy quand vous aurez beaucoup de domaines à gérer.

### Ajouter un nouveau domaine

1. Créez ou choisissez la VM de destination et notez son IP.
2. Ajoutez une ACL HTTP, une ACL SNI et les deux `use_backend` dans `haproxy.cfg`, puis les deux backends.
3. Autorisez 80 et 443 depuis `10.10.10.10` dans le pare-feu de cette VM.
4. Créez l'enregistrement DNS de type A.
5. Lancez `sudo haproxy -c -f /etc/haproxy/haproxy.cfg`, puis `sudo systemctl reload haproxy` : les connexions en cours ne sont pas coupées.

### Un seul IP public : tous les autres ports passent aussi par HAProxy

Les VM n'ont aucune IP publique : elles restent en 10.10.10.x. Tout ce qui doit être joignable depuis Internet (SSH, RDP, bases de données, autres services) passe par HAProxy, en mode `tcp`, avec **un port public par service**. Les ports 80 et 443 restent réservés au web, routé par nom de domaine.

Plan de ports conseillé :

| Service | Port public | Destination |
| --- | --- | --- |
| SSH de la VM DSI | 22101 | 10.10.10.101:22 |
| SSH de la VM tsenamilay | 22102 | 10.10.10.102:22 |
| SSH de la VM UF2 | 22103 | 10.10.10.103:22 |
| RDP éventuel d'une VM | 33101 | 10.10.10.101:3389 |

Éditez toujours le même fichier `nano /etc/haproxy/haproxy.cfg`, et ajoutez-y ces blocs (un frontend et un backend par service) :

```
frontend ssh_dsi
    bind *:22101
    mode tcp
    default_backend be_ssh_dsi
backend be_ssh_dsi
    mode tcp
    server dsi 10.10.10.101:22 check

frontend ssh_tsenamilay
    bind *:22102
    mode tcp
    default_backend be_ssh_tsenamilay
backend be_ssh_tsenamilay
    mode tcp
    server tsenamilay 10.10.10.102:22 check

frontend ssh_uf2
    bind *:22103
    mode tcp
    default_backend be_ssh_uf2
backend be_ssh_uf2
    mode tcp
    server uf2 10.10.10.103:22 check
```

Ouvrez ces ports sur l'hôte avec une règle de redirection par port, comme pour 80 et 443 :

```
    post-up   iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 22101:22103 -j DNAT --to 10.10.10.10
    post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 22101:22103 -j DNAT --to 10.10.10.10
```

Autorisez ensuite ces ports dans le pare-feu de la VM HAProxy (`100.fw`), par exemple `IN ACCEPT -p tcp -dport 22101:22103`.

Points d'attention :

- Un port exposé à Internet reçoit des tentatives de connexion permanentes : SSH par clés uniquement, `fail2ban` actif, et si possible une restriction par IP source.
- Pour les services sensibles (bases de données, RDP), préférez le VPN WireGuard plutôt qu'un port public.
- HAProxy devient le point d'entrée unique : sauvegardez sa configuration et surveillez sa disponibilité.

Sauvegardez la VM HAProxy et son fichier de configuration : si elle tombe, tous les sites tombent.

## Accès VPN avec WireGuard

FortiClient est lié à un équipement Fortinet payant. L'équivalent libre recommandé est WireGuard : rapide, simple et sûr, avec des clients gratuits pour Windows, macOS, Linux, Android et iOS.

Chaque client reçoit une clé et une adresse VPN, et le pare-feu l'autorise à joindre uniquement l'IP de sa VM. Les administrateurs ont un accès séparé, qui seul permet d'atteindre l'interface Proxmox (port 8006).

| Outil | Points forts | Limites |
| --- | --- | --- |
| **WireGuard** (recommandé) | Très rapide, configuration courte, sécurité moderne | Un fichier de clés par client à gérer, ou l'outil web wg-easy |
| OpenVPN | Très répandu, bien documenté | Plus lourd et plus lent à configurer |
| OPNsense ou pfSense en VM | Pare-feu et VPN avec interface web, proche d'un FortiGate | Plus de ressources et de configuration |

### 1. Installer WireGuard sur l'hôte Proxmox

Décision retenue : installer WireGuard **directement sur l'hôte** (Debian, donc `apt`). C'est la mise en place la plus simple ; si le trafic VPN devient important, migrer plus tard vers une VM ou un conteneur dédié. Le module noyau est présent dans les noyaux récents, le paquet fournit `wg` et `wg-quick`.

```bash
apt update && apt install wireguard -y
```

### 2. Générer les clés

Le `umask 077` doit être actif AVANT chaque génération : les clés privées doivent rester en permissions 600.

```bash
cd /etc/wireguard
umask 077
wg genkey | tee server.key | wg pubkey > server.pub
wg genkey | tee dsi.key | wg pubkey > dsi.pub       # préfixer chaque client par son nom
```

Conservez `server.key` en deux endroits : sur l'hôte bien sûr, et dans votre gestionnaire de mots de passe. Sans elle, tous les clients doivent être re-provisionnés en cas de perte.

### 3. Fichier /etc/wireguard/wg0.conf

Créez-le avec `nano /etc/wireguard/wg0.conf` :

```
[Interface]
Address = 10.20.20.1/24
ListenPort = 51820
PrivateKey = <contenu de server.key>

# Les règles d'isolation sont gérées ici : elles survivent au reboot sans iptables-persistent
PostUp = iptables -A FORWARD -i wg0 -s 10.20.20.11 -d 10.10.10.101 -j ACCEPT
PostUp = iptables -A FORWARD -i wg0 -j DROP

# Client DSI (VM 101)
[Peer]
PublicKey = <contenu de dsi.pub>
AllowedIPs = 10.20.20.11/32
```

Chaque nouveau client = une ligne `PostUp ACCEPT` placée **avant** le `DROP` final, plus un bloc `[Peer]`.

### 4. Routage permanent du noyau

```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-vpn.conf
sysctl -p /etc/sysctl.d/99-vpn.conf
```

⚠️ Pour plus tard : si le pare-feu Proxmox est activé au niveau Datacenter avec politique FORWARD en DROP, il faudra aussi autoriser le trafic `wg0` dans *Datacenter → Firewall*.

### 5. Activer le service et vérifier

```bash
systemctl enable --now wg-quick@wg0
wg show                  # listening port 51820, 0 peers tant qu'aucun client n'est actif
ip -br a | grep wg0      # wg0 UP 10.20.20.1/24
```

Ouvrez ensuite le port **51820/udp** vers `192.168.88.2` sur votre routeur, comme pour les 80/443 (attention au CGNAT chez certains fournisseurs).

### 6. Limiter chaque client à sa VM

Les règles `PostUp` du fichier `wg0.conf` font déjà le travail — c'est la méthode retenue, pas de `iptables-persistent` nécessaire. Le retour des paquets (VM → client) passe par conntrack et est accepté automatiquement.

### 7. Fichier à remettre au client

Utilisez le couple `dsi.key` / `dsi.pub` généré à l'étape 2, puis créez le fichier client — la rédaction se fait côté admin au brouillon `nano ~/fichiers-clients/dsi.conf` — et remettez-lui ce fichier (l'`Address` et l'`AllowedIPs` limitent ce qu'il peut atteindre, même s'il modifie le sien) :

```
[Interface]
PrivateKey = <clé privée du client>
Address = 10.20.20.11/32

[Peer]
PublicKey = <contenu de server.pub>
Endpoint = IP-PUBLIQUE-OU-DOMAINE:51820
AllowedIPs = 10.10.10.101/32
PersistentKeepalive = 25
```

Le client importe ce fichier dans l'application WireGuard, active le tunnel, puis se connecte en SSH ou en RDP à `10.10.10.101`.

### 8. Tester le tunnel et l'accès administrateur

```bash
# Côté client : import du .conf dans l'appli WireGuard, activation, puis
ssh -i /root/cle-admin admin@10.10.10.101

# Côté serveur : suivre le handshake et le trafic
wg show                                          # latest handshake, transfer RX/TX
journalctl -u wg-quick@wg0 -f
```

Un client sans handshake a probablement le port 51820/udp fermé sur le routeur, ou un `AllowedIPs` inversé dans l'un des deux fichiers.

Accès administrateur : même principe, avec des `AllowedIPs` contenant l'hôte et l'interface — clé séparée, jamais le fichier d'un client :

```
[Peer]
PublicKey = <clé publique du admin.key>
AllowedIPs = 10.20.20.10/32
```

Fichier admin correspondant :

```
[Interface]
PrivateKey = <clé privée du admin.key>
Address = 10.20.20.10/32

[Peer]
PublicKey = <contenu de server.pub>
Endpoint = IP-PUBLIQUE-OU-DOMAINE:51820
AllowedIPs = 10.20.20.0/24, 10.10.10.101/32, 192.168.88.2/32   # selon les VM légitimes
PersistentKeepalive = 25
```

Avec `AllowedIPs = ... , 192.168.88.2/32`, l'interface Proxmox `https://192.168.88.2:8006` n'est joignable que depuis ce tunnel : ne l'exposez jamais tel quel sur Internet.

### 9. Révoquer un accès

Supprimez son bloc `[Peer]` et sa ligne `PostUp` du `wg0.conf`, puis :

```bash
wg syncconf wg0 <(wg-quick strip wg0)    # sans coupure des autres peers
iptables -D FORWARD -i wg0 -s 10.20.20.11 -d 10.10.10.101 -j ACCEPT
```

### Bonnes pratiques VPN

- Une clé par personne, jamais partagée.
- Pour révoquer un accès, supprimez son bloc `[Peer]` puis appliquez `wg syncconf wg0 <(wg-quick strip wg0)`.
- Réservez l'accès à l'interface Proxmox (8006) et au SSH de l'hôte aux adresses VPN des administrateurs.
- Avec beaucoup de clients, utilisez wg-easy ou un script qui génère les fichiers et les règles.

## Plan d'action et checklist

Suivez ces étapes dans l'ordre, et validez l'isolation avec une VM de test avant d'accueillir le premier client.

- [ ] BIOS : Boot Mode UEFI et Virtualization Technology activée
- [ ] Passer sur le dépôt No-Subscription et mettre à jour
- [ ] Créer le pool `vmdata` sur `sdb` (LVM-thin, séparé du groupe `pve`)
- [ ] Préparer la machine de sauvegarde externe (PBS, NAS ou disque USB) et l'ajouter dans Datacenter → Storage
- [ ] Planifier les sauvegardes nocturnes et sauvegarder la configuration de l'hôte
- [ ] Créer un compte administrateur nominatif, activer la 2FA, restreindre l'accès à l'interface 8006
- [ ] Créer le réseau privé `vmbr1` avec NAT
- [ ] Activer le pare-feu au niveau Datacenter
- [ ] Créer le template cloud-init (ID 9000)
- [ ] Créer HAProxy (100) et DSI (101) sur `local-lvm`, tsenamilay (102) et UF2 (103) sur `vmdata`
- [ ] Appliquer les pare-feu par VM et vérifier que `tsenamilay` ne joint pas UF2
- [ ] Configurer HAProxy et le DNS, puis tester chaque domaine
- [ ] Installer WireGuard sur l'hôte (`/etc/wireguard`, réseau 10.20.20.0/24), créer un tunnel et un accès par personne
- [ ] Tester une restauration complète depuis la sauvegarde externe
- [ ] Écrire le plan de reprise pour `sda` et `sdb`, et prévoir un SSD de rechange
- [ ] Configurer les alertes email et la surveillance de `lvs` et SMART sur les deux disques
- [ ] Documenter l'inventaire : VM, IP, domaines, clés
- [ ] Accueillir le premier client

Avec l'option C, la capacité est d'environ 410 Go, mais sans redondance : la sauvegarde externe et le plan de reprise sont ce qui protège vos clients. La migration vers ZFS RAID1 reste possible plus tard.
