---
layout: post
title: Docker, premiers pas
author: baptiste
---

Recemment on a mis en place des tests [behat](http://docs.behat.org/) sur le projet de [BisElectric](http://www.bis-electric.com/), lancé automatiquement sur [CircleCI](https://circleci.com/) à chaque `push`.

Pour être iso à chaque lancement de la batterie de tests, on a choisi d'utiliser [Docker](https://www.docker.com/) pour faciliter le tout autant sur CircleCI qu'en local. Et vu que ça a été sujet à quelques prises de têtes, il est temps de faire partager :)

## Késako

Pour ceux qui ne connaissent pas encore, Docker est l'outil de *virtualisation* qui a le vent en poupe. Le principe est le même que [Vagrant](https://www.vagrantup.com/), à savoir faire tourner son application dans un environnement clos; le but principal étant quand même de ne pas poluer sa machine de dev et monter un environnement en moins de 2 minutes.

Dans la philosophie, chaque machine géré par Docker correspond à un processus. Dans notre cas, on a une application magento avec un serveur MySQL et un serveur Redis (pour cache et sessions), ce qui aurait du aboutir à 3 machines (une pour chaque service).

Pour aboutir à une machine il y a 3 notions à connaitre:

* le `Dockerfile`, un fichier texte de *recette* expliquant toutes les commandes à lancer pour construire son app (définition de l'OS, installation de packages, etc...)
* l'image, construite à partir du `Dockerfile`, est en quelque sorte un snapshot fonctionnel de l'application qui sera lancé à chaque `docker run`
* le container, une instance de l'image

## Mise en pratique

Pour ce premier container, en pensant faire simple, on a mis tout les services dans celui-ci. Comme on a pu le faire dans une VM Vagrant. Première erreur!

En effet, Docker garde un container *en vie* tant que le process spécifié au lancement de celui-ci reste actif. Par conséquent il a fallu avoir recours à `supervisord` pour pouvoir lancer tous les services ([Geoffrey Bachelet](https://twitter.com/ubermuda) explique bien comment faire dans son [article](http://geoffrey.io/a-php-development-environment-with-docker.html)).
Au final, on est allé à l'encontre de la philosophie de base de Docker tout en rajoutant un niveau de complexité par dessus.

On suit le tutorial de Geoffrey, tout se passe plutôt bien. En local le container se lance bien avec le serveur web qui tourne correctement. On push, le build CircleCI se lance, et là, c'est le drame. L'app ne se lance pas.
Impossible de se connecter dans notre container pour voir les logs, CircleCI ne permet pas de lancer la commande `docker exec`. Pause réflexion.

Le problème venait que `composer` n'arrivait pas à télécharger les dépendances. La raison? Vous voyez les commandes de base comme `ps`, `curl` (ou `wget`) et `git`? Elles ne sont pas présente dans notre container. Le problème était passé inaperçu en local vu que les *vendor*s étaient déjà installés.
Retour sur le `Dockerfile` pour rajouter l'installation de ces commandes (on peut vraiment dire que l'image `debian:wheezy` est réduite au minimum).

Prochaine étape, faire tourner les tests dans le container. Premier réflexe, laisser le script qui lance le container tel quel, et lancer notre commande qui run les tests via `docker exec app ./test`. Après quelques essais, les tests se lancent, passent et on a un joli exit code à `0`. Génial me direz vous, mais pas si vite!
Dans le doute on test aussi que les tests fail correctement. On crée un faux test, on lance, les tests fail, et là un exit code à `0` alors qu'on s'attend à `1`. On a surement dû oublié une étape?! En fait non, Docker ne supporte pas (encore) l'exit code via sa commande `exec`.

Il faut trouver un autre moyen pour lancer nos tests. La solution de dépannage a été de lancer nous même `supervisord` dans notre fichier de test, et une fois que l'env est build, on lance `behat`. Pas des plus propres mais au moins ça marche.
Au final notre fichier ressemble à ça:

```sh
#! /bin/sh

if [ -n "$DOCKER" ]; then
    /usr/bin/supervisord &>/dev/null

    while [ ! -f /tmp/.docker.ok ] #on crée nous même ce fichier pour savoir quand l'env est build
    do
        sleep 2
    done
fi

./vendor/bin/behat
```
Et on lance le tout via: `docker run -P -v .:/srv -e DOCKER=true app ./test`.

## Quelques points à retenir

La commande `exec` ne gère pas correctement les exit code, donc à proscrire dans vos scripts. Faut s'en contenter pour se connecter à un container (ie: `docker exec -it app bash`).

Sujet non discuté dans l'article, mais assez pénible, concerne le `Dockerfile`. Chaque ligne est mis en cache après la première execution de `docker build`; le système reprend un peu le principe de git avec un commit par ligne. L'avantage est que si on rajoute une instruction `RUN` (ou modifie une existante) et on relance le build, il va pas relancer toutes les autres instructions.
En pratique, il y a petit problème dans le cas où vous commencez votre `Dockerfile` par un `apt-get update -y` et sur une autre ligne vous installez php (ou autre package). Exemple:

```
RUN apt-get update -y
RUN apt-get install -y php5
```

Maintenant si après quelques temps vous rajoutez un package lié à php, et le mettez sur la même ligne que `php5` vous allez avoir une petite surprise au prochain `docker build`. En effet, si il y a une mise à jour de la version php, le chemin vers le fichier du package php, récupéré par l'`update` au premier `docker build`, n'est plus bon; et par conséquent php ne pourra pas s'installer. Et si vous avez des systèmes automatisés (comme CircleCI), au prochain lancement l'exécution plantera aussi.

Pour contourner le problème, vous pouvez ne placer qu'un seul install de package par ligne; mais on atteint vite un Dockerfile avec plein de lignes. Mais attention là encore, Docker ne supporte que `127` layers par image (c'est-à-dire 127 instructions).
Mais si on atteint cette limite, il faut peut-être se poser la question de savoir si l'application ne pourrait pas être redécoupée.
Une autre possibilité est de lancer le build avec l'option `--no-cache` à chaque fois, mais avoir une système qui peut changer d'état à chaque lancement n'est pas très stable.
Sur papier (car pas encore testé), la solution ultime reste par la création de tag sur votre image. A chaque fois que vous modifiez des dépendances dans le `Dockerfile`, on build, on teste que tout tourne correctement et on tag. Au final c'est le même principe que son code qu'on tag dans git.

Dernier point unqiement lié à ceux qui ne sont pas sur linux et par conséquent vont devoir utiliser `boot2docker`. Si vous voulez utiliser Docker pour votre environnement de dev et donc accéder à votre application via un navigateur, vous allez remarquer que *out-of-the-box* l'exposition de port ne marche pas. Avec un `docker run -p 8080:80 app` on s'attendrait à pouvoir accéder à notre app via `localhost:8080`, mais en fait il manque une étape. Il faut aller dans VirtualBox et modifier les paramètres de la VM `boot2docker-vm`, dans `Network > Adapter 1 > Port forwarding` il faut rajouter une ligne qui redirige les connections sur le port `8080` de votre machine sur celui de la VM, qui a son tour sera redirigé sur le `80` de votre container.
A noter qu'il ne faut pas utiliser l'option `-P` avec `run` sinon docker va attribuer un port aléatoire et il vous faudra modifier les paramètres de la VM à chaque fois que vous relancez le container de votre app, ce qui est juste ingérable.

## Conclusion

Docker est une technologie très prometteuse qui vaut le coup de prendre du temps pour l'étudier. Mais honnetement elle reste encore très barbu, j'en veux pour exemple les commandes à lancer pour démarrer notre app découpée par service:
```sh
docker run --name mysql -d mysql:5.5 -e MYSQL_ROOT_PASSWORD=somep@ssword
docker run --name redis -d redis:latest
docker run --name my-app -d -p 8080:80 --link mysql:mysql --link redis:redis app-image
```
Imaginez vous en train de taper ça à chaque fois que vous voulez lancer votre app. Pas très user friendly.

Mais Docker évolue vite, et des projets comme [Docker Compose](https://blog.docker.com/2015/01/dockercon-eu-introducing-docker-compose/) arrivent pour nous faciliter la vie (un fichier yaml et un `docker up` c'est quand même plus sexy ^^).

Docker est bien parti pour devenir incontournable, alors n'hésitez pas à commencer à jouer avec dès aujourd'hui (même s'il peut être prise de tête quelques fois).

Après tout, c'est en se plantant qu'on apprend.
