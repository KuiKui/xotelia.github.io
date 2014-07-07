---
layout: post
title: Gérer le routing avec NewRelic
author: florent
---

Dernièrement au boulot nous avons mis en place NewRelic sur le projet que nous développons, à savoir Xotelia. Alors pour commencer NewRelic c’est tout simplement génial ! C’est un outil de monitoring d’application qui permet d’avoir toutes les informations nécessaires à la vie d’une application, comme le temps de chargement des pages, les différents composants qui rentrent en compte dans ce temps de chargement (appel à la base de donnée, appel à des services externes, temps de traitement, etc). NewRelic fait bien plus que ça mais je ne vais pas rentrer dans le détail. Nous écrirons prochainement un article détaillé sur tout ce que fait NewRelic, et comment expliquer à son patron que ça fait gagner du temps.

Rentrons dans le vif du sujet. Notre projet est développé en PHP et donc pour activer NewRelic il faut charger leur extension et installer un démon sur le serveur. Au bout de quelques minutes on voit apparaître les données sur notre dashboard.

![Dashboard NewRelic](/public/images/dashboard.png)

### Nom des transactions (one file to rule them all)

Seulement voilà, notre application est développée à partir d’un framework maison et donc dans le dashboard on ne voit apparaitre qu’une seule transaction nommée "index.php". Donc on perd tout l'intérêt du service.

![Une seule transaction](/public/images/transaction-one.png)

### La solution (the magic trick)

En cherchant un peu sur la documentation de l’agent PHP, je suis tombé sur leur [FAQ](https://docs.newrelic.com/docs/php/php-agent-faq#wt-naming) qui explique pourquoi toutes les transactions peuvent s’appeler "index.php". NewRelic connaît certains framework PHP comme Drupal, mais pour les autres c’est à l’utilisateur de lui indiquer comment nommer ses transactions. Pour cela, l’agent PHP met a disposition une [fonction](https://docs.newrelic.com/docs/php/php-agent-api#api-name-wt). Il suffit d’utiliser cette fonction à l’endroit du framework ou est fait le routing :

```php
<?php

if (extension_loaded('newrelic')) { 
    newrelic_name_transaction($controller . '/' . $action);
 }
```

Sans oublier de vérifier si l’extension PHP est chargée (ce qui sera le cas seulement en production), sinon l’application ne fonctionnera plus dans votre environnement local.

Après avoir déployé cette modification en production, on voit apparaître ses transactions nommées dans le dashboard. Houra !

![Liste des transactions](/public/images/transaction-multiple.png)

Nous pouvons maintenant commencer l’analyse de performance de nos différents contrôleurs.
