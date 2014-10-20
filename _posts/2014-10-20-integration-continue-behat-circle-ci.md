---
layout: post
title: Intégration continue avec Behat sur CircleCI
author: florent
---

Dans le but d'avoir le meilleur suivi de code possible, nous avons mis en place de l'intégration continue. Après un de réflexion, et surtout parce qu'on avait pas envie de gérer ça nous-mêmes, nous avons choisi [Circle CI](https://circleci.com/). Pour les dépôts privés, ça me parait être la meilleure solution.

## Environnement

Tout d'abord, concernant notre application d'email processor, nous avons les dépendences suivantes :

* PHP cli **5.4** ou supérieur
* serveur SMTP (postfix ou Gmail)
* serveur IMAP (dovecot ou Gmail)
* RabbitMQ **3.2** ou supérieur

Nous avons aussi besoin des extensions suivantes de PHP :

* `imap.so`
* `mailparse.so`
* `intl.so`
* `curl.so`

L'extension `mailparse` est disponible sur PECL, les autres sont disponibles via des paquest Debian.

## Exécution des tests dans un environnement natif

Après avoir naïvement lancé un premier build sans aucune configuration sur Circle CI, hormis la définition de la version de PHP (5.4.4), les tests d'intégration ont échoué. Circle CI permet de relancer un build en activant la connexion SSH, ce qui permet de se connecter sur la machine qui éffectue les tests. Passé quelques minutes d'investigation, j'ai remarqué que lq version de PHP disponible c'est pas celle fournit par les dépôts Debian, mais une version compilée en utilisant [phpenv](https://github.com/phpenv/phpenv). J'ai aussi lancé un `php -m` pour avoir la liste des modules PHP activés, et bien sûr, l'extension imap ne l'est pas.

Je commence d'abord par essayer d'installer tout ce dont j'ai besoin via le gestionnaire de paquets. Seulement voilà, les boxs sur lesquelles les tests sont exécutés sont installées avec une vieille version d'ubuntu et la version de PHP dans les dépôts est **5.3**. L'application que nous développons requiert la version 5.4 au minimum car nous utilisons par exemple les _short array syntax_ `['foo' => 'bar']`.

## Compilation de PHP : échec

Je décide donc d'abandonner cette solution pour PHP, et je commence a étudier phpenv afin de savoir comment je peux relancer une compilation de la version de PHP que j'ai besoin, mais en ajoutant les extensions PHP nécessaires à mon application. Après un certain temps d'investigation, je remarque que Circle CI n'utilise pas le fichier de build de phpenv, mais celui de [php-build](https://github.com/CHH/php-build). Je commence donc à étudier ce projet et la je vois que je peux àjouter les paramètres que je veux au `./configure` en utilisant la variable suivante `PHP_BUILD_CONFIGURE_OPTS`.

Je relance donc une compilation de PHP sur Circle CI avec `PHP_BUILD_CONFIGURE_OPTS=--with-imap`, et la s'en suit plusieurs échecs. Il manque des dépendances sur la machine pour compiler PHP avec le support d'imap, d'autres extensions pqr défaut ont aussi des dépendences manquantes. Après avoir parcouru _les Internets_ à la recherche d'une solution, je n'arrive toujours pas à compiler PHP avec imap.

## Docker pour sauver la mise

C'est là que je me rappelle qu'il n'y a pas si longtemps que ça, Circle CI à annoncé le [support de Docker](http://blog.circleci.com/continuous-delivery-with-docker-containers/). Je me dis, super !, je vais pouvoir lancer mes dépendances dans des containers Docker et ne plus avoir à toucher la box Circle CI.

Pour RabbitMQ, j'ai trouvé une image Docker sur le [registry](https://registry.hub.docker.com/u/dockerfile/rabbitmq/). Pour les serveurs SMTP et IMAP, je n'ai rien trouvé qui correspond à mes besoins, alors j'ai créé moi-même une [image Docker](https://registry.hub.docker.com/u/luxifer/docker-postfix-dovecot/). Et pour PHP, je me suis inspiré du travail fait sur notre [box vagrant](https://github.com/Xotelia/VagrantBox) en gardant seulement ce qui correspond à PHP. Docker permet de _linker_ les containers entre eux, pour partager des informations via des variables d'environnement (voir [Docker links](https://docs.docker.com/userguide/dockerlinks/)).

Donc je lance les dépendances :

```
$ docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 dockerfile/rabbitmq
$ docker run -d --name mail -p 25:25 -p 143:143 -p 993:993 luxifer/docker-postfix-dovecot
```

Ce qui me permet d'avoir un serveur RabbitMQ dans un container Docker avec le port **5672** et **15672** ouverts, respectivement le protocole AMQP et le plugin de management. Et aussi un autre container Docker avec à l'intérieur un serveur Postfix et un serveur Dovecot, respectivement SMTP et IMAP, le tout orchestré par Supervisor. Les ports **25** (SMTP), **143** et **993** (IMAP sans et avec SSL) sont eux aussi ouverts. Il ne me reste plus qu'à _linker_ ces deux containers avec celui qui contient PHP et exécuter les tests avec Behat :

```
$ docker run --link rabbitmq:rabbitmq --link mail:mail xotelia/php bin/behat
```

## Configuration Symfony

Seulement voilà, les paramètres Symfony de configuration d'accès aux différents serveurs sont dans un fichier (parameters.yml) et il me faut les définir au lancement de l'application en les lisant depuis les variables d'environnement fournies par Docker. En fouillant la documentation de Symfony, je suis tombé sur [cette page](http://symfony.com/fr/doc/current/cookbook/configuration/external_parameters.html). Qui décrit, via le fichier de configuration de base de Symfony (config.yml), comment charger des configurations externes. J'ai commencer par regarder comment me servir des variables d'environnement `SYMFONY__<param name>`. Seulement il faudrait que je renomme mes containers `symfony__rabbitmq_`, etc. Et il faudrait aussi que je change le nommage de mes paramètres. Ce qui représente trop de travail pour la valeur ajoutée. Je me suis rendu compte qu'on pouvait aussi charger un fichier PHP, qui est la solution que j'ai retenu. Elle me permet de garder le système avec le fichiers "parameters.yml", et à la suite je charge un fichier "parameters.php" dans lequel je surcharge uniquement les paramètres qui m'intéresse :

```php
<?php

if (getenv('MAIL_NAME')) {
    $container->setParameter('mailer_host', getenv('MAIL_PORT_25_TCP_ADDR')); // variable d'env contenant l'adresse IP sur serveur SMTP
}
```

Je passe les détails sur les autres paramètres que je surcharger car ils se basent sur le même principe : vérifier qu'on à un container de _linké_, surcharger les paramètres relatifs à ce container en allant lire les variables d'environnement.

## Je conv, tu convs, `iconv`...

Je suis donc fin prêt à lancer mes tests, au début tout se passe bien, Behat se lance, les tests d'intégration commencent jusqu'à un moment ou ça bloque au moment où un mail de _fixture_ qui doit être traité ne passe pas dans la bonne pile. J'essaye en local, pas de problèmes. Je mets ce problème de côté quelque temps, car je ne trouve toujours aucune solution, jusqu'au monent ou je tombe sur la page de la documentation de la fonction PHP [iconv](http://fr2.php.net/manual/en/function.iconv.php). En fait pour savoir si un mail doit être traité ou pas, j'analyse le sujet et j'exécute une série de _regex_. Mais avant d'analyser le sujet, je supprime tous les caractères qui ne font pas partie de la table ASCII via cette fonction `iconv`.

Le problème avec cette fonction, c'est qu'elle se base sur la locale de la machine, via la variable d'environnement `LC_CTYPE`. et le problème avec Docker c'est la locale est `posix` (voir [ici](http://fr2.php.net/manual/en/function.iconv.php#74101)). Et avec cette locale, les accents ne sont pas remplacés par leur lettre analogue mais par un "?" dans un rectangle noir. Ce qui fait que ma suite de regex ne _matchait_ pas. Le fait est aussi que se baser sur l'environnement pour une translitération n'est pas fiable.

## Intl à la rescousse

Vu qu'on utilise PHP 5.4, je suis tombé sur une réponse Stackoverflow qui propose d'utiliser la class [Transliterator](http://fr2.php.net/manual/en/class.transliterator.php). Cette classe est fournie par l'extension PHP Intl :

```php
<?php

$transliterator = \Transliterator::create(
    'NFD; [:Nonspacing Mark:] Remove; NFC;'
);
$transliterator->transliterate('é_è'); // e_e
```


Cette fonction n'étant pas documentée, j'ai pris la chaîne de caractère fourie dans la réponse Stackoverflow. Cette méthode à le même résultat qu'avec la fonction `iconv`, sauf qu'elle ne se base pas sur l'environnement de la machine mais est agnostique.

Nous voilà avec un environnement d'exécution des tests fonctionnel !
