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
4. Compiler les pilotes de disque (`AHCI S
