### Mémo de Commandes Complet du Projet

#### Pré-étape : Configuration initiale de la VM Gentoo (Hôte)

1.  **Partitionnement du disque (`fdisk /dev/sda`)**
    *   Commandes internes : `g` (GPT), `n` (new part), `t` (type), `w` (write).
    *   Partitions créées : `sda1` (EFI), `sda2` (Swap), `sda3` (Gentoo Root), `sda4` (Labo Part 1), `sda5` (Labo Part 2).

2.  **Formatage et montage initial**
    ```bash
    mkfs.fat -F 32 /dev/sda1
    mkfs.ext4 /dev/sda3
    mkswap /dev/sda2 && swapon /dev/sda2
    mount /dev/sda3 /mnt/gentoo
    ```

3.  **Installation et configuration de la base Gentoo**
    ```bash
    cd /mnt/gentoo
    # Télécharger et extraire le stage3-openrc.tar.xz
    tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
    # Configurer /mnt/gentoo/etc/portage/make.conf (MAKEOPTS, CFLAGS, etc.)
    # Préparer le chroot
    mount --types proc /proc /mnt/gentoo/proc
    mount --rbind /sys /mnt/gentoo/sys && mount --make-rslave /mnt/gentoo/sys
    mount --rbind /dev /mnt/gentoo/dev && mount --make-rslave /mnt/gentoo/dev
    chroot /mnt/gentoo /bin/bash
    source /etc/profile
    # Synchroniser Portage
    emerge-webrsync && emerge --sync
    # Mise à jour du système et installation des outils de base
    emerge --verbose --update --deep --newuse @world
    emerge sys-kernel/gentoo-sources sys-boot/grub net-misc/dhcpcd
    # Configurer /etc/fstab, locales, timezone...
    passwd # Définir le mot de passe root
    ```

4.  **Compilation et installation du noyau de base (Gentoo hôte)**
    ```bash
    cd /usr/src/linux
    make menuconfig # Activer les pilotes VMware (SATA/AHCI, VMXNET3), support GPT
    make && make modules_install
    cp arch/x86/boot/bzImage /boot/kernel-gentoo-base
    grub-install /dev/sda
    grub-mkconfig -o /boot/grub/grub.cfg
    # Quitter chroot, démonter et rebooter.
    ```

---

#### Étape 2 : Création de l'OS Personnalisé

1.  **Préparation du système de fichiers (sur la VM Gentoo)**
    ```bash
    mkfs.ext4 /dev/sda5
    mount /dev/sda5 /mnt/monlinux
    # Création du FHS
    mkdir -p /mnt/monlinux/{bin,boot,dev,etc,home,lib,lib64,mnt,proc,root,sbin,sys,tmp,usr,var}
    mkdir -p /mnt/monlinux/usr/{bin,include,lib,lib64,local,sbin,src,share/man}
    mkdir -p /mnt/monlinux/var/{lock,log,run,spool}
    ```

2.  **Compilation croisée de Glibc et BusyBox**
    ```bash
    # Glibc
    cd /usr/src/glibc-2.39/
    mkdir build && cd build
    ../configure --prefix=/usr --host=x86_64-pc-linux-gnu --enable-kernel=...
    make
    make install_root=/mnt/monlinux install

    # BusyBox
    cd /usr/src/busybox-1.36.1/
    make menuconfig # Activer ash, init, udhcpc, ip, ls, etc.
    make
    make CONFIG_PREFIX=/mnt/monlinux install
    ```

