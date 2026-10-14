# serveur-linux-securise

Mise en place et sécurisation (hardening) d'un serveur **Ubuntu Server** dans une machine virtuelle **VirtualBox**, administré depuis un poste **Ubuntu** : connexion SSH par clé uniquement, accès root interdit, compte à privilèges limités, pare-feu UFW.

## Objectif

Partir d'un serveur Linux vierge et appliquer les bonnes pratiques de base pour réduire sa surface d'attaque :

- ne pas travailler en root, mais avec un compte administrateur (sudo)
- appliquer le principe du moindre privilège (un compte sans aucun droit d'administration)
- supprimer l'authentification par mot de passe en SSH (anti brute-force)
- interdire la connexion directe de root
- filtrer le trafic réseau avec un pare-feu

## Environnement

| Élément | Valeur |
|---|---|
| Hyperviseur | Oracle VirtualBox |
| Système invité | Ubuntu Server 24.04.5 LTS (noyau 6.8.0-146-generic) |
| Nom d'hôte | `alexserver` |
| Mémoire vive | 2048 Mo |
| Disque | 25 Gio (`ubuntu-server.vdi`), volume racine `/` de 11,21 Go |
| Réseau | NAT, avec redirection de port `127.0.0.1:2222` vers `10.0.2.15:22` |
| Client | Terminal Ubuntu, OpenSSH |

## Étapes réalisées

### 1. La machine virtuelle

Création de la VM `ubuntu-server` dans VirtualBox à partir de l'ISO `ubuntu-24.04.5-live-server-amd64.iso`.

![Assistant de création de la VM](captures/01-creation-vm.png)

Installation d'Ubuntu Server avec le paquet **OpenSSH server**. Caractéristiques de la VM : 2048 Mo de mémoire vive, disque de 25 Gio, une carte réseau en **NAT**.

![Détails de la VM](captures/02-vm-systeme.png)

Le mode NAT est complété par une règle de redirection de port : le port 2222 de l'hôte (`127.0.0.1`) est envoyé vers le port 22 de la VM (`10.0.2.15`). Comme la règle n'écoute que sur `127.0.0.1`, seul le PC hôte peut atteindre le SSH de la VM.

### 2. Connexion SSH par clé

Génération de la paire de clés sur le poste hôte (la clé privée ne quitte jamais le PC), protégée par une passphrase :

```bash
ssh-keygen -t ed25519 -C "tp-server"
```

![Génération de la clé](captures/03-cle-ssh.png)

Envoi de la clé publique vers le serveur :

```bash
ssh-copy-id -p 2222 alex@127.0.0.1
```

Résultat : `Number of key(s) added: 1`. Connexion avec la clé (le serveur demande la passphrase de la clé, pas le mot de passe du compte) :

```bash
ssh -p 2222 alex@127.0.0.1
```

![Copie de la clé et connexion](captures/04-connexion-ssh.png)

### 3. Le système

À la connexion, le bandeau affiche l'état du serveur : disque, mémoire, adresse réseau `10.0.2.15` (interface `enp0s3`) et mises à jour disponibles.

![Informations système](captures/05-systeme.png)

### 4. Utilisateur non-root et compte limité

Le travail se fait avec le compte `alex`, un utilisateur normal membre du groupe `sudo`. Les droits d'administrateur ne sont utilisés que ponctuellement, avec `sudo`.

Création d'un second compte, `stagiaire`, sans aucun droit d'administration :

```bash
whoami
id
sudo whoami
sudo adduser stagiaire
id stagiaire
su - stagiaire
sudo apt update
```

| Compte | Rôle | Droits |
|---|---|---|
| `alex` | Administrateur | Membre du groupe `sudo` |
| `stagiaire` | Compte limité | Aucun droit `sudo` |

Test du moindre privilège : `stagiaire` tente `sudo apt update` et est refusé (`stagiaire is not in the sudoers file`).

![Utilisateurs et droits](captures/06-utilisateurs.png)

Vérification depuis le compte administrateur :

```bash
sudo -l -U stagiaire
```

Résultat : `User stagiaire is not allowed to run sudo on alexserver`.

![stagiaire sans sudo](captures/07-stagiaire-sans-sudo.png)

### 5. Durcissement de SSH

Le fichier principal `sshd_config` ne chargeait pas le dossier `sshd_config.d/`. Ajout de la ligne suivante en début de fichier :

```
Include /etc/ssh/sshd_config.d/*.conf
```

Fichier dédié `/etc/ssh/sshd_config.d/00-hardening.conf` :

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Validation de la syntaxe (aucune sortie = pas d'erreur), redémarrage, puis vérification de la configuration réellement appliquée :

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin'
```

Résultat : `permitrootlogin no` et `passwordauthentication no`.

![Durcissement SSH](captures/08-durcissement-ssh.png)

### 6. Tests de refus

Tests réalisés depuis le poste hôte, après le durcissement :

| Test | Résultat attendu | Résultat |
|---|---|---|
| Connexion `root` | refusée | `Permission denied (publickey)` |
| Mot de passe seul (`-o PubkeyAuthentication=no`) | refusé | `Permission denied (publickey)` |

```bash
ssh -p 2222 root@127.0.0.1
ssh -p 2222 -o PubkeyAuthentication=no alex@127.0.0.1
```

La connexion de `alex` avec sa clé reste acceptée (voir l'étape 2).

![Tests de refus](captures/09-tests-refus.png)

### 7. Pare-feu UFW

Politique par défaut : tout le trafic entrant est bloqué, le trafic sortant est autorisé. Seul SSH est ouvert. La règle SSH doit exister **avant** l'activation du pare-feu, sinon on se coupe l'accès au serveur.

```bash
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

Résultat : `Status: active`, `Default: deny (incoming), allow (outgoing)` et `22/tcp ALLOW IN Anywhere` (IPv4 et IPv6).

![Pare-feu UFW](captures/10-ufw.png)

## Choix de sécurité

- **Clé ed25519 + passphrase** : sans la clé privée et sa passphrase, aucune connexion n'est possible.
- **Mot de passe SSH désactivé** : supprime l'attaque par brute-force sur les mots de passe.
- **Root interdit en SSH** : root est la cible n°1 des attaquants, l'administration passe par `sudo`.
- **NAT + redirection limitée à `127.0.0.1`** : le SSH de la VM n'est joignable que depuis le PC hôte.
- **Configuration dans `sshd_config.d/`** : fichier séparé, chargé en priorité.
- **Moindre privilège** : un compte sans `sudo` pour les usages limités.
- **Pare-feu** : politique par défaut restrictive, seul le port 22 est ouvert.

## Difficultés rencontrées

- **`Connection refused` sur le port 2222** : la VM était éteinte, ou la règle de redirection n'était pas validée dans VirtualBox.
- **Configuration SSH non prise en compte** : il manquait la ligne `Include` dans `sshd_config`.
- **Confusion entre les deux machines** : les commandes `ssh` se tapent sur le PC hôte, pas dans la VM.

## Ce que j'ai appris

- Gérer des utilisateurs et les droits `sudo`
- Générer et déployer une clé SSH, et vérifier ce que le serveur applique réellement (`sshd -T`)
- Configurer un pare-feu dans le bon ordre pour ne pas se verrouiller dehors
- Savoir sur quelle machine on tape chaque commande (poste hôte ou serveur)

## Pistes d'amélioration

- Retirer `alex` du groupe `lxd` : on le voit dans la sortie de `id`, et ce groupe est considéré comme équivalent à root
- Appliquer les 17 mises à jour disponibles (`apt update`, `apt upgrade`)
- Utiliser `ufw limit` pour limiter les tentatives de connexion
- Installer `fail2ban`
- Mettre en place la surveillance des connexions (`journalctl`, `lastlog`)

## Sécurité du dépôt

Ce dépôt ne contient aucune clé privée ni aucun mot de passe. Le fichier `.gitignore` bloque les clés privées courantes, et les captures d'écran ont été relues avant publication.
