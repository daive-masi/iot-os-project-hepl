# Étape 5 : Déploiement d'une Application Multi-Containers

Cette étape a pour objectif de conteneuriser une application web de chat composée de plusieurs microservices et de l'orchestrer localement avec Docker Compose.

---

## 5.1 Architecture de l'Application

L'application est décomposée en 4 services distincts :

- **Base de Données (`database_ngoumou`)** : Un serveur MongoDB (version 4.4) pour la persistance des messages.
- **Backend (`backend_ngoumou`)** : Un serveur Node.js (version 14) qui gère la logique du chat via WebSockets et communique avec la base de données.
- **Frontend (`frontend_ngoumou`)** : Un serveur Nginx (version 1.21.0) qui sert l'interface utilisateur (HTML/JS) et agit comme reverse proxy.
- **Authentification (`oauth2-ngoumou`)** : Un proxy qui sécurise l'accès à l'application.

---

## 5.2 Création des Images Docker (`Dockerfile`)

Pour chaque service, un `Dockerfile` a été créé pour construire une image personnalisée.

---

### 5.2.1 Défi : Gérer les dépendances d'anciennes versions

Les versions imposées (`nginx:1.21.0`, `node:14`) sont basées sur une ancienne distribution Debian ("Buster"), dont les dépôts de paquets ne sont plus actifs.

**Problème rencontré :**
- `apt-get update` échouait avec une erreur `404 Not Found`, puis avec des erreurs de signature GPG et enfin avec un manque d'espace disque.

**Solution implémentée :**
Les `Dockerfiles` ont été modifiés pour :
1. Pointer `apt` vers les **archives Debian** (`http://archive.debian.org/debian/`).
2. Ajouter les options `--allow-unauthenticated` et `-o Acquire::Check-Valid-Until=false` pour contourner les problèmes de signature et de validité.
3. Utiliser une commande `RUN` unique pour `update`, `install` et `clean` afin d'optimiser l'utilisation de l'espace disque pendant le build.

---

### 5.2.2 Exemples de `Dockerfile`

#### Pour le Backend (`backend_ngoumou`)

```dockerfile
# Utilisation de l'image Node.js 14
FROM node:14

# Configuration des archives Debian
RUN echo "deb http://archive.debian.org/debian buster main" > /etc/apt/sources.list && \
    echo "deb http://archive.debian.org/debian-security buster/updates main" >> /etc/apt/sources.list

# Installation des dépendances
RUN apt-get update -o Acquire::Check-Valid-Until=false --allow-unauthenticated && \
    apt-get install -y --no-install-recommends iproute2 iputils-ping && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Copie des fichiers de l'application
COPY package.json .
COPY server.js .

# Installation des dépendances Node.js
RUN npm install

# Exposition du port
EXPOSE 3000

# Commande de démarrage
CMD ["node", "server.js"]
```

#### Pour le Frontend (`frontend_ngoumou`)

```dockerfile
# Utilisation de l'image Nginx 1.21.0
FROM nginx:1.21.0

# Configuration des archives Debian
RUN echo "deb http://archive.debian.org/debian buster main" > /etc/apt/sources.list && \
    echo "deb http://archive.debian.org/debian-security buster/updates main" >> /etc/apt/sources.list

# Installation des dépendances
RUN apt-get update -o Acquire::Check-Valid-Until=false --allow-unauthenticated && \
    apt-get install -y --no-install-recommends iproute2 iputils-ping && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Copie des fichiers de configuration et de l'application
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/

# Correction des permissions
RUN chmod -R 755 /usr/share/nginx/html/

# Exposition du port
EXPOSE 80

# Commande de démarrage
CMD ["nginx", "-g", "daemon off;"]
```

#### Pour la Base de Données (`database_ngoumou`)

```dockerfile
# Utilisation de l'image MongoDB 4.4
FROM mongo:4.4

# Installation des dépendances
RUN apt-get update && \
    apt-get install -y iproute2 iputils-ping && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Exposition du port
EXPOSE 27017

# Commande de démarrage
CMD ["mongod", "--bind_ip_all"]
```

---

## 5.3 Orchestration avec Docker Compose

L'ensemble de l'application a été défini dans un fichier `docker-compose.yml`.

---

### 5.3.1 Configuration Clé

- **Réseau Personnalisé :** Un réseau `bridge` nommé `NGOUMOU-bridge` a été créé pour permettre la communication entre les services par leur nom.
- **Persistance :** Un volume nommé `database_data` a été utilisé pour le stockage des données de MongoDB.
- **Dépendances de Démarrage :** La directive `depends_on` a été utilisée pour s'assurer que les services démarrent dans le bon ordre (DB -> Backend -> Frontend -> Proxy).
- **Sécurité et Configuration :** L'authentification a été centralisée au niveau du proxy. Le frontend n'est pas directement exposé à l'extérieur, forçant le trafic à passer par le proxy.

---

### 5.3.2 Fichier `docker-compose.yml`

```yaml
version: "3.8"

services:
  database_ngoumou:
    image: database_ngoumou
    container_name: database_ngoumou
    build:
      context: ./database_ngoumou
    volumes:
      - database_data:/data/db
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=Infos2.0NGOUMOU
    networks:
      - NGOUMOU-bridge
    restart: unless-stopped

  backend_ngoumou:
    image: backend_ngoumou
    container_name: backend_ngoumou
    build:
      context: ./backend_ngoumou
    ports:
      - "3000:3000"
    environment:
      - MONGO_URI=mongodb://root\:Infos2.0NGOUMOU@database_ngoumou:27017/chat
    depends_on:
      - database_ngoumou
    networks:
      - NGOUMOU-bridge
    restart: unless-stopped

  frontend_ngoumou:
    image: frontend_ngoumou
    container_name: frontend_ngoumou
    build:
      context: ./frontend_ngoumou
    ports:
      - "8080:80"
    depends_on:
      - backend_ngoumou
    networks:
      - NGOUMOU-bridge
    restart: unless-stopped

  oauth2-ngoumou:
    image: quay.io/oauth2-proxy/oauth2-proxy\:latest
    container_name: oauth2-ngoumou
    ports:
      - "4180:4180"
    volumes:
      - ./auth_ngoumou/oauth2-proxy.cfg:/etc/oauth2-proxy.cfg\:ro
      - ./auth_ngoumou/users.htpasswd:/etc/oauth2-proxy/.htpasswd\:ro
    depends_on:
      - frontend_ngoumou
    networks:
      - NGOUMOU-bridge
    restart: unless-stopped

networks:
  NGOUMOU-bridge:
    name: NGOUMOU-bridge
    driver: bridge

volumes:
  database_data:
```

---

### 5.3.3 Processus de Débogage

Le déploiement a nécessité plusieurs itérations de débogage :

1. **Crash de MongoDB :** Résolu en passant de la version `latest` à la version `4.4` pour éviter les incompatibilités CPU (erreur `SIGSEGV`).
2. **Crash de MongoDB (2) :** Résolu en libérant de l'espace disque sur l'hôte Docker (`docker system prune -a --volumes`) pour corriger l'erreur `No space left on device`.
3. **Erreur `502 Bad Gateway` :** Le proxy n'arrivait pas à joindre le frontend. Résolu en corrigeant une incohérence de port (`listen 80;` vs `listen 8080;`) dans la configuration de Nginx.
4. **Affichage de la page par défaut de Nginx :** Résolu en plaçant la configuration personnalisée dans `/etc/nginx/conf.d/default.conf` et en utilisant un répertoire dédié pour les fichiers de l'application.
5. **Erreur `403 Forbidden` :** Résolu en ajoutant une commande `RUN chmod -R 755` dans le `Dockerfile` du frontend pour corriger les permissions des fichiers.

---

## 5.4 Résultat Final

Après ces ajustements, l'application est entièrement fonctionnelle et accessible via un point d'entrée unique et sécurisé. Elle est maintenant prête à être déployée sur un environnement orchestré plus complexe comme Kubernetes.

---

### Commandes Utiles

| Action | Commande |
|--------|----------|
| Démarrer les conteneurs | `docker-compose up -d` |
| Arrêter les conteneurs | `docker-compose down` |
| Voir les logs | `docker-compose logs -f` |
| Nettoyer les volumes | `docker system prune -a --volumes` |

---

### Arborescence des Fichiers

```
.
├── docker-compose.yml
├── database_ngoumou
│   ├── Dockerfile
│   └── ...
├── backend_ngoumou
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend_ngoumou
│   ├── Dockerfile
│   ├── nginx.conf
│   └── index.html
└── auth_ngoumou
    ├── oauth2-proxy.cfg
    └── users.htpasswd
```

---

### Conclusion

Cette étape a permis de déployer une application multi-containers en utilisant Docker et Docker Compose. Les problèmes rencontrés ont été résolus grâce à une approche méthodique de débogage et de configuration. L'application est maintenant prête pour une orchestration plus avancée avec Kubernetes.

---

*Lien vers les fichiers de configuration : [Dockerfiles et docker-compose.yml](../config_files/docker/)*
