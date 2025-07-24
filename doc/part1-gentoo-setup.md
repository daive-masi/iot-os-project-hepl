# Étape 1 : Préparation de l'environnement Gentoo

Cette étape décrit l'installation d'un système de base Gentoo qui servira d'environnement de développement pour la suite du projet.

## 1.1 Pré-requis

- **Logiciel** : VMware Workstation/Player
- **ISO** : `install-amd64-minimal-2024xxxx.iso`
- **Stage 3** : `stage3-amd64-openrc-2024xxxx.tar.xz`

## 1.2 Partitionnement du disque

Le disque `/dev/sda` est partitionné avec `fdisk` en utilisant une table GPT.

### Commandes de partitionnement

```bash
# Lancer l'utilitaire de partitionnement
fdisk /dev/sda
```

**Séquence des commandes dans fdisk :**

- `g` : Créer une nouvelle table de partition GPT.
- `n` -> `1` -> (default) -> `+512M` : Partition EFI.
- `t` -> `1` -> `1` : Changer le type en EFI System.
- `n` -> `2` -> (default) -> `+4G` : Partition Swap.
- `t` -> `2` -> `19` : Changer le type en Linux Swap.
- `w` : Écrire les changements.

**Concept Théorique** : Le partitionnement divise un disque physique en plusieurs disques logiques. GPT est un standard moderne qui remplace MBR, permettant plus de partitions et de plus grands disques.

## 1.3 Formatage et Montage

### Commandes de formatage et de montage

```bash
# Formatage
mkfs.fat -F 32 /dev/sda1
mkfs.ext4 /dev/sda3
mkswap /dev/sda2
swapon /dev/sda2

# Montage
mount /dev/sda3 /mnt/gentoo
```

## 1.4 Configuration de `make.conf`

Ce fichier est crucial, il définit les options de compilation pour tout le système.

### Édition de `make.conf`

```bash
nano /mnt/gentoo/etc/portage/make.conf
```

**Contenu :**

```ini
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
MAKEOPTS="-j5"
ACCEPT_LICENSE="*"
```

> **Note** : J'ai rencontré une erreur `emake failed` car j'avais initialement mis `MAKEOPTS="-jX"`. **Leçon apprise** : Toujours remplacer les variables de template par des valeurs concrètes.
