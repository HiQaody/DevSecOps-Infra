# Demande au client DSI : formulaire d'accès SSH

Oct 26, 2026 · @Randria

## Objectif unique

Rendre `ssh admin@10.10.10.101` accessible depuis la machine locale du DSI **via le tunnel WireGuard** — rien d'autre. Aucune couche applicative n'est interrogée (pas de stack, pas de base de données, pas de web) : le formulaire ne sert qu'à construire le tunnel et l'accès SSH.

Les cases destinées au client sont repérées par le symbole ✍. Réponse archivée avec l'inventaire (`iq-101-dsi-formulaire-20261026.pdf` par exemple).

## 1. Localisation (la question qui conditionne tout)

| ✍ Question | Sert à | Réponse du client |
| --- | --- | --- |
| Votre poste est-il sur le même réseau local que le serveur ? (oui / non) | `Endpoint` du fichier de tunnel : `192.168.88.2:51820` en LAN, ou IP publique:51820 à distance | ✍ |
| Si distant : votre fournisseur d'accès bloque-t-il des ports ? | Vérifier avant remise que 51820/udp reste joignable | ✍ |
| Combien de personnes utiliseront le tunnel ? (1 clé par personne) | Un bloc `[Peer]` et une ligne `PostUp` par personne | ✍ |

**Pourquoi on le demande** : sans nom de domaine, l'`Endpoint` est une adresse IP fixe. Si elle change, les fichiers clients doivent être corrigés — mieux vaut le savoir avant.

## 2. Accès SSH

| ✍ Question | Sert à | Réponse du client |
| --- | --- | --- |
| Nom d'utilisateur souhaité (défaut : `admin`) | Option `--ciuser` du provisioning | ✍ |
| **Clé publique SSH** de chaque personne (`ssh-ed25519 ...` ou `ssh-rsa ...`) | Injectée dans la VM, archivée dans `/root/cles-clients/dsi.pub` | ✍ |
| Contacts : 1 responsable + 1 suppléant (email, téléphone) | Escalade en cas d'incident | ✍ |

⚠️ **Rappel au client** : transmettez seulement la clé **publique** (`.pub`). La clé privée ne quitte jamais votre poste. Le fichier de tunnel vous est remis par nos soins, jamais l'inverse.

## 3. Fenêtres de maintenance

| ✍ Question | Sert à | Réponse du client |
| --- | --- | --- |
| Heures jamais à empêcher (ex. « jamais 8h–17h ») | Planification des snapshots et redémarrages de la VM | ✍ |

## Section de l'administrateur

### Ce que déclenche le formulaire

| Donnée reçue | Action |
| --- | --- |
| LAN / distant | `Endpoint` du fichier de tunnel (redirection routeur 51820/udp si distant) |
| Nombre de personnes | `[Peer]` + `PostUp` dans `wg0.conf` par personne, puis `wg syncconf wg0 <(wg-quick strip wg0)` |
| Clé publique SSH | `qm set 101 --ciuser <user> --sshkeys /root/cles-clients/dsi.pub` |
| Heures interdites | Calendrier des snapshots / redémarrages |

### Ordre des phases

1. Vérifier que tous les champs sont remplis — rappeler le client sur les manquants, ne pas deviner.
2. Créer/valider la VM 101 (IP `10.10.10.101/24`, pare-feu : SSH réservé à l'hôte `10.10.10.1`).
3. Déclarer le(s) `[Peer]` dans `wg0.conf`, appliquer `wg syncconf`.
4. **Test d'isolement obligatoire** : depuis une machine extérieure au tunnel, `ssh admin@10.10.10.101` doit échouer ; tunnel activé, la connexion doit réussir.
5. Remettre le fichier de tunnel (canal chiffré ou QR en personne) et la documentation client.
6. Archiver le formulaire rempli.

## Politique de remise

- Clé privée : jamais transmise par qui que ce soit, dans aucun sens.
- Fichier de tunnel : canal chiffré, en personne, ou QR généré localement — jamais d'email en clair.
- Formulaire relu par une seconde personne ; la localisation LAN/distant est l'erreur la plus coûteuse de ce formulaire.
