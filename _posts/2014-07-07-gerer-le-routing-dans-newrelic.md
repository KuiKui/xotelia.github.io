---
layout: post
title: Gérer le routing dans New Relic
author: florent
---

Dernièrement au boulot nous avons mis en place [New Relic](http://newrelic.com/) sur le projet que nous développons, à savoir [Xotelia](http://www.xotelia.com/). Alors pour commencer New Relic c’est tout simplement génial ! C’est un outil de monitoring d’application qui permet d’avoir toutes les informations nécessaires à la vie d’une application, comme le temps de chargement des pages, les différents composants qui rentrent en compte dans ce temps de chargement (appels à la base de données, appels à des services externes, temps de traitement, etc). New Relic fait bien plus que ça mais je ne vais pas rentrer dans le détail. Nous écrirons prochainement un article détaillé sur toutes les fonctionnalités de New Relic, et comment expliquer à son patron que ça fait gagner du temps.

Rentrons dans le vif du sujet. Notre projet est développé en PHP et donc pour activer New Relic il faut charger leur extension et installer un démon sur le serveur. Au bout de quelques minutes on voit apparaître les données sur notre dashboard.

![Dashboard New Relic](/public/images/dashboard.png)

### Un fichier pour les unir, tous

Seulement voilà, notre application est développée à partir d’un framework maison et donc dans le dashboard on ne voit apparaitre qu’une seule transaction nommée "index.php". Donc on perd tout l'intérêt du service.

![Une seule transaction](/public/images/transaction-one.png)

### L'API à la rescousse

En cherchant un peu sur la documentation de l’agent PHP, je suis tombé sur leur [FAQ](https://docs.newrelic.com/docs/php/php-agent-faq#wt-naming) qui explique pourquoi toutes les transactions peuvent s’appeler "index.php". New Relic connaît certains framework PHP comme Drupal, mais pour les autres c’est à l’utilisateur de lui indiquer comment nommer ses transactions. Pour cela, l’agent PHP met a disposition une [fonction dédiée](https://docs.newrelic.com/docs/php/php-agent-api#api-name-wt). Il suffit d’utiliser cette fonction à l’endroit du framework où est fait le routing :

```php
<?php

if (extension_loaded('newrelic')) { 
    newrelic_name_transaction($controller . '/' . $action);
 }
```

Sans oublier de vérifier si l’extension PHP est chargée (ce qui sera le cas seulement en production), sinon l’application ne fonctionnera plus dans votre environnement local.

Après avoir déployé cette modification en production, on voit apparaître ses transactions nommées dans le dashboard. Houra !

![Liste des transactions](/public/images/transaction-multiple.png)

Nous pouvons maintenant commencer l’analyse des performances de nos différents contrôleurs (ça promet de longues heures de travail).
