---
layout: post
title: Gérer les labels Gmail avec l'extension PHP IMAP
author: florent
---

Chez [Xotelia](http://www.xotelia.com/) nous avons mis en place un système qui analyse des emails, détecte des reservations et les enregistre ensuite dans notre système via une API.

Afin de garder une boîte email propre et ainsi pouvoir suivre en temps réel l'évolution du système et comprendre rapidement les dysfonctionnements, nous avons décidé de trier les emails à l'aide des labels Gmail en fonction des étapes de traitement.

Après plusieurs essais infructeux, attribuant des labels de manière plus ou moins aléatoire, j'ai enfin trouvé le moyen d'assigner facilement des labels aux emails au travers d'une connexion [IMAP](http://fr.wikipedia.org/wiki/Internet_Message_Access_Protocol).

## Manipulation standard des emails avec IMAP

Pour se connecter à un compte email en IMAP, nous utilisons l'[extension PHP IMAP](http://php.net/manual/fr/book.imap.php) au travers d'une [petite lib](https://github.com/tedious/Fetch) permettant de travailler en OOP.

Même si l'implémentation du protocole IMAP en PHP est assez limitée, elle permet tout de même d'interagir avec les emails de deux façons différentes :

* soit avec le numéro de `sequence`, qui correspond à l'ordre de l'email dans la boîte,
* soit avec l'`UID`, qui est un identifiant unique définit par le serveur IMAP pour la boîte courante.

Cependant le développeur de la lib a fait le choix de ne travailler qu'avec l'`UID`. C'est effectivement plus prudent car l'`UID` d'un email reste identique quelque soit son ordre dans la boîte.

## Scope de l'UID

Mais l'`UID` ne reste unique qu'au sein d'une même boite email. Ainsi, si l'on déplace/copie un email dans une autre boîte, il n'aura plus le même `UID`.

Suivant les implémentations du serveur IMAP, les dossiers peuvent être :

* soit physiques : si on se connecte à un compte UNIX, on peut se rendre physiquement dans le dossier à l'aide de la commande `cd` et lister simplement les messages via la commande `ls`.
* soit virtuels : ils sont gérés uniquement par le serveur IMAP. C'est par exemple le cas du dossier `Sent Mails`, ou sur Gmail, du dossier `All Mail`, qui en IMAP s'appelle `[Gmail]/All Mail`. Ces dossiers virtuels peuvent contenir tout ou partie des emails présents.

## Notre première erreur

Lors de la première implémentation, une fois l'email récupéré depuis la boîte de réception, nous l'archivions tout de suite. C'est à dire en language IMAP, que nous le déplaçions du dossier `INBOX` vers le dossier `[Gmail]/All Mail` : son `UID` était alors différent dans la boîte de réception et dans le dossier d'archivage.

Ainsi, après plusieurs traitements asynchrones sur cet email, lorsque l'on était amené à lui ajouter un label (success, error, etc.), nous ne disposions que de son `UID` dans le dossier `INBOX` qui n'était donc plus valable puisqu'il avait été archivé.

Nous ajoutions donc potentiellement un label au mauvais email...

## La première solution

Afin de corriger ce problème, nous ne lisons désormais plus que les emails non lus présents dans le dossier `INBOX` :

```php
<?php

$server = new \Fetch\Server($serverPath, $port, 'imap');

$messages = $server->search('UNSEEN', $limit);

foreach ($messages as $message) {
    // do stuff
}
```

Au début du traitement ils sont marqués comme lus :

```php
<?php

$message->setFlag('seen');
```

Mais ils restent dans le même dossier tout au long des différents traitements asynchrones qui peuvent ainsi leurs assigner des labels grâce à leur `UID` inchangé.

Puis, à la fin du traitement, on les déplace dans le dossier d'archive :

```php
<?php

$message->moveToMailBox('[Gmail]/All Mail');
```

## Notre deuxième erreur

Notre projet est découpé en plusieurs petits processus qui font chacun une partie du traitement de manière asynchrone en communiquant via [RabbitMQ](http://www.rabbitmq.com/).

Le premier processus s'occupe de lire la boîte de réception, et de passer le contenu des emails aux processus suivants qui ajoutent des labels au fur et à mesure des traitements. Or tous ces processus ont leur propre connexion IMAP, qui s'établie lors de l'initialisation du processus. Et chaque processus reste en attente d'un élément à dépiler et à traiter, mais de manière complètement asynchrone.

Il se peut donc qu'un processus devant ajouter un label, initialise sa propre connexion IMAP avant le processus qui récupère l'email à traiter : lors de l'ajout du label, le message a traiter n'existe donc pas encore pour le processus s'il a été réceptionné entre les deux initialisations de connexion IMAP.


## Notre deuxième solution

Il faut donc forcer la réouverture de la connexion IMAP à chaque fois qu'un processus n'a pas connaissance d'un email qu'il devrait pourtant traiter :

```php
<?php

imap_reopen($stream, 'INBOX');
```

Ainsi, lors de l'ajout d'un label ou de l'archivage d'un email, le processus à maintenant connaissance du message dans la boîte de réception et peut donc le manipuler via son `UID` :

```php
<?php

imap_mail_copy($stream, $message->getUid(), $label, CP_UID);
```

Si le processus n'a toujours pas connaissance de l'email à traiter même après une réouverture de la connexion IMAP, nous générons une erreur.

## Traçabilité

Lorsque l'on travaille en asynchrone il est primordial de pouvoir suivre les différentes étapes de traitement. Les problèmes que nous avons rencontrés avec la gestion des labels dans Gmail nous l'ont encore prouvé.

Nous vous détaillerons donc bientôt la manière dont nous avons utilisé notre [stack ELK](/installer-une-stack-elk-rapidement/) pour tracer tous les traitements de nos différents workers développés en PHP à l'aide de [Symfony](http://symfony.com/).