3.  **Configuration de l'OS personnalisé**
    ```bash
    # Créer les fichiers de configuration minimaux dans /mnt/monlinux/etc/
    # /mnt/monlinux/etc/passwd -> root::0:0:root:/root:/bin/ash
    # /mnt/monlinux/etc/fstab -> /proc /proc proc defaults 0 0
    # /mnt/monlinux/etc/inittab -> ::sysinit:/etc/rc ... tty1::respawn:/bin/ash -l
    # Créer le script de démarrage /mnt/monlinux/etc/rc et le rendre exécutable
    nano /mnt/monlinux/etc/rc
    chmod +x /mnt/monlinux/etc/rc
    ```
    *Contenu clé de `/etc/rc` :*
    `mount -av`
    `mount -o remount,rw /`
    `hostname NGOUMOU`
    `ip link set eth0 up`
    `udhcpc -i eth0 &`

4.  **Compilation du noyau personnalisé et mise à jour de GRUB**
    ```bash
    # Sur la VM Gentoo
    cd /usr/src/linux-6.6.14/
    make defconfig # Puis make menuconfig pour ajuster
    make && make modules_install
    # Copier le noyau sur la partition de boot de l'hôte
    cp arch/x86/boot/bzImage /boot/kernel-6.6.14-NGOUMOU
    # Mettre à jour la configuration de GRUB pour ajouter une entrée
    grub-mkconfig -o /boot/grub/grub.cfg
    # Éditer manuellement /boot/grub/grub.cfg pour pointer vers root=/dev/sda5
    ```

---

#### Étape 3 : Création de la Clé USB Bootable

1.  **Partitionnement et formatage de la clé (`/dev/sdb`)**
    ```bash
    fdisk /dev/sdb # g, n, t, w...
    mkfs.fat -F 32 /dev/sdb1
    mkfs.ext4 /dev/sdb2
    mkfs.ntfs -f /dev/sdb3
    ```

2.  **Installation du bootloader Syslinux**
    ```bash
    mount /dev/sdb1 /mnt/usb
    mkdir -p /mnt/usb/EFI/BOOT
    # Copier syslinux.efi (renommé en bootx64.efi), ldlinux.e64, et les .c32
    # Copier le noyau vmlinuz et l'initrd fournis pour ce labo
    ```

3.  **Configuration de Syslinux**
    ```bash
    blkid /dev/sdb2 # Obtenir l'UUID
    nano /mnt/usb/EFI/BOOT/syslinux.cfg
    ```
    *Contenu clé :* `LABEL NGOUMOU`, `LINUX vmlinuz`, `INITRD initrd`, `APPEND root=UUID=...`

4.  **Copie de l'OS personnalisé**
    ```bash
    mount /dev/sdb2 /mnt/cleusbpart1
    cp -a /mnt/monlinux/* /mnt/cleusbpart1/
    umount /mnt/usb /mnt/cleusbpart1
    ```

---

#### Étape 4 & 5 : Docker et Application Multi-Containers

1.  **Recompilation du noyau Gentoo pour Docker**
    `make menuconfig` (activer namespaces, cgroups, overlayfs, netfilter...).

2.  **Débogage et finalisation des `Dockerfiles`**
    *   Gestion des dépôts archivés pour `nginx:1.21.0` et `node:14`.
    *   `Dockerfile` final pour frontend/backend :
        `RUN echo "deb http://archive.debian.org/debian/ buster main" > /etc/apt/sources.list ...`
        `RUN apt-get update -o ... --allow-unauthenticated && apt-get install -y ... && apt-get clean && rm ...`
    *   `Dockerfile` final pour database : `FROM mongo:4.4`, copie du script `init-mongo.js`.

3.  **Finalisation de `docker-compose.yml`**
    *   Suppression du `build` pour `oauth2-proxy`, utilisation de l'image officielle.
    *   Externalisation de la configuration : `volumes` pour le `oauth2-proxy.cfg`, `secrets` pour les mots de passe et clés.
    *   Suppression des ports exposés pour le frontend et le backend.
    *   Configuration du port du proxy : `ports: - "80:4180"`.

4.  **Déploiement et débogage final local**
    ```bash
    # Nettoyage complet si nécessaire
    docker compose down --volumes --rmi all
    # Lancement
    docker compose up --build -d
    # Vérification
    docker ps
    docker logs <container>
    ```

