---
layout: post
title: Installer Behat rapidement
author: denis
---

## Contexte

Nous souhaitons rapidement mettre en oeuvre des tests fonctionnels sur l'un de nos projets Magento.
Effectivement, nous commençons à configurer les modes d'expédition et ce qui peut paraitre simple ne l'est pas forcément : nous allons gérer 5 modes d'expéditions totalisant plusieurs dizaines de règles de sélection utilisant autant le poids, que la destination en passant par plusieurs attributs personnalisés des produits.

![Dashboard New Relic](/public/images/modes-livraison.jpg)

Même si nous avons préféré utiliser un [module](http://www.owebia.com/os2/fr/) permettant d'écrire des règles au format JSON que de modifier directement le code au sein de Magento, nous serions plus sereins si les innombrables tests que nous réalisons manuellement pour valider chaque cas étaient automatisés.

Nous avons alors décidé d'utiliser [Behat](http://behat.org/) pour écrire ces tests.

## Initialisation de Behat 2.5

### Mise à jour des dépendances

Modifier le composer.json en ajoutant les dépendances nécessaires :

```json
"require-dev": {
  "behat/mink-extension": "~1.3",
  "behat/mink-goutte-driver": "~1.0",
  "behat/mink-selenium2-driver": "~1.1"
}
```

Puis mettre à jour le projet :

```
$ php composer.phar update
```

### Configuration

Créer un fichier `behat.yml` à la racine du projet :

```yaml
default:
    paths:
        features:  features
        bootstrap: %behat.paths.features%/bootstrap
    formatter:
        name: pretty
    extensions:
        Behat\MinkExtension\Extension:
            base_url: "http://localhost/"
            goutte: ~
            selenium2:
                wd_host: "http://localhost:8643/wd/hub"
```

### Initialisation

Mettre en place la structure standard de Behat :

```
$ vendor/bin/behat --init
```

Créer un contexte Mink dans le contexte par défaut, en modifiant le fichier `features/booststrap/FeatureContexte.php` :

```
<?php

use Behat\MinkExtension\Context\MinkContext;

public function __construct(array $parameters)
{
    $this->useContext('mink', new MinkContext($parameters));
}
```

## Ecriture des features

### Feature standard

Créer un nouveau fichier `features/homepage.feature` contenant (par exemple) :

```
Feature: La homepage est fonctionnelle

    Scenario: La homepage est accessible
        Given I am on the homepage
        Then the response status code should be 200
```

Lancer les tests avec la commande :

```
$ vendor/bin/behat
```

### Feature nécessitant Javascript

Il faut d'abord installer [PhantomJS](http://phantomjs.org/download.html) puis le lancer :

```
$ ./phantomjs --webdriver=8643
```

*Note : modifier le fichier `behat.yml` en précisant le port sur lequel PhantomJS a été lancé.*

Ajouter un scénario dans le fichier `features/homepage.feature` en le prefixant par `@javascript` : c'est ce tag qui permet d'utiliser automatiquement Selenium2 (et donc PhantomJS) à la place de Goutte.

```
Feature: La homepage est fonctionnelle

    @javascript
    Scenario: Le menu en Ajax fonctionne correctement
        Given I am on the homepage
        When I press 'mon-bouton-qui-fait-de-l-ajax-moisi'
        Then I should see 'Action Ajax réussie'
```

Lancer les tests avec la commande :

```
$ vendor/bin/behat
```

## Limitation

Cette installation rapide est surtout destinée à pouvoir écrire rapidement des tests fonctionnels (ex: accompagner ou même précéder la phase de développement par l'écriture des tests correspondants ou simplement présenter le principe à son boss).

Par la suite, lors de l'augmentation du nombre de tests ou de développeurs au sein de l'équipe, il faudra certainement :
* passer par un fichier `behat.yml.dist` (pour faciliter le multi-environnement : dev, CI, etc),
* générer aléatoirement un port pour l'utilisation de PhantomJS (afin d'éviter que plusieurs builds n'utilisent le même navigateur),
* *killer* chaque instance de PhantomJS quelques soit le résultat du build pour des raisons de performances.

Mes anciens collègues de [M6Web](tech.m6web.fr) devraient pouvoir détailler tous les pièges à éviter lors de l'utilisation de Behat à grande échelle ;-)
