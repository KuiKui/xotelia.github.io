---
layout: post
title: Mise en place de Robocop
author: florent
---

Celà fait maintenant quelque temps que c'est en production, mais il n'est jamais trop tard pour en parler. [Xotelia](http://www.xotelia.com) est un gestionnaire de canaux de vente. Notre job c'est de faire en sorte que les calendriers de toutes les chambres, gîtes, hôtels, etc qui sont gérés par Xotelia soient à jour sur tous les sites de vente sur lesquels ils sont présent, ce peut être Airbnb, comme Booking.com ou encore Expedia. Donc il faut sans cesse aller mettre à jour les calendriers de ces sites dès qu'on reçoit une réservation, ou qu'un propriétaire modifie ses calendriers sur notre _back office_.

## Contextualisation

Historiquement, cette synchronisation est faite de manière asynchrone via un système d'événements et de queues. Seulement le problème c'est que la queue en question c'est MySQL et MySQL est fait pour faire plein de choses, mais surtout pas un système de queue. Pour ça il y a [RabbitMQ](https://www.rabbitmq.com/) par exemple. Non seulement MySQL n'est pas fait pour, mais plus la base client augmente, plus le nombre de synchronisation demandé augments, plus la taille de la queue augmente et donc plus les performances de MySQL se dégradent. Suite à cette constatation nous avons voulu aller plus loin en sortant le système de synchronisation du _back office_ pour 1. ne pas gêner les performances du backoffice et 2. réécrire le avec des technos adaptées.

Ainsi est né Robocop.

Cette décision fait suite à notre volonté d'orienter Xotelia vers une architecture de type _microservices_ pour pouvoir plus facilement le maintenir mais surtout encaisser la charge à venir.

## Architecture

Nous sommes tout de suite parti sur une architecture complèmetent asynchrone, en essayant de découper au maximum les process. Désormais, quand une demande de synchronisation apparait, au lieu qu'elle soit stockée dans la _queue_ MySQL, elle est envoyée à Robocop via une API REST.

![process](/public/images/robocop.png)

Comme vous pouvez le voir ce n'est pas les données à synchroniser qui sont envoyées à Robocop, mais seulement un _masque_ de synchronisation qui indique ce qui doit être synchronisé. Nous avons fais ce choix d'une part pour des raisons de performamce, mais surtout parce que entre le moment où la demande est envoyée et le moment où Robocop va effectivement mettre à jour le canal de vente, il peut se passer plusieurs secondes voire minutes. Et donc la donnée peut avoir changé entre temps.

Donc nous avons plusieurs process, le premier, est un simple contrôleur, il se contente de prendre le masque de synchronisation et de l'envoyer dans un premier _exchange_. Le deuxième est un _worker_ de calcul, il prend le masque, récupère les données à synchroniser, les re découpe en fonction des besoins du canal de vente sur lequel il faut mettre à jour et envoi le tout dans un deuxième _exchange_ pour la synchronisation.

Pour ce deuxième _exchange_, nous sommes partis sur le modèle _topic_ de RabbitMQ, qui permet et rediriger vers telle ou telle queue en fonction d'une clée de routage. Ce modèle correspond très bien à nos besoins, car il permet d'avoir un ou plusieurs par queue spécifique à chaque canal de vente.

Ensuite la dernière étape consiste à prendre le message dans la queue relative au canal de vente en question, et de pousser les données.

## Under the hood

Il s'agit d'un projet Symfony 2 avec RabbitMQ. Rien de bien complexe à mettre en place. Le problème a surtout été de trouver comment gérer les _workers_. Nous sommes d'abord parti sur une technique pas très propre, avec un script bash qui lance les workers, récupère les PID et les relance s'ils ont consommer le nombre de message demandé. Cette technique n'est pas a recommander car se baser sur le PID via bash n'est pas ce qu'il y a de plus sûr.

### Supervision des workers

Nous sommes donc parti sur [Supervisor](http://supervisord.org/). Il s'agit un superviseur de processus écrit en python avec une configuration déclarative comme _systemd_, à l'inverse d'initscript ou la configuration se fait en bash (mourir). L'avantage de cette technique c'est que ce programme est conçu pour maintenir les services qu'il gère _up_. Si le processus est tué ou que le nombre de message à consomme est atteint, supervisor se charge de relancer le processus. Le problème c'est que du coup on ne peut plus vraiment spécifier un nombre de message pour chaque worker, car supervisor ne ferait que relancer des processus en permanence, et il n'est pas conçu pour ça. Désormais, chacun de nos workers consomme un nombre de message infini jusqu'à être stoppé ou relancé.

### Déclaration des queues

Pour RabbitMQ nous sommes parti sur le très bon [RabbitMQBundle](https://github.com/videlalvaro/rabbitmqbundle). La documentation est très bien pour faire des choses simple, par contre dès qu'on souhaite profiter au maximum des possibilités de RabbitMQ, il faut fouiller. C'est le cas pour les exchanges de type topic. Pour la partie `consumer` il faut définir un consumer pour chaque queue différente qu'on veut. Un seul producer suffit.

```yaml
old_sound_rabbit_mq:
    producers:
        test:
            connection: default
            exchange_options: { name: 'test', type: topic }
    consumers:
        foo:
            connection: default
            exchange_options: { name: 'test', type: topic }
            queue_options: { name: 'test.foo', routing_keys: ['foo'] }
            callback: foo.consumer
        bar:
            connection: default
            exchange_options: { name: 'test', type: topic }
            queue_options: { name: 'test.bar', routing_keys: ['bar'] }
            callback: bar.consumer
```

Avec cette configuration nous disposons d'un _producer_ et de deux _consumers_. Tous les messages envoyés par notre _producer_ `test` avec la clé `foo` iront dans la queue `test.too` et ceux avec la clé `bar` iront dans la queue `test.bar`. Il est possible de ne pas spécifier le nom de la queue et de laisser RabbitMQ s'en charger. Mais ceci devient problématique quand les _consumers_ sont redémarrés, RabbitMQ génère un nouveau nom de queue et donc tous les messages dans l'ancienne queue sont perdus. Aussi, si on spécifie le nom de la queue, mais qu'on lance les _consumers_ avec des clés de routage différents, il se passe la même chose que si on avait configuré l'_exchange_ en mode direct.

## État des lieux

Celà fait maintenant plusieurs mois que Robocop est en production, nous avons maintenant une dizaine de canaux de ventes gérés par Robocop et les performances s'en ressentent sur le _back office_. De plus avec RabbitMQ, qu'on ait 3 messages dans une queue, ou 3 millions, ler performances seront les même (à peu de chose près bien sûr). Nous avons une latence de sychonisation d'une dizaine de seconde en moyenne. Mais grâce à cette architecture nous allons pouvoir profiter du _cloud_ et ainsi déployer Robocop sur autant de machines que nécessaire pour s'adapter au traffic.

La prochaine étape conciste à avoir un dashboard sur lequel on peut choisir les _workers_ dont on a besoin, dire le nombre de process qu'on veut et laisser Robocop les placer sur tel ou tel serveur en fonction de la charge et des restrictions.