---

#### Étape 6 : Déploiement sur Kubernetes

1.  **Préparation de l'environnement (Registry & NFS)**
    *   Sur le serveur de services (`192.168.2.202`) : `docker run ... registry:2`, `mkdir /mnt/mongo-ngoumou`, `exportfs -a`.
    *   Sur la VM Gentoo (build) ET tous les nœuds K8s : configurer `/etc/containerd/config.toml` pour la registry non sécurisée, `sudo systemctl restart containerd`.
    *   Sur tous les nœuds K8s workers : `sudo apt-get install -y nfs-common`.

2.  **Push des images**
    ```bash
    # Sur la VM Gentoo, pour chaque image
    docker tag <image_locale> 192.168.2.202:5000/<nom_image>:<tag>
    docker push 192.168.2.202:5000/<nom_image>:<tag>
    ```

3.  **Déploiement sur le cluster**
    *   Copier les dossiers `kubernetes/` et `secrets/` vers le master node avec `scp`.
    *   Sur le master node :
        ```bash
        # Connexion en root
        sudo -i
        export KUBECONFIG=/etc/kubernetes/admin.conf

        # Nettoyage si nécessaire
        kubectl delete namespace chat-app

        # Déploiement dans l'ordre
        kubectl create namespace chat-app
        kubectl create secret generic mongo-secret-ngoumou --from-file=... -n chat-app
        kubectl create secret generic oauth-secret-ngoumou --from-file=... -n chat-app
        kubectl apply -f kubernetes/ # Appliquer tous les YAMLs
        ```

4.  **Vérification et débogage sur Kubernetes**
    ```bash
    kubectl get all -n chat-app
    kubectl get pods -n chat-app -w # Surveiller
    kubectl describe pod <nom_pod> -n chat-app # Le couteau suisse du débogage
    kubectl logs <nom_pod> -n chat-app
    kubectl logs <nom_pod> --previous -n chat-app # Pour les pods en CrashLoopBackOff
    ```

5.  **Accès à l'application**
    ```bash
    # Trouver le NodePort
    kubectl get service oauth-service-ngoumou -n chat-app
    # Accéder via http://<IP_WORKER>:<NODE_PORT>
    ```
---

#### Étape 6 : Déploiement sur Kubernetes (Suite et fin)

