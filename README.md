# Introduction à l'IoT et aux Systèmes d'Exploitation (HEPL 2024-2025)

Ce dépôt documente la création d'un système d'exploitation Linux personnalisé, sa conteneurisation avec Docker, et son déploiement sur un cluster Kubernetes, dans le cadre du cours B38-M18P-MI1.

## Objectifs du Projet

- Comprendre le processus de construction d'un système GNU/Linux à partir des sources.
- Maîtriser les concepts de virtualisation au niveau de l'OS (conteneurs).
- Apprendre les bases de l'orchestration de conteneurs avec Kubernetes.
- Lier les concepts théoriques (processus, mémoire, I/O, FS) à leur application pratique.

## Étapes du Projet

Ce projet est découpé en plusieurs étapes, chacune documentée dans un fichier dédié :

1. **[Étape 1 : Préparation de l'environnement Gentoo](./doc/part1-gentoo-setup.md)**
   - Installation d'un système de base Gentoo dans une VM VMware.
   - Partitionnement, configuration de la compilation et installation d'un noyau de base.

2. **[Étape 2 : Création d'un OS personnalisé](./doc/part2-custom-os.md)**
   - Création d'un système de fichiers FHS.
   - Compilation de la `glibc`, de `BusyBox` et d'un noyau Linux personnalisé.

3. **[Étape 3 : Création d'une clé USB Bootable](./doc/part3-usb-boot.md)**
   - Transfert de l'OS personnalisé sur une clé USB et configuration du bootloader `syslinux`.

4. **[Étape 4 : Installation et Configuration de Docker](./doc/part4-docker-setup.md)**
   - Recompilation du noyau avec les options nécessaires pour Docker (`namespaces`, `cgroups`).
   - Installation du moteur Docker.

5. **[Étape 5 : Application Multi-Conteneurs](./doc/part5-multicontainer-app.md)**
   - Création des `Dockerfiles` pour une application web (frontend, backend, base de données).
   - Orchestration locale avec `docker-compose`.

6. **[Étape 6 : Déploiement sur Kubernetes](./doc/part6-kubernetes.md)**
   - Écriture des manifestes `Deployment` et `Service`.
   - Déploiement de l'application sur le cluster K8s du laboratoire et gestion du stockage persistant avec NFS.

## Fichiers de Configuration

Les fichiers de configuration clés utilisés tout au long de ce projet sont stockés dans le dossier [config_files/](./config_files/).
