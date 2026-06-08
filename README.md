# Projet Docker Swarm sur Debian

Ce projet présente la mise en place d’une infrastructure Docker Swarm sur plusieurs machines virtuelles Debian, avec un nœud manager, des nœuds workers, un stockage NFS et plusieurs services critiques conteneurisés. Le fonctionnement de Swarm repose sur des nœuds managers pour l’orchestration, des workers pour exécuter les tâches, et un quorum de managers pour conserver les fonctions d’administration du cluster.

## Objectif du projet

L’objectif est de construire une architecture Docker Swarm sécurisée, résiliente et documentée, capable d’héberger plusieurs services critiques avec persistance des données et mécanismes de reprise après incident.

Les services à déployer sont les suivants :

- Un **registry interne** pour stocker les images Docker.
- Une base de données **MariaDB**.
- Un serveur applicatif **PHP**.
- Un proxy inverse **Nginx**.
- Un environnement de développement **VSCode Server**.

## Dépôt GitHub

Le projet est à rendre sur : [https://github.com/samuel-rigaux/swarm](https://github.com/samuel/rigaux/swarm)

Structure recommandée du dépôt :

```text
swarm/
├── README.md
├── docker-stack.yml
├── conf/
│   ├── nginx/
│   ├── php/
│   └── mariadb/
├── scripts/
│   ├── backup.sh
│   ├── restore.sh
│   └── test-failover.sh
└── docs/
    └── captures/
```

Cette organisation facilite la lecture, la maintenance et la vérification par le correcteur, notamment pour les procédures de déploiement, de sauvegarde et de tests.

## Architecture cible

### Machines virtuelles

| Machine | Rôle | Description |
|---------|------|-------------|
| `manager-1` | Manager | Orchestration du cluster Swarm, lancement des commandes d’administration. |
| `worker-1` | Worker | Exécution des conteneurs applicatifs. |
| `worker-2` | Worker | Redondance et répartition de charge. |
| `nfs-1` | NFS | Hébergement des volumes persistants partagés. |

### Services conteneurisés

| Service | Rôle | Stockage |
|---------|------|----------|
| Registry | Dépôt local d’images Docker. | Volume persistant NFS. |
| MariaDB | Base de données de l’application. | Volume persistant NFS, sauvegarde logique recommandée. |
| PHP | Exécution de l’application métier. | Peut utiliser un volume partagé si nécessaire. |
| Nginx | Reverse proxy et serveur web. | Configurations versionnées dans Git. |
| VSCode Server | Poste de développement distant. | Données utilisateur sur volume NFS. |

## Pré-requis

Avant le déploiement, prévoir :

- 4 VM Debian minimum avec IP fixes recommandées.
- Docker installé sur les nœuds Swarm.
- `nfs-kernel-server` installé sur la VM NFS.
- `nfs-common` installé sur les nœuds clients.
- Un accès SSH administrateur sur toutes les machines.

## Déploiement du cluster

### 1. Initialiser le manager

Sur le nœud manager :

```bash
docker swarm init --advertise-addr <IP_MANAGER>
```

Cette commande initialise le cluster et génère la commande `docker swarm join` nécessaire pour rattacher les workers.

### 2. Ajouter les workers

Sur chaque worker :

```bash
docker swarm join --token <TOKEN> <IP_MANAGER>:2377
```

Vérification depuis le manager :

```bash
docker node ls
```

Le manager doit apparaître en `Leader` et les workers en état `Ready`.

### 3. Mettre en place le serveur NFS

Sur la VM NFS :

```bash
apt install -y nfs-kernel-server
mkdir -p /nfs/data
chmod 777 /nfs/data
```

Dans `/etc/exports` :

```text
/nfs/data 192.168.1.0/24(rw,sync,no_subtree_check)
```

Puis :

```bash
exportfs -a
systemctl restart nfs-kernel-server
```

Le partage NFS servira à héberger les volumes persistants des services Docker.

### 4. Monter NFS sur les nœuds Swarm

Sur chaque manager et worker :

```bash
apt install -y nfs-common
mkdir -p /mnt/nfs-data
mount -t nfs <IP_NFS>:/nfs/data /mnt/nfs-data
```

Cette étape permet aux services de retrouver leurs données après redéploiement ou déplacement sur un autre nœud.

### 5. Déployer la stack applicative

Depuis le manager :

```bash
docker stack deploy -c docker-stack.yml swarmapp
```

Vérifications :

```bash
docker service ls
docker stack ps swarmapp
```

`docker service ls` permet de vérifier les réplicas, et `docker stack ps` affiche la répartition des tâches dans le cluster.

## Procédures de tests de fonctionnement

Les tests ci-dessous doivent apparaître dans le README, avec objectif, commandes et résultat attendu. Cette présentation permet au correcteur de rejouer les scénarios et de vérifier la logique d’exploitation du cluster.

### Test 1 — Vérification générale du cluster

**Objectif :** confirmer que le cluster est opérationnel.

**Commandes :**

```bash
docker node ls
docker service ls
docker stack ps swarmapp
```

**Résultat attendu :**

- Tous les nœuds sont en état `Ready`.
- Le manager est `Leader`.
- Tous les services ont le nombre de réplicas attendu.

---

### Test 2 — Vérification d’accessibilité des services

**Objectif :** vérifier que les services exposés sont accessibles.

**Commandes :**

```bash
curl http://<IP_NGINX>
curl http://<IP_MANAGER>:5000/v2/_catalog
curl http://<IP_MANAGER>:8443
```

**Résultat attendu :**

- Nginx répond en HTTP.
- Le registry répond via son API.
- VSCode Server est joignable sur son port publié si le service est démarré.

---

### Test 3 — Vérification de la persistance NFS

**Objectif :** vérifier que les données restent présentes après recréation d’un conteneur.

**Procédure :**

1. Créer un fichier test dans un volume stocké sur NFS.
2. Forcer le redéploiement du service.
3. Vérifier que le fichier existe encore.

**Exemple :**

```bash
docker exec -it <conteneur_php> sh
echo "test persistance" > /var/www/html/test.txt
exit
docker service update --force swarmapp_php
```

**Résultat attendu :** le fichier est toujours présent après la recréation du conteneur, ce qui valide le stockage persistant.

---

### Test 4 — Simulation de perte d’un conteneur

**Objectif :** vérifier que Swarm recrée automatiquement une tâche perdue pour respecter le nombre de réplicas défini.

**Commandes :**

```bash
docker ps
docker rm -f <ID_CONTENEUR>
docker service ps swarmapp_php
```

**Résultat attendu :**

- Le conteneur supprimé apparaît comme arrêté.
- Une nouvelle tâche est recréée automatiquement par Swarm.

---

### Test 5 — Simulation de perte d’un worker

**Objectif :** vérifier la continuité de service en cas de panne d’un nœud de calcul.

**Procédure :**

1. Arrêter Docker sur un worker ou éteindre la VM.
2. Vérifier l’état du nœud.
3. Contrôler la relocalisation des tâches.

**Commandes :**

```bash
systemctl stop docker
docker node ls
docker service ps swarmapp_php
```

**Résultat attendu :**

- Le worker passe en `Down` ou `Unavailable`.
- Les services répliqués sont replacés sur les nœuds disponibles si les contraintes le permettent.

---

### Test 6 — Simulation de perte d’un manager

**Objectif :** observer l’impact d’une panne manager sur l’administration du cluster.

**À documenter :**

- Avec un seul manager, la perte du manager empêche toute administration du cluster, même si des services déjà lancés peuvent continuer à tourner.
- Avec trois managers, la perte d’un manager reste tolérable tant que le quorum est conservé, soit 2 managers sur 3.

**Commandes :**

```bash
docker node ls
docker service ls
```

**Résultat attendu :**

- Les services peuvent continuer à fonctionner.
- Les commandes d’administration dépendent du maintien du quorum des managers.

---

### Test 7 — Vérification des healthchecks

**Objectif :** s’assurer qu’un service défaillant est détecté puis redémarré si le `healthcheck` échoue plusieurs fois de suite dans Swarm.

**Exemple de configuration :**

```yaml
healthcheck:
  test: ["CMD", "test", "-f", "/tmp/healthy"]
  interval: 10s
  timeout: 3s
  retries: 3
```

**Scénario :**

1. Déployer le service.
2. Supprimer le fichier surveillé.
3. Attendre plusieurs cycles.
4. Vérifier que le conteneur est remplacé.

**Résultat attendu :** le conteneur devient `unhealthy`, puis Swarm le remplace après plusieurs échecs consécutifs.

## Vérification PCA

Le **PCA** (plan de continuité d’activité) consiste à maintenir le service disponible malgré une panne partielle. Dans Docker Swarm, cela repose sur la réplication des services, plusieurs workers, et le maintien du quorum côté managers pour conserver la capacité de pilotage du cluster.

### Points de contrôle PCA

- Les services web doivent avoir au moins 2 réplicas.
- Les services doivent être répartis sur plusieurs nœuds si possible.
- La perte d’un worker ne doit pas rendre l’application indisponible.
- Les données doivent rester accessibles grâce au stockage NFS.

**Exemple de commandes :**

```bash
docker service inspect swarmapp_nginx --pretty
docker service inspect swarmapp_php --pretty
```

**Validation PCA :** après arrêt d’un worker, l’application reste accessible et les réplicas encore actifs prennent le relais.

## Vérification PRA

Le **PRA** (plan de reprise d’activité) décrit la restauration du service ou de l’administration après une panne majeure. En cas de perte du quorum manager, les conteneurs en cours peuvent continuer à tourner, mais l’administration du cluster devient impossible tant qu’un quorum n’est pas restauré ou qu’un nouveau cluster n’est pas recréé depuis un manager survivant.

### Sauvegardes à prévoir

- Sauvegarde du répertoire NFS `/nfs/data`.
- Sauvegarde logique de MariaDB avec `mysqldump`.
- Sauvegarde du fichier `docker-stack.yml` et des fichiers de configuration dans GitHub.

### PRA — Perte d’un worker

1. Recréer la VM.
2. Réinstaller Docker.
3. Relancer la commande `docker swarm join`.
4. Vérifier l’état du cluster.

**Commandes de contrôle :**

```bash
docker node ls
docker service ps swarmapp
```

Cette procédure permet de réintégrer rapidement un nœud de calcul dans l’infrastructure.

### PRA — Perte du quorum manager

La bonne pratique consiste d’abord à remettre en ligne les managers manquants. Si cela est impossible, un manager survivant peut recréer un cluster à un seul manager avec `docker swarm init --force-new-cluster`, puis de nouveaux managers doivent être promus afin de retrouver une haute disponibilité correcte.

**Commande de reprise :**

```bash
docker swarm init --force-new-cluster --advertise-addr <IP_MANAGER_SURVIVANT>:2377
```

**Étapes après reprise :**

1. Vérifier que le cluster répond.
2. Réintégrer ou recréer les autres managers.
3. Contrôler le quorum.
4. Revalider les services applicatifs.

## Critères de validation du projet

Le projet peut être considéré comme valide si les éléments suivants sont démontrés :

- Le cluster Swarm est fonctionnel et les nœuds sont visibles.
- Les services critiques démarrent correctement et sont accessibles.
- Les données persistent après redéploiement grâce au NFS.
- La suppression d’un conteneur entraîne sa recréation automatique.
- La perte d’un worker ne coupe pas le service si plusieurs réplicas existent.
- Le PCA est testé et validé.
- Le PRA est documenté avec une procédure claire de reprise.
