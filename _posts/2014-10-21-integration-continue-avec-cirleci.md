---
layout: post
title: Intégration continue avec CircleCI
author: florent
---

Après avoir [convaincu notre boss](/installer-behat-rapidement/) de l'utilité des tests automatisés, nous nous sommes lancés dans la mise en place de l'intégration continue. Comme nous ne souhaitons héberger aucun outil (l'administration de la prod est déjà assez...intéressante), nous nous sommes tournés vers [Circle CI](https://circleci.com/) qui parait être la meilleure solution pour les dépôts GitHub privés.

## Environnement

L'application que nous souhaitons tester automatiquement et continuellement a les dépendances suivantes :

* PHP cli **5.4+**,
* serveur SMTP (postfix ou Gmail),
* serveur IMAP (dovecot ou Gmail),
* RabbitMQ **3.2+**.

Nous avons aussi besoin de quelques extensions PHP spécifiques :

* `imap.so`,
* `mailparse.so`,
* `intl.so`,
* `curl.so`.

L'extension `mailparse` est disponible sur PECL, les autres sont disponibles via des paquets Debian.

## Environnement natif de Circle CI

Après avoir naïvement lancé un premier build sans aucune configuration, hormis la version 5.4.4 de PHP, les tests ont échoués. Il est possible de relancer un build en activant une connexion SSH, ce qui permet de se connecter sur la machine qui effectue les tests. J'ai donc remarqué que la version de PHP installée n'était pas celle des dépôts Debian mais une version compilée avec [phpenv](https://github.com/phpenv/phpenv). Il manquait aussi l'extension `imap`.

J'ai donc essayé d'installer toutes les dépendances via le gestionnaire de paquets mais les boxes exécutant les tests sont basées sur une vieille version d'Ubuntu et la version de PHP est donc trop ancienne (5.3).

## Compilation de PHP : échec

J'abandonne donc la solution du gestionnaire de paquets pour PHP et je commence a étudier *phpenv* afin de savoir comment relancer une compilation de la version de PHP dont j'ai besoin, mais en ajoutant les extensions PHP nécessaires à mon application.

Après un certain temps d'investigation, je remarque que Circle CI n'utilise pas le fichier de build de *phpenv*, mais celui de [php-build](https://github.com/CHH/php-build) : je peux donc ajouter des paramètres au `./configure` en utilisant la variable `PHP_BUILD_CONFIGURE_OPTS`.

Je relance alors une compilation de PHP sur Circle CI avec `PHP_BUILD_CONFIGURE_OPTS=--with-imap`, mais il manque toujours des dépendances sur la machine pour réussir la compilation avec le support d'imap. D'autres extensions par défaut ont aussi des dépendences manquantes. Après avoir parcouru _les Internets_ à la recherche d'une solution, je n'arrive toujours pas à compiler PHP avec imap.

## Docker pour sauver la mise

C'est là que je me rappelle qu'il n'y a pas si longtemps que ça, Circle CI a annoncé le [support de Docker](http://blog.circleci.com/continuous-delivery-with-docker-containers/). Je vais pouvoir lancer mes dépendances dans des containers Docker et ne plus avoir à toucher la box de Circle CI !

Pour RabbitMQ, j'ai trouvé une image Docker sur le [registry](https://registry.hub.docker.com/u/dockerfile/rabbitmq/). Pour les serveurs SMTP et IMAP, je n'ai rien trouvé qui corresponde à mes besoins, alors j'ai créé ma propre [image Docker](https://registry.hub.docker.com/u/luxifer/docker-postfix-dovecot/). Et pour PHP, je me suis inspiré du travail fait sur notre [box vagrant](https://github.com/Xotelia/VagrantBox) en gardant seulement ce qui concerne PHP.

Docker permet de [lier les containers entre eux](https://docs.docker.com/userguide/dockerlinks/), afin de partager des informations via des variables d'environnement.

Pour créer un serveur RabbitMQ dans un container Docker avec les ports **5672** (AMQP) et **15672** (plugin de management) ouverts :

```
$ docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 dockerfile/rabbitmq
```

Pour créer un autre container Docker contenant un serveur Postfix (SMTP) et un serveur Dovecot (IMAP) orchestrés par Supervisor et ayant les ports **25** (SMTP), **143** et **993** (IMAP sans et avec SSL) ouverts :

```
$ docker run -d --name mail -p 25:25 -p 143:143 -p 993:993 luxifer/docker-postfix-dovecot
```

Il ne me reste plus qu'à lier ces deux containers avec celui qui contient PHP tout en lançant les tests Behat :

```
$ docker run --link rabbitmq:rabbitmq --link mail:mail xotelia/php bin/behat
```

## Configuration Symfony

Cependant, les paramètres de configuration de [Symfony](http://symfony.com/) présents dans le fichier `parameters.yml` doivent être définis au lancement de l'application à partir des variables d'environnement fournies par Docker. En fouillant la documentation de Symfony, je suis tombé sur [cette page](http://symfony.com/fr/doc/current/cookbook/configuration/external_parameters.html) qui décrit comment charger des configurations externes via le fichier `config.yml`.

J'ai commencé par regarder comment me servir des variables d'environnement `SYMFONY__<param name>`, mais cela impliquait de renommer mes containers `symfony__rabbitmq_`, etc. Il aurait aussi fallu que je change le nom de mes paramètres et cela représentait trop de travail pour la valeur ajoutée.

Je me suis enfin aperçu que l'on pouvait simplement surcharger le fichiers `parameters.yml` avec un fichier PHP `parameters.php` :

```php
<?php

if (getenv('MAIL_NAME')) {
    $container->setParameter('mailer_host', getenv('MAIL_PORT_25_TCP_ADDR')); // variable d'env contenant l'adresse IP sur serveur SMTP
}
```

Je ne détaille pas les autres paramètres que je surcharge car c'est le même principe : vérifier qu'on a un container lié et surcharger les paramètres relatifs à ce container en allant lire les variables d'environnement.

## Je conv, tu convs, `iconv`...

Nos tests Behat se lancent enfin sur Circle CI.

Cependant, ils échouent rapidement car un email de _fixture_ ne se trouve pas dans la queue RabbitMQ attendue. J'essaie en local: pas de problèmes, forcément. Je mets ce problème de côté quelque temps, car je ne trouve aucune solution, jusqu'au moment où je tombe sur la page de la documentation de la fonction PHP [iconv](http://fr2.php.net/manual/en/function.iconv.php).

Afin de savoir si un email doit être traité, j'analyse son sujet à l'aide d'une série d'expressions régulières. Mais avant cela, je supprime tous les caractères qui ne font pas partie de la table ASCII à l'aide de la fonction `iconv`.

Or cette fonction se base sur la locale de la machine, via la variable d'environnement `LC_CTYPE`. Tandis que Docker utilise la locale `posix` (voir [ici](http://fr2.php.net/manual/en/function.iconv.php#74101)) : les accents ne sont donc pas remplacés par leur lettre analogue mais par un "?" dans un rectangle noir. Ce qui fait que ma détection ne pouvais pas fonctionner correctement.

Se baser sur l'environnement pour une translitération n'est pas fiable.

## Intl à la rescousse

La solution de ce problème, venue de Stackoverflow (étrange...), est d'utiliser la class [Transliterator](http://fr2.php.net/manual/en/class.transliterator.php) fournie par l'extension PHP Intl :

```php
<?php

$transliterator = \Transliterator::create(
    'NFD; [:Nonspacing Mark:] Remove; NFC;'
);
$transliterator->transliterate('é_è'); // e_e
```


Cette fonction n'étant pas documentée, j'ai copié/collé la chaîne de caractère fournie dans la réponse Stackoverflow : j'obtiens alors le bon résultat, sans me baser sur l'environnement.

## La gloire

Nous disposons donc maintenant du lancement automatisé et délocalisé de nos tests fonctionnels lors de chaque modification de notre code source.

Nous avons ensuite facilement lié notre intégration continue à notre projet sur GitHub :

![Intégration continue lié à GitHub](http://i.imgur.com/IJuQB7Y.png)

Ainsi que sur HipChat :

![Intégration continue lié à HipChat](http://i.imgur.com/JzR2Bul.png)

BG.
