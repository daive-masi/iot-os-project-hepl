# Étape 6 : Déploiement sur un Cluster Kubernetes

L'objectif final de ce projet est de déployer l'application multi-containers, développée et testée localement avec Docker Compose, sur un cluster Kubernetes "bare-metal" fourni dans l'environnement du data center.

## 6.1 Préparation de l'Infrastructure Externe

Le cluster Kubernetes étant dans un environnement déconnecté d'Internet, une préparation minutieuse était nécessaire pour assurer un déploiement autonome.

### 6.1.1 Registry d'Images Privée

Toutes les images Docker requises pour l'application (images personnalisées et images publiques comme `oauth2-proxy`) ont dû être stockées sur une registry privée accessible par le cluster.

1.  **Mise en place :** Un container `registry:2` a été lancé sur un serveur dédié (`192.168.2.202`).
2.  **Préparation des Images :** Sur la VM Gentoo (avec accès Internet), toutes les images ont été téléchargées (`docker pull`), puis re-taguées pour pointer vers la registry privée.
3.  **Transfert :** Les images ont été poussées (`docker push`) vers `192.168.2.202:5000`.
4.  **Configuration du Cluster :** Sur chaque nœud du cluster (master et workers), le démon `containerd` a été configuré pour accepter cette registry non sécurisée (HTTP) en modifiant le fichier `/etc/containerd/config.toml`.

### 6.1.2 Stockage Persistant NFS

Pour assurer la persistance des données de la base de données MongoDB, un partage NFS a été mis en place.

1.  **Mise en place :** Le service `nfs-kernel-server` a été installé sur le serveur dédié (`192.168.2.202`).
2.  **Création du Partage :** Un répertoire (`/mnt/mongo-ngoumou`) a été créé et exporté via `/etc/exports` pour être accessible par le sous-réseau du cluster.
3.  **Configuration du Cluster :** Sur chaque nœud worker, le client NFS (`nfs-common`) a été installé pour permettre le montage des volumes.

## 6.2 "Kubernetisation" de l'Application

La configuration `docker-compose.yml` a été "traduite" en une série de manifestes YAML déclaratifs pour Kubernetes, en suivant les meilleures pratiques "cloud-native".

*   **Isolation :** Toutes les ressources ont été déployées dans un `Namespace` dédié (`chat-app`).
*   **Configuration :** Les paramètres de configuration statiques ont été externalisés dans des `ConfigMaps` (ex: `oauth2-proxy.cfg`).
*   **Secrets :** Toutes les données sensibles (mots de passe, clés secrètes) ont été gérées via des `Secrets` Kubernetes. Ces secrets ont été créés à partir de fichiers locaux avec `kubectl create secret`.
*   **Stockage :** La persistance a été gérée par un couple `PersistentVolume` (PV), décrivant l'offre de stockage NFS, et `PersistentVolumeClaim` (PVC), qui est la demande de stockage faite par le `StatefulSet` de la base de données.
*   **Charges de travail :**
    *   Les services sans état (`backend`, `frontend`, `proxy`) ont été déployés via des `Deployments`.
    *   La base de données a été déployée avec un `StatefulSet` pour garantir une identité réseau stable et une gestion ordonnée.
*   **Réseau :**
    *   La communication interne entre les services est assurée par des `Services` de type `ClusterIP`.
    *   L'accès externe à l'application est fourni par des `Services` de type `NodePort`, exposant le proxy d'authentification et le frontend (pour les WebSockets) sur des ports spécifiques des nœuds workers.

## 6.3 Processus de Débogage

Le déploiement a nécessité la résolution de plusieurs problèmes typiques d'un environnement Kubernetes :

1.  **`ErrImagePull` :** Le cluster ne pouvait pas télécharger les images. Résolu en configurant `containerd` sur chaque nœud pour qu'il fasse confiance à la registry privée non sécurisée.
2.  **Pod `Pending` (Stockage) :** Le pod de la base de données ne démarrait pas. Diagnostiqué avec `kubectl describe` comme un problème de `PersistentVolumeClaim` non lié. Résolu en nettoyant et en recréant correctement le `PersistentVolume` et le `PersistentVolumeClaim`.
3.  **`CrashLoopBackOff` (Dépendances) :** Les pods démarraient dans le désordre et plantaient car leurs dépendances (comme la base de données ou un autre service) n'étaient pas prêtes. Résolu en ajoutant des `initContainers` aux déploiements pour forcer une attente active.
4.  **`CrashLoopBackOff` (DNS) :** Nginx ne parvenait pas à résoudre le nom du service backend au démarrage. Résolu en ajoutant une directive `resolver` dans `nginx.conf` pour forcer l'utilisation du DNS de Kubernetes.
5.  **`Connection Refused` (Pare-feu) :** Le navigateur ne pouvait pas atteindre les `NodePort`. Résolu en ajoutant des règles au pare-feu `ufw` sur les nœuds workers pour autoriser le trafic entrant sur les ports `30080` et `30081`.

## 6.4 Victoire Finale : Application Fonctionnelle

Après ces étapes, l'application est entièrement fonctionnelle et accessible via l'adresse d'un nœud worker et le `NodePort` configuré (`http://192.168.2.211:30080`).

![Application Fonctionnelle sur Kubernetes](URL_DE_VOTRE_SCREENSHOT_FINAL) <!-- Optionnel : Intégrez votre screenshot final ici -->

Ce déploiement final valide la maîtrise de l'ensemble de la chaîne, de la compilation d'un noyau Linux à l'orchestration d'une application microservices complexe dans un environnement distribué.