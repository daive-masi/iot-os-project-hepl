### Mémo Complet : Du Noyau à Kubernetes

#### Étape 1 : Installation de Gentoo (Environnement de Travail)

*   **Partitionnement (`fdisk /dev/sda`)**
    *   `g` : Créer une table **GPT** (moderne).
    *   `n` : Créer les partitions (`+512M`, `+4G`, `+20G`...).
    *   `t` : Changer le type (1: EFI, 19: Swap, 24: Linux root).
    *   `w` : Écrire les changements.
    *   **Rappel Théo** : **GPT vs MBR**. Un disque est structuré en partitions pour organiser les données. GPT est le standard moderne.

*   **Formatage (`mkfs`)**
    *   `mkfs.fat -F 32 /dev/sda1` : Partition EFI **doit** être en FAT32.
    *   `mkfs.ext4 /dev/sda3` : Standard pour Linux.
    *   `mkswap /dev/sda2` & `swapon /dev/sda2` : Préparer l'espace d'échange.
    *   **Rappel Théo** : Chaque partition doit avoir un **système de fichiers** pour structurer les données. C'est le rôle du noyau de le gérer.

*   **Chroot**
    *   `mount /dev/sda3 /mnt/gentoo`
    *   `mount --rbind /dev /mnt/gentoo/dev`, `/sys`, `/proc`
    *   `chroot /mnt/gentoo /bin/bash` : Change la racine (`/`) du shell courant. Permet de travailler "à l'intérieur" du nouveau système avant même qu'il ait son propre noyau.
    *   **Rappel Théo** : Le **chroot** est un mécanisme d'isolation au niveau du système de fichiers, un ancêtre des containers.

*   **Compilation (Kernel)**
    *   `cd /usr/src/linux`
    *   `make menuconfig` : Configurer les pilotes.
    *   `make && make modules_install` : Compiler le noyau et ses modules.
    *   `cp arch/x86/boot/bzImage /boot/kernel-...`
    *   **Rappel Théo** : Le **noyau** est le cœur de l'OS. Il a besoin des **pilotes** (drivers) pour communiquer avec le matériel. Les pilotes peuvent être compilés "en dur" (`*`) ou en tant que **modules** (`M`) chargés à la demande.

*   **Bootloader (`GRUB`)**
    *   `grub-install /dev/sda`
    *   `grub-mkconfig -o /boot/grub/grub.cfg`
    *   **Rappel Théo** : Le **bootloader** est le premier programme lancé par le firmware (BIOS/UEFI). Son rôle est de charger le noyau en mémoire et de lui passer la main.

---

#### Étape 2 & 3 : OS Personnalisé & Clé USB

*   **FHS (`mkdir -p`)** : Créer la structure (`/bin`, `/etc`, `/usr`...).
    *   **Rappel Théo** : **Filesystem Hierarchy Standard**. Une convention de nommage qui organise les fichiers sur un système Linux.

*   **Bibliothèque C (`glibc`) & Utilitaires (`BusyBox`)**
    *   `configure --prefix=/usr`
    *   `make && make install_root=/mnt/monlinux install` : Cross-compilation. On compile sur le système hôte pour installer sur le système cible.
    *   **Rappel Théo** : Un OS a besoin d'une **bibliothèque C** (pour que les programmes puissent faire des appels système) et d'**utilitaires** de base (`ls`, `cp`, `sh`...).

*   **Configuration Minimale**
    *   `/etc/passwd` : Définit les utilisateurs.
    *   `/etc/fstab` : Dit au système quelles partitions monter au démarrage.
    *   `/etc/inittab` : Fichier de configuration du processus **`init`** (PID 1). Lui dit quoi lancer (ex: un script de démarrage, un shell de login).
    *   `/etc/rc` : Le script de démarrage principal.

