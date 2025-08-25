# Étape 2 : Création d'un OS Linux entièrement à partir des sources

Cette étape consiste à construire un système d'exploitation Linux minimaliste **entièrement à partir des sources**, sans utiliser de gestionnaire de paquets. L'objectif est de comprendre le processus de compilation et d'installation manuelle des composants essentiels d'un OS : la **glibc**, le **noyau Linux**, et **BusyBox**. Le système sera installé sur la partition `/dev/sda5`, en respectant la hiérarchie standard des systèmes de fichiers (FHS).

---

## I. Préparation des sources

### Pourquoi ?
Un système d'exploitation est composé de plusieurs couches logicielles. Pour le construire à partir des sources, il est nécessaire de télécharger et compiler manuellement chaque composant. Cela permet de comprendre leur rôle et leur interaction.

### Étapes :
1. **Télécharger les sources** dans `/usr/src` (sur `/dev/sda4`) :
   - **glibc** : Bibliothèque C standard (version compatible avec votre VM Gentoo, vérifiable via `ldd --version`).
     Téléchargement : [http://ftp.gnu.org/gnu/libc/glibc-2.XX.tar.gz](http://ftp.gnu.org/gnu/libc/glibc-2.XX.tar.gz)
   - **Noyau Linux** : Cœur du système d'exploitation.
     Téléchargement : [https://www.kernel.org/pub/linux/kernel](https://www.kernel.org/pub/linux/kernel)
   - **BusyBox** : Collection d'utilitaires Unix légers (remplace les commandes de base comme `ls`, `grep`, `mount`, etc.).
     Téléchargement : [https://busybox.net/downloads](https://busybox.net/downloads)

> **Note** : Les versions précises de chaque composant vous ont été communiquées lors du cours théorique.

---

## II. Création du système de fichiers (FHS)

### Pourquoi ?
Le **Filesystem Hierarchy Standard (FHS)** définit la structure des répertoires et leur contenu dans les systèmes Unix. Respecter cette norme assure la compatibilité et la cohérence du système.

### Étapes :
1. **Formater et monter `/dev/sda5`** :
   ```bash
   mkfs.ext4 /dev/sda5          # Créer un système de fichiers ext4
   mount /dev/sda5 /mnt/monlinux # Monter la partition
   ```

2. **Créer l'arborescence FHS** dans `/mnt/monlinux` :
   ```bash
   mkdir -p /mnt/monlinux/{bin,boot,dev,etc,home,lib64,mnt,proc,root,sbin,sys,tmp,usr,var}
   mkdir -p /mnt/monlinux/usr/{bin,include,lib64,local,sbin,share,src}
   mkdir -p /mnt/monlinux/usr/share/man/{man1,man2,...,man8}
   mkdir -p /mnt/monlinux/var/{lock,log,run,spool}
   ```

3. **Créer un lien symbolique pour les pages de manuel** :
   ```bash
   ln -s share/man /mnt/monlinux/usr/man
   ```

---

## III. Compilation et installation de la glibc

### Pourquoi ?
La **glibc** est la bibliothèque C standard utilisée par presque tous les programmes Linux. Elle fournit les fonctions de base pour les entrées/sorties, la gestion de la mémoire, etc.

### Étapes :
1. **Extraire les sources** :
   ```bash
   tar xzf glibc-2.XX.tar.gz -C /usr/src
   ```

2. **Configurer et compiler** :
   ```bash
   mkdir glibc-build && cd glibc-build
   CFLAGS="-O2 -U_FORTIFY_SOURCE -march=x86-64 -pipe" ../glibc-2.XX/configure --enable-addons --prefix=/usr --with-headers=/usr/include
   make
   ```

3. **Installer dans le nouvel environnement** :
   ```bash
   make install_root=/mnt/monlinux install
   ```

> **Explication des flags** :
> - `--prefix=/usr` : Installe la glibc dans `/usr`.
> - `--with-headers=/usr/include` : Spécifie l'emplacement des en-têtes du noyau.
> - `CFLAGS` : Optimise la compilation pour l'architecture x86-64.

---

## IV. Compilation et installation de BusyBox

### Pourquoi ?
**BusyBox** regroupe des centaines d'utilitaires Unix en un seul binaire léger, idéal pour les systèmes embarqués ou minimaux.

### Étapes :
1. **Extraire et configurer** :
   ```bash
   tar xzf busybox-XX.X.tar.bz2 -C /usr/src
   cd busybox-XX.X
   make menuconfig
   ```
   > **Conseil** : Désactivez les utilitaires qui génèrent des erreurs de compilation.

2. **Compiler et installer** :
   ```bash
   make
   make CONFIG_PREFIX=/mnt/monlinux install
   ```

---

## V. Configuration des fichiers système

### Pourquoi ?
Les fichiers de configuration définissent le comportement du système au démarrage (montage des partitions, réseau, utilisateurs, etc.).

### Étapes :
1. **Créer `/mnt/monlinux/etc/passwd`** :
   ```plaintext
   root::0:0\:root:/root:/bin/ash
   ```

2. **Créer `/mnt/monlinux/etc/fstab`** :
   ```plaintext
   /proc /proc proc defaults 0 0
   ```

3. **Sauvegarder la disposition du clavier** :
   ```bash
   busybox dumpkmap > /mnt/monlinux/etc/monclavier.kmap
   ```

4. **Créer un script de démarrage `/mnt/monlinux/etc/rc`** (exécutable) :
   ```bash
   #!/bin/ash
   /bin/mount -av
   /bin/hostname NOMDEFAMILLE
   /sbin/loadkmap < /etc/monclavier.kmap
   ```
   Rendre le script exécutable :
   ```bash
   chmod +x /mnt/monlinux/etc/rc
   ```

5. **Créer `/mnt/monlinux/etc/inittab`** :
   ```plaintext
   ::sysinit:/etc/rc
   tty1::respawn:/bin/ash -l
   ```

---

## VI. Compilation du noyau Linux

### Pourquoi ?
Le **noyau** est le cœur du système. Il gère les ressources matérielles (CPU, mémoire, périphériques) et permet l'exécution des programmes.

### Étapes :
1. **Extraire et configurer** :
   ```bash
   tar xzf linux-XX.tar.xz -C /usr/src
   cd linux-XX
   make defconfig  # Génère une configuration par défaut
   make menuconfig # Personnaliser si nécessaire
   ```

2. **Compiler et installer** :
   ```bash
   make
   cp arch/x86/boot/bzImage /mnt/monlinux/boot/kernel-XX-NOMDEFAMILLE
   ```

> **Note** : Le nom du noyau doit inclure la version complète et votre nom de famille (ex: `kernel-5.15.88-DUPONT`).

---

## VII. Configuration de GRUB

### Pourquoi ?
**GRUB** est le chargeur d'amorçage qui permet de démarrer le noyau Linux. Sans GRUB, le système ne peut pas démarrer.

### Étapes :
1. **Mettre à jour la configuration de GRUB** :
   ```bash
   grub-mkconfig -o /boot/grub/grub.cfg
   ```

2. **Modifier `/boot/grub/grub.cfg`** pour démarrer sur `/dev/sda5` :
   ```plaintext
   menuentry "Mon OS Linux" {
       linux /kernel-XX-NOMDEFAMILLE root=/dev/sda5 console=tty0 console=ttyS1 rootfstype=ext4 ro
   }
   ```

---

## VIII. Explications supplémentaires

### Pourquoi compiler manuellement ?
- **Contrôle total** : Vous choisissez exactement ce qui est inclus dans votre système.
- **Optimisation** : Le système est compilé spécifiquement pour votre matériel.
- **Apprentissage** : Comprendre le fonctionnement interne d'un OS.

### Points d'attention :
- **Dépendances** : Assurez-vous que chaque composant est compilé dans le bon ordre (glibc → BusyBox → noyau).
- **Débogage** : En cas d'erreur, vérifiez les logs de compilation et les permissions des fichiers.
- **Compatibilité** : Utilisez des versions compatibles entre elles (glibc, noyau, BusyBox).

---

## IX. Résultat final

Après ces étapes, vous devriez avoir :
- Un système Linux minimaliste fonctionnel sur `/dev/sda5`.
- Un noyau personnalisé et optimisé.
- Un ensemble d'utilitaires de base fournis par BusyBox.
- Un système respectant la norme FHS.

> **Félicitations !** Vous avez construit un OS Linux à partir des sources. Ce processus illustre le fonctionnement interne des systèmes d'exploitation et renforce vos compétences en administration système.



# Annexe - Étape 2 : Débogage du Processus de Démarrage du Noyau

La création d'un système d'exploitation à partir des sources est un processus complexe. Après avoir compilé et installé le noyau personnalisé, plusieurs problèmes de démarrage sont apparus. Cette section documente le processus itératif de diagnostic et de résolution qui a mené à un système entièrement fonctionnel.

---

## Problème 1 : Écran noir après GRUB (Pas de menu de démarrage)

**Symptôme :**
Après le redémarrage de la VM, l'écran restait noir, sans même afficher le menu de sélection de GRUB. En démarrant avec l'ISO d'installation, on constatait que c'était le GRUB du LiveCD qui s'affichait.

**Diagnostic :**
La machine virtuelle était configurée pour démarrer en mode **Legacy BIOS**, alors que l'installation de GRUB, effectuée depuis l'environnement UEFI du LiveCD, avait créé un chargeur d'amorçage **UEFI**. La VM ne trouvait donc pas le bon chargeur.

**Solution :**
1. Éteindre la machine virtuelle.
2. Dans les paramètres de la VM (`Settings > Options > Advanced`), changer le **Firmware Type** de `BIOS` à `UEFI`.
3. Dans les paramètres du `CD/DVD`, **déconnecter l'image ISO** pour forcer la VM à démarrer sur le disque dur virtuel.

**Résultat :**
Le menu GRUB installé sur le disque dur s'est affiché correctement.

---

## Problème 2 : Écran noir après la sélection du noyau dans GRUB

**Symptôme :**
Après avoir sélectionné le noyau personnalisé dans le menu GRUB, l'écran affichait "Loading Linux..." puis devenait noir et se figeait.

**Diagnostic :**
Le noyau démarrait mais échouait lors de l'initialisation de la console graphique. Le pilote **Framebuffer Console (`fbcon`)**, activé par défaut, est connu pour causer des problèmes dans certains environnements de virtualisation.

**Solution :**
1. Démarrer sur le LiveCD et `chrooter` dans l'installation.
2. Lancer la configuration du noyau : `cd /usr/src/linux && make menuconfig`.
3. Naviguer vers `Device Drivers ---> Graphics support ---> Console display driver support --->`.
4. **Désactiver** l'option `[*] Framebuffer Console Support`.
5. **Vérifier** que `[*] Legacy VGA text console` était bien activé.
6. Recompiler le noyau, le copier dans `/boot`, et mettre à jour GRUB avec `grub-mkconfig`.

**Résultat :**
Le noyau ne tentait plus d'initialiser une console graphique. Le démarrage affichait désormais des messages texte, mais se terminait par un `Kernel Panic`.

---

## Problème 3 : Kernel Panic - "Unable to mount root fs"

**Symptôme :**
Le noyau démarrait mais s'arrêtait avec une erreur fatale : `Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)`.

**Diagnostic :**
Le message d'erreur était clair : le noyau ne trouvait pas la partition racine. Cela indiquait que le **pilote matériel pour le contrôleur de disque** (SATA/AHCI ou SCSI pour VMware) n'était pas inclus dans le noyau, ou pas de la bonne manière.

**Solution :**
1. Retourner dans `make menuconfig` via le chroot.
2. S'assurer que les pilotes essentiels étaient compilés **en dur (`*`)** dans le noyau. Après plusieurs essais, la méthode la plus robuste a été d'utiliser un **initramfs**.
3. Installer l'outil de génération d'initramfs : `emerge sys-kernel/dracut`.
4. Compiler les pilotes de disque (`AHCI S_ATA support`, `VMware PVSCSI driver`) en tant que **modules (`<M>`)**.
5. Générer un initramfs en forçant l'inclusion de ces pilotes :
   ```bash
   dracut --force --force-drivers "ahci vmw_pvscsi" /boot/initramfs-mon-os.img <version-kernel>
   ```
6. Mettre à jour `/boot/grub/grub.cfg` pour charger cet initramfs avec la directive `initrd`.

**Résultat :**
Le `Kernel Panic` a disparu, mais le processus de démarrage s'arrêtait dans le shell de débogage de `dracut`.

---

## Problème 4 : Erreur Dracut - "UUID does not exist" et Shell de Débogage

**Symptôme :**
L'initramfs se chargeait, mais affichait une erreur `dracut Warning: /dev/disk/by-uuid/XXXXXXXX does not exist` et ouvrait un shell de débogage.

**Diagnostic :**
Le pilote de disque était maintenant bien chargé (la preuve est que `dracut` pouvait chercher des disques), mais l'UUID de la partition racine spécifié dans `grub.cfg` était incorrect. Dans le shell de débogage, la commande `ls -l /dev/sd*` a renvoyé "No such file or directory", confirmant que même l'initramfs n'avait pas le bon pilote.

**Solution (Itération Finale) :**
1. Retourner dans `make menuconfig` pour compiler les pilotes de disque en tant que **modules `<M>`**.
2. Lancer `make modules_install` pour installer les fichiers `.ko`.
3. Régénérer l'initramfs en forçant l'inclusion des pilotes avec `dracut --force --force-drivers "ahci vmw_pvscsi" ...`.
4. Modifier `/boot/grub/grub.cfg` pour utiliser un identifiant de partition plus simple et fiable : `root=/dev/sda5`.
5. Modifier `/etc/default/grub` en ajoutant `GRUB_DISABLE_LINUX_UUID=true` pour rendre ce changement permanent.

---

## Problème 5 : Pas de prompt après le démarrage réseau

**Symptôme :**
Le système démarrait, obtenait une adresse IP via DHCP, mais le prompt du shell n'apparaissait jamais.

**Diagnostic :**
Le script de démarrage `/etc/rc` lançait le client DHCP `udhcpc` en **premier plan**. Comme `udhcpc` est un processus qui tourne en continu, il bloquait l'exécution du script `rc`. Le processus `init` attendait donc indéfiniment que `/etc/rc` se termine et ne lançait jamais le shell de login.

**Solution :**
1. Modifier `/etc/rc`.
2. Changer la ligne de commande `/sbin/udhcpc -f -i "$INTERFACE"` en `/sbin/udhcpc -i "$INTERFACE" &`.
3. Le `&` à la fin lance le processus `udhcpc` en **arrière-plan**, ce qui permet au script `rc` de se terminer et au processus `init` de lancer le shell.

---

## Victoire : Système Fonctionnel

Après ces étapes de débogage, le système d'exploitation personnalisé démarre correctement, active le réseau, obtient une adresse IP, et fournit un shell interactif. La connectivité Internet a été confirmée par un `ping 8.8.8.8`.

**Leçons apprises :**
- L'importance de la cohérence entre le mode de démarrage (BIOS/UEFI) et l'installation du bootloader.
- La différence critique entre une console Framebuffer et une console VGA texte.
- La nécessité d'inclure les pilotes matériels (disque, réseau) dans le noyau ou, de manière plus robuste, dans un initramfs.
- La gestion des processus en premier plan vs arrière-plan dans les scripts de démarrage.
