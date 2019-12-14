# Moncker, the replicaset

## Introduction:

Ce projet à pour but de déployer un cluster mongoDB à 4 noeuds (1 primaire, 3 workers) en utilisant la technologie du ReplicaSet et les capacités de Docker Compose.

Les schémas suivant vous présentes brièves le type d'architecture mise en place:
![alt text](https://docs.mongodb.com/manual/_images/replica-set-read-write-operations-primary.bakedsvg.svg)

Source: https://docs.mongodb.com/manual/replication/
![alt text](https://docs.mongodb.com/manual/_images/replica-set-primary-with-two-secondaries.bakedsvg.svg)

Source: https://docs.mongodb.com/manual/replication/

Un noeud primaire sera resposable de l'écriture et de la lecture quand les workers répliquerons la donnée.

## Description du docker-composer.yml:

Les volumes permettent d'allouer un espace disque pour la persistance des données et le partage entre container. Ici chaque noeud possède son propre volume (pour la réplication des données).
Un volume nous servira à stocker les clés de chiffrement quand les autres seront des bases mongoDb.

Network nous permet de définir le mode de réseau et le management de IP sous un "profil". Subnet nous permet de définir notre masque de sous réseau qui définit les IP machines adressables.

Enfin 5 services seront définis pour chacun de nos volumes. Nous retrouvons:

-Le service chargée du chiffrement des communications . Nous utilisons une image depop/openssl-bats qui prend en charge le chiffrement SSL/TLS à l'aide d'openssl.
La commande indiquée nous permet de générer une chaine aléatoire de 768 octects encodée en BASE64 qui sera stockées dans le fichier mongodb-keyfile et limitant les droit de modifications au propriétaire et la lecture à tous.

-Le noeud Primaire responsable de la lecture écriture et les workers auront chacun un espace disque pour leur base de données respectives. Ils ont tous accès à l'espace disque qui contient la clé de chiffrement. On mappera un port HOST différent pour chaque pour 27017 de nos containers. Dans la commande nous précisons le chemin vers notre keyfile puis le nom de notre replicaSet (identique pour tous les noeuds).
Seul le noeud primaire bénéficie des variables d'environnement qui définisse le login et password du compte root mongodb.
On s'assure que chaque noeud sera lancé après génération des clés de notre service gen-mongo-key avec le depends_on.
Tous les noeuds mongo partagent la même configuration réseau définit plus haut sous le profil.

## Deploiment:

Pour déployer le cluster

```
docker-compose up
```

puis en ouvrant un autre terminal nous nous connectons au noeud principal

```
docker-compose exec node-mongo-primary mongo -u "root" -p "password"
```

nous initialisons le replicaSet

```
rs.initiate({"_id" : "profile-replica-set","members" : [{"_id" : 0,"host" : "node-mongo-primary:27017"},{"_id" : 1,"host" : "node-mongo-worker-1:27017"},{"_id" : 2,"host" : "node-mongo-worker-2:27017"},{"_id" : 3,"host" : "node-mongo-worker-3:27017"}]});
```

afin de définir le noeud comme primaire nous modifions sa priorité à 2

```
conf = rs.config();
conf.members[0].priority = 2;
rs.reconfig(conf);
```

nous créons un compte admin pour le cluster

```
use admin;
db.createUser({user: "cluster_admin",pwd: "password",roles: [ { role: "userAdminAnyDatabase", db: "admin" },  { "role" : "clusterAdmin", "db" : "admin" } ]});
db.auth("cluster_admin", "password");
```

puis une collection dans notre base de donnée

```
use my_data;
db.createUser({user: "my_user",pwd: "password",roles: [ { role: "readWrite", db: "my_data" } ]});
db.createCollection('my_collection');
```

Enfin nous pouvons nous connecter et afficher la liste des collections présentes pour vérifier que notre ajout a fonctionné

```
docker-compose exec node-mongo-primary mongo -u "my_user" -p "password" --authenticationDatabase "my_data"
show collections
```

## Ouverture possible:

-Sharding : Permet la haute disponibilité grâce à une répartion de la donnée sur plusieurs machines. Il peut(doit) être couplé au ReplicaSet.

![alt text](https://docs.mongodb.com/manual/_images/sharded-cluster-production-architecture.bakedsvg.svg)

Source: https://docs.mongodb.com/manual/sharding/

###Source:
https://www.youtube.com/watch?v=-XzMfd4XQak +
Passage de mongo 4.0.3 à 4.2 avec correction de certaines erreurs liée au changement de version.

```

```
