# Étape 3 : Création d'une Clé USB Bootable

L'objectif de cette étape est de transférer l'environnement Linux personnalisé (créé à l'**Étape 2**) sur une clé USB et de la rendre **amorçable** (bootable) à l'aide du bootloader **Syslinux**. Cette clé USB servira de support portable pour votre système GNU/Linux minimaliste.

---

## 3.1 Préparation de la Clé USB

### Pourquoi ?
Une clé USB bootable permet de démarrer votre système personnalisé sur n'importe quel matériel compatible UEFI. Le partitionnement et le formatage adéquats sont essentiels pour assurer la compatibilité et la fonctionnalité du système.

---

### 3.1.1 Détection du Périphérique

**Problème rencontré :**
Après avoir inséré la clé USB dans la machine virtuelle (via `VM > Removable Devices > Connect`), le système ne la reconnaissait pas.

**Diagnostic :**
Le contrôleur USB virtuel de VMware était configuré en mode **USB 2.0**, ce qui créait un conflit de pilotes avec le noyau Gentoo.

**Solution :**
1. Éteindre la VM.
2. Dans les paramètres de la VM (`VM > Settings > USB Controller`), changer la **`USB compatibility`** en **`USB 3.1`**.
3. Redémarrer la VM. La clé est désormais détectée en tant que `/dev/sdb` (vérifiable avec `lsblk`).

> **Note** : Cette étape est cruciale pour éviter les problèmes de détection matérielle dans un environnement virtuel.

---

### 3.1.2 Partitionnement et Formatage

**Étapes :**
1. **Lancer `fdisk`** pour partitionner la clé USB (`/dev/sdb`) en utilisant une table de partition **GPT** :
   ```bash
   fdisk /dev/sdb
   ```
   - **Séquence de commandes dans `fdisk`** :
     - `g` : Créer une nouvelle table de partition GPT.
     - `n` → `1` → `+200M` : Partition 1 (200 Mo, type **EFI System**).
     - `t` → `1` : Changer le type de la partition 1 en **EFI System**.
     - `n` → `2` → `+200M` : Partition 2 (200 Mo, type **Linux root (x86-64)**).
     - `t` → `2` → `24` : Changer le type de la partition 2 en **Linux root (x86-64)**.
     - `n` → `3` → (taille par défaut) : Partition 3 (reste de l'espace, type **Microsoft basic data**).
     - `t` → `3` → `11` : Changer le type de la partition 3 en **Microsoft basic data**.
     - `p` : Vérifier la structure.
     - `w` : Écrire les changements.

2. **Formater les partitions** avec les systèmes de fichiers appropriés :
   ```bash
   mkfs.fat -F 32 /dev/sdb1    # FAT32 pour la partition EFI
   mkfs.ext4 /dev/sdb2         # ext4 pour la partition racine
   mkfs.ntfs -f /dev/sdb3      # NTFS pour la partition de stockage (rapport, fichiers VMware)
   ```

**Structure finale :**
| Partition | Taille   | Type                  | Système de fichiers | Point de montage |
|-----------|----------|-----------------------|---------------------|------------------|
| `/dev/sdb1` | 200 Mo   | EFI System            | FAT32               | `/mnt/usb/bootpart` |
| `/dev/sdb2` | 200 Mo   | Linux root (x86-64)   | ext4                | `/mnt/usb/rootos`  |
| `/dev/sdb3` | Reste    | Microsoft basic data  | NTFS                | `/mnt/usb/data`    |

> **Remarque** : Le formatage en **NTFS** permet une compatibilité avec Windows pour le stockage de documents.

---

## 3.2 Installation du Bootloader Syslinux

### Pourquoi Syslinux ?
**Syslinux** est un bootloader léger et compatible UEFI, idéal pour les clés USB bootables. Il est plus simple à configurer que GRUB pour ce cas d'usage.

---

### 3.2.1 Montage des Partitions
```bash
mkdir -p /mnt/usb/bootpart /mnt/usb/rootos /mnt/usb/data
mount /dev/sdb1 /mnt/usb/bootpart
mount /dev/sdb2 /mnt/usb/rootos
mount /dev/sdb3 /mnt/usb/data
```

---

### 3.2.2 Copie des Fichiers de Syslinux

**Problème rencontré :**
Difficulté à localiser les fichiers `syslinux.efi`, `ldlinux.e64`, `ldlinux.c32`, et `ldlinux.sys` dans l'arborescence des sources extraites.

**Solution :**
Utiliser la commande `find` pour rechercher les fichiers :
```bash
find / -name "syslinux.efi" 2>/dev/null
find / -name "ldlinux.e64" 2>/dev/null
find / -name "ldlinux.c32" 2>/dev/null
find / -name "ldlinux.sys" 2>/dev/null
```

**Copie des fichiers :**
```bash
cp /chemin/vers/syslinux.efi /mnt/usb/bootpart/EFI/BOOT/bootx64.efi
cp /chemin/vers/ldlinux.e64 /mnt/usb/bootpart/EFI/BOOT/
cp /chemin/vers/ldlinux.c32 /mnt/usb/bootpart/EFI/BOOT/
cp /chemin/vers/ldlinux.sys /mnt/usb/bootpart/EFI/BOOT/
```

> **Explication** :
> - `syslinux.efi` est renommé en `bootx64.efi` pour être reconnu automatiquement par le firmware UEFI.
> - Les fichiers `ldlinux.*` sont nécessaires pour le fonctionnement de Syslinux.

---

### 3.2.3 Configuration de Syslinux

**Création du fichier `syslinux.cfg`** (`/mnt/usb/bootpart/EFI/BOOT/syslinux.cfg`) :
```plaintext
PROMPT 0
TIMEOUT 10
DEFAULT NGOUMOU
LABEL NGOUMOU
    LINUX vmlinuz
    INITRD initrd
    APPEND root=UUID=<UUID_de_sdb2> ro
```

**Obtenir l'UUID de `/dev/sdb2`** :
```bash
blkid /dev/sdb2
```

> **Exemple de sortie** :
> `/dev/sdb2: UUID="1234-abcd" TYPE="ext4"`
> Remplacez `<UUID_de_sdb2>` par l'UUID réel de votre partition.

---

## 3.3 Transfert de l'OS Personnalisé

### 3.3.1 Copie du Système
```bash
cp -a /mnt/monlinux/* /mnt/usb/rootos/
```

> **Option `-a`** : Conserve les permissions, la propriété, et les liens symboliques.

---

### 3.3.2 Problème de Démarrage : "Kernel not found"

**Symptôme :**
Au démarrage, Syslinux affichait une erreur indiquant qu'il ne trouvait pas le fichier `vmlinuz`.

**Diagnostic :**
Incohérence entre les noms des fichiers du noyau (`vmlinuz-6.2.0-32-generic`) et ceux spécifiés dans `syslinux.cfg` (`vmlinuz`).

**Solution :**
1. **Renommer les fichiers** sur la partition EFI :
   ```bash
   cd /mnt/usb/bootpart/EFI/BOOT/
   mv vmlinuz-6.2.0-32-generic vmlinuz
   mv initrd.img-6.2.0-32-generic initrd
   ```
2. **Alternative** : Modifier `syslinux.cfg` pour utiliser les noms complets des fichiers.

> **Leçon apprise** :
> Les noms de fichiers dans les configurations de boot doivent **correspondre exactement** aux noms réels.

---

## 3.4 Tests et Validation

### 3.4.1 Démarrage de la Clé USB
1. Redémarrer le PC et sélectionner la clé USB comme périphérique de démarrage (via le menu UEFI/BIOS).
2. Syslinux se lance et charge le noyau et l'initrd.
3. Le noyau monte la partition racine (`/dev/sdb2`) et démarre l'OS personnalisé.

### 3.4.2 Vérification du Système
- **Réseau** : Utiliser `udhcpc` pour obtenir une adresse IP.
- **Connectivité** : Tester avec `wget` ou `ftpget`.
- **Portabilité** : Essayer de démarrer sur plusieurs machines différentes.

---

## 3.5 Leçons Apprises

- **Configuration matérielle** : L'importance de bien configurer les contrôleurs USB dans une VM.
- **Diagnostic** : Utiliser `find`, `lsblk`, et `dmesg` pour résoudre les problèmes de détection et de chemins.
- **Rigueur** : Les noms de fichiers et les UUID doivent être **exacts** dans les configurations de boot.
- **Compatibilité** : Syslinux est une solution légère et efficace pour les clés USB bootables en mode UEFI.

---

## 3.6 Résultat Final

La clé USB est désormais **bootable** et contient :
- Un **système Linux minimaliste** fonctionnel.
- Une partition de stockage pour les rapports et fichiers VMware.
- Un bootloader **Syslinux** configuré pour démarrer en mode UEFI.

> **Félicitations !** Votre système personnalisé est désormais portable et peut être démarré sur n'importe quel matériel compatible UEFI.

---

### Commandes Utiles pour le Débogage
| Problème                | Commande de Diagnostic                     |
|-------------------------|---------------------------------------------|
| Détection de la clé USB | `lsblk`, `dmesg \| grep usb`                |
| Recherche de fichiers   | `find / -name "fichier" 2>/dev/null`        |
| UUID des partitions     | `blkid`                                     |
| Montage des partitions  | `mount \| grep sdb`                         |