3.  **Création des Manifestes YAML pour Kubernetes**
    L'objectif est de traduire la logique de `docker-compose` en objets Kubernetes, en externalisant la configuration.

    *   **`namespace.yaml`** : Isoler l'application.
        *   `kind: Namespace`, `metadata: name: chat-app`
    *   **`nfs-persistent-volume.yaml`** : Définir l'offre de stockage.
        *   `kind: PersistentVolume`, `spec: capacity`, `accessModes`, `nfs: {path: ..., server: ...}`.
    *   **`secrets.yaml`** (créé via `kubectl create secret --dry-run`)
        *   `kind: Secret`, `metadata: name: ...-ngoumou`, `data: {key: value_base64}`.
    *   **`configmap.yaml`** : Fichier de configuration du proxy.
        *   `kind: ConfigMap`, `metadata: name: oauth-config-ngoumou`, `data: oauth2-proxy.cfg: | ...`.
    *   **`10-database.yml`** : Déploiement de la base de données.
        *   **`kind: Service`** (`clusterIP: None`, "headless") pour donner une identité réseau au `StatefulSet`.
        *   **`kind: StatefulSet`** :
            *   `replicas: 1`.
            *   `serviceName`: pointe vers le service headless.
            *   `template.spec.containers.image`: pointe vers l'image sur la **registry privée**.
            *   `env`: utilise `valueFrom: secretKeyRef:` pour injecter le mot de passe depuis le secret.
            *   `volumeClaimTemplates`: définit la **demande** de stockage (le PVC) qui sera automatiquement créée.
    *   **`30-backend.yml`** : Déploiement du backend.
        *   **`kind: Service`** (`ClusterIP`) pour l'accès interne.
        *   **`kind: Deployment`** :
            *   `template.spec.initContainers`: `busybox` (depuis la registry privée) qui attend le service de la base de données (`nslookup database-service-ngoumou`).
            *   `template.spec.containers.image`: image du backend depuis la registry privée.
            *   `env`: variables pour l'hôte et le nom de la DB, et `MONGO_PASSWORD_FILE` pointant vers un volume secret.
            *   `volumeMounts`: monte le secret MongoDB dans `/etc/mongo-secret`.
            *   `volumes`: définit le volume à partir du `Secret` `mongo-secret-ngoumou`.
    *   **`20-frontend.yml`** : Déploiement du frontend.
        *   **`kind: Service`** (`NodePort`, ex: `30081`) pour le contournement du WebSocket.
        *   **`kind: Deployment`** :
            *   `template.spec.initContainers`: `busybox` qui attend le service du backend (`nslookup backend-service-ngoumou`).
            *   `template.spec.containers.image`: image du frontend depuis la registry privée (avec le `index.html` et `nginx.conf` corrigés).
            *   `imagePullPolicy: Always` : Très utile pendant le débogage pour forcer la mise à jour de l'image.
    *   **`40-oauth.yml`** : Déploiement du proxy.
        *   **`kind: Service`** (`NodePort`, ex: `30080`) pour être le point d'entrée principal.
        *   **`kind: Deployment`** :
            *   `template.spec.containers.image`: **l'image officielle `oauth2-proxy`** poussée sur la registry privée.
            *   `args`: pointe vers le fichier de configuration monté (`--config=/etc/oauth-config/oauth2-proxy.cfg`).
            *   `volumeMounts`: monte le `ConfigMap` (`oauth-config-ngoumou`) et le `Secret` (`oauth-secret-ngoumou`).
            *   `volumes`: définit les volumes à partir du `ConfigMap` et du `Secret`.

4.  **Déploiement sur le cluster**
    *   Copier les dossiers `kubernetes/` et `secrets/` vers le master node avec `scp`.
    *   Sur le master node :
        ```bash
        # Connexion en root
        sudo -i
        export KUBECONFIG=/etc/kubernetes/admin.conf

        # Nettoyage complet
        kubectl delete namespace chat-app
        # Attendre la suppression
        
        # Déploiement dans l'ordre
        kubectl create namespace chat-app
        kubectl create secret generic mongo-secret-ngoumou --from-file=... -n chat-app
        kubectl create secret generic oauth-secret-ngoumou --from-file=... -n chat-app
        
        # Appliquer toutes les configurations
        kubectl apply -f kubernetes/
        ```

5.  **Vérification et débogage sur Kubernetes**
    ```bash
    # Lister toutes les ressources dans le namespace
    kubectl get all -n chat-app
    
    # Surveiller les pods en temps réel
    kubectl get pods -n chat-app -w 
    
    # Le couteau suisse du débogage pour comprendre pourquoi un pod est en "Pending" ou "CrashLoopBackOff"
    kubectl describe pod <nom_pod> -n chat-app
    
    # Voir les logs d'un pod en cours d'exécution
    kubectl logs <nom_pod> -n chat-app
    
    # Voir les logs du dernier crash pour un pod en "CrashLoopBackOff"
    kubectl logs <nom_pod> --previous -n chat-app
    
    # Entrer dans un pod pour des tests réseau
    kubectl exec -it <nom_pod> -n chat-app -- sh
    ```

6.  **Accès à l'application**
    ```bash
    # Trouver le NodePort
    kubectl get service oauth-service-ngoumou -n chat-app
    
    # Accéder via http://<IP_UN_WORKER>:<NODE_PORT_OAUTH>
    # La connexion WebSocket se fera sur le <NODE_PORT_FRONTEND>
    ```