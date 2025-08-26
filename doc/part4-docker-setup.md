# Étape 4 : Installation et Configuration de Docker

Cette étape consiste à préparer le système Gentoo pour la conteneurisation en **recompilant le noyau** avec les fonctionnalités requises par Docker, puis en installant l'environnement Docker.

---

## 4.1 Recompilation du noyau pour Docker

### Pourquoi ?
Docker repose sur des fonctionnalités spécifiques du noyau Linux pour l'isolation (`namespaces`) et la limitation des ressources (`cgroups`). Le noyau Gentoo de base doit être reconfiguré pour activer ces options.

---

### Étapes détaillées :

#### 1. **Démarrer sur le noyau Gentoo existant**
Assurez-vous que votre système Gentoo est démarré avec le noyau actuel avant de commencer la recompilation.

---

#### 2. **Lancer la configuration du noyau**
Accédez au répertoire des sources du noyau et lancez `menuconfig` :
```bash
cd /usr/src/linux
make menuconfig
```

---

#### 3. **Activer les options requises**
Les options suivantes doivent être activées dans le noyau pour que Docker fonctionne correctement :

##### **General setup --->**
- **Namespaces support** :
  - `UTS namespace`
  - `IPC namespace`
  - `User namespace`
  - `PID Namespaces`
  - `Network namespace`

- **Control Group support** :
  - `Memory controller`
  - `IO controller`
  - `CPU controller`
  - `Group scheduling for SCHED_OTHER`
  - `CPU bandwidth provisioning for FAIR_GROUP_SCHED`
  - `Group scheduling for SCHED_RR/FIFO`
  - `PIDs controller`
  - `Freezer controller`
  - `HugeTLB controller`
  - `Cpuset controller`
  - `Device controller`
  - `Simple CPU accounting controller`
  - `Perf controller`
  - `Support for eBPF programs attached to cgroups`

##### **Networking support --->**
- **Network packet filtering framework (Netfilter)** :
  - `Advanced netfilter configuration`
  - `Bridge IP/ARP packets filtering`
  - `Netfilter connection tracking support`
  - `IP virtual server support`
  - `IP tables support (required for filtering/masq/NAT)`
  - `Packet filtering`
  - `iptables NAT support`
  - `MASQUERADE target support`
  - `NETMAP target support`
  - `REDIRECT target support`

##### **Device Drivers --->**
- **Network device support** :
  - `Network core driver support`
  - `MAC-VLAN support`
  - `IP-VLAN support`
  - `Virtual eXtensible Local Area Network (VXLAN)`
  - `Virtual ethernet pair device`

##### **File systems --->**
- **Overlay filesystem support** : Nécessaire pour le driver de stockage `overlay2` de Docker.

---

#### 4. **Compilation et installation du noyau**
Une fois les options configurées, compilez et installez le noyau :
```bash
make && make modules_install
```
Copiez le noyau compilé vers le répertoire `/boot` avec un nom explicite :
```bash
cp arch/x86/boot/bzImage /boot/kernel-6.6.14-DOCKER
```
Mettez à jour la configuration de GRUB pour inclure le nouveau noyau :
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

#### 5. **Redémarrer sur le nouveau noyau**
Redémarrez votre système et sélectionnez le nouveau noyau (`kernel-6.6.14-DOCKER`) dans le menu GRUB :
```bash
reboot
```

---

## 4.2 Installation de l'environnement Docker

### Pourquoi ?
Une fois le noyau prêt avec les fonctionnalités requises, vous pouvez installer Docker et ses outils associés.

---

### Étapes détaillées :

#### 1. **Installation via Portage**
Utilisez Portage pour installer Docker et ses composants :
```bash
emerge app-containers/docker app-containers/docker-cli app-containers/docker-compose
```

---

#### 2. **Activation du service Docker**
Activez le service Docker pour qu'il démarre automatiquement au démarrage du système :
```bash
systemctl enable docker
systemctl start docker
```

---

#### 3. **Validation de l'installation**
Vérifiez que Docker est correctement installé et fonctionnel :
```bash
docker --version
```
Vous devriez voir une sortie similaire à :
```plaintext
Docker version 24.0.7, build afdd53b
```

---

#### 4. **Tester l'accès aux images Docker**
Recherchez une image Docker pour valider que tout fonctionne correctement :
```bash
docker search ubuntu
```

---

## 4.3 Résolution des problèmes courants

### Problème : Erreurs lors de l'installation de Docker
Si des erreurs apparaissent lors de l'installation de Docker, cela peut être dû à une configuration incorrecte du noyau. Dans ce cas :
1. **Recompilez le noyau** en vérifiant que toutes les options requises sont activées.
2. **Redémarrez** sur le nouveau noyau.
3. **Réinstallez Docker** après avoir vérifié la configuration du noyau.

---

## 4.4 Conclusion

L'étape 4 est terminée. Votre système Gentoo est maintenant prêt pour la conteneurisation avec Docker. Vous pouvez passer à l'étape suivante pour déployer votre application conteneurisée.

---

### Commandes utiles pour le débogage :
| Problème | Commande |
|----------|----------|
| Vérifier les options du noyau | `zcat /proc/config.gz | grep NAMESPACE` |
| Vérifier les modules chargés | `lsmod` |
| Vérifier le statut du service Docker | `systemctl status docker` |
| Vérifier les logs de Docker | `journalctl -u docker` |

---

### Exemple de sortie de `docker --version` :
```plaintext
Docker version 24.0.7, build afdd53b
```

---

### Exemple de sortie de `docker search ubuntu` :
```plaintext
NAME      DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
ubuntu    Ubuntu is a Debian-based Linux operating sys…   15000    [OK]
```