*   **Réseau OS Perso**
    *   `ip a` : Voir les interfaces.
    *   `ip link set eth0 up` : Activer une interface.
    *   `udhcpc -i eth0 &` : Obtenir une IP via DHCP en arrière-plan.
    *   **Rappel Théo** : La configuration réseau implique la **couche 2** (activer l'interface) et la **couche 3** (assigner une adresse IP et une route par défaut).

---

#### Étape 4 & 5 : Docker & Docker Compose

*   **Vérification Kernel pour Docker**
    *   `./check-config.sh` : Script essentiel pour valider que le noyau a bien les `namespaces` et `cgroups`.
    *   **Rappel Théo** : Les **containers** ne sont que des processus Linux normaux, mais **isolés** par les **namespaces** (chacun a sa propre vue du réseau, des PIDs, etc.) et **limités** par les **cgroups** (limite CPU/RAM).

*   **Commandes Docker Essentielles**
    *   `docker build -t <nom>:<tag> .` : Construire une image depuis un `Dockerfile`.
    *   `docker run -d --name <nom> -p <port_hôte>:<port_container> ... <image>` : Démarrer un container.
    *   `docker ps` : Lister les containers actifs.
    *   `docker logs <nom_container>` : Voir les logs.
    *   `docker exec -it <nom_container> sh` : Entrer dans un container.
    *   `docker network create <nom>` : Créer un réseau privé.
    *   `docker system prune -a --volumes` : Nettoyer l'espace disque.

*   **Docker Compose**
    *   `docker compose up --build -d` : Construire et démarrer toute l'application.
    *   `docker compose down --volumes` : Arrêter et supprimer l'application et ses données.
    *   **Rappel Théo** : Docker Compose permet de définir une application multi-containers de manière **déclarative** dans un fichier `YAML`, ce qui automatise le déploiement local. C'est un "mini-orchestrateur".

---

#### Étape 6 : Kubernetes

*   **Préparation (Registry & NFS)**
    *   `docker run -d -p 5000:5000 ... registry:2` : Lancer une registry locale.
    *   `/etc/docker/daemon.json` ou `/etc/containerd/config.toml` : Fichiers clés pour configurer l'accès aux **registries non sécurisées**.
    *   `exportfs -a` : Appliquer la configuration du serveur NFS.
    *   `docker push <registry>/<image>:<tag>` : Pousser une image vers une registry.

*   **Commandes `kubectl` Essentielles**
    *   `kubectl apply -f <fichier.yaml>` : Appliquer une configuration.
    *   `kubectl get <ressource>` : Lister des ressources (`pods`, `svc`, `pv`, `pvc`, `deploy`...).
    *   `kubectl describe <ressource> <nom>` : Obtenir des informations détaillées et les `Events`. **Commande de débogage la plus importante.**
    *   `kubectl logs <nom_pod>` : Voir les logs d'un pod.
    *   `kubectl delete <ressource> <nom>` : Supprimer une ressource.
    *   `-n <namespace>` : Option **indispensable** pour travailler dans le bon namespace.
    *   `-w` : "Watch", pour voir les changements en temps réel.

*   **Concepts Clés de Kubernetes**
    *   **`Pod`** : La plus petite unité de déploiement. Un ou plusieurs containers partageant le même réseau.
    *   **`Deployment`** : Gère des `ReplicaSet` qui assurent qu'un certain nombre de `Pods` stateless sont toujours en cours d'exécution.
    *   **`StatefulSet`** : Comme un `Deployment`, mais pour des applications avec état (bases de données), garantissant une identité stable.
    *   **`Service`** : Fournit une **adresse IP et un nom DNS stables** pour un groupe de pods. Agit comme un load balancer interne.
        *   `ClusterIP` : Accessible uniquement depuis l'intérieur du cluster.
        *   `NodePort` : Expose le service sur un port statique sur chaque nœud du cluster.
    *   **`Secret`** : Pour stocker des données sensibles (mots de passe, clés).
    *   **`ConfigMap`** : Pour stocker des données de configuration non sensibles.
    *   **`PersistentVolume (PV)`** : L'**offre** de stockage (ex: "J'ai un disque NFS de 1Gi disponible").
    *   **`PersistentVolumeClaim (PVC)`** : La **demande** de stockage faite par un pod (ex: "J'ai besoin de 1Gi de stockage"). Kubernetes lie un PV à un PVC.