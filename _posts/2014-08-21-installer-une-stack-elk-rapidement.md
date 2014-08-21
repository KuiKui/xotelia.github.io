---
layout: post
title: Installer une stack ELK rapidement
author: denis
---

Même s'il est maintenant possible d'[analyser correctement](/gerer-le-routing-dans-newrelic) le fonctionnement de notre application monolythique ([sic](http://media.giphy.com/media/Tggma69l9JOik/giphy.gif)), nous essayons tout de même de développer les nouvelles fonctionnalités au sein de [microservices](http://martinfowler.com/articles/microservices.html). Cela permet de minimiser l'interaction avec le code historique et ainsi de claquemurer la dette technique.

Une des problématiques récurrentes lors de la mise en place d'une architecture qui tend à être [distribuée](http://fr.wikipedia.org/wiki/Architecture_distribu%C3%A9e), est la re-centralisation des informations nécessitant une analyse régulière tel que l'[ensemble des logs](https://twitter.com/bdu_p/status/501622688813973504).

C'est pourquoi, dans la série des _"installations rapides pour tester le principe et convaincre son boss"_, après avoir vu [Behat](/installer-behat-rapidement) qui devrait permettre d'être un peu plus serein lors des nombreuses mises en prod, découvrons la stack ELK ([Elasticsearch](http://www.elasticsearch.org/) + [logstash](http://logstash.net/) + [Kibana](http://www.elasticsearch.org/overview/kibana/)) qui permet de centraliser, structurer et requêter un grand nombre de données issues des fichiers de logs.

## Mise en place

Notre serveur dédié à la centralisation des logs tournant sous [Debian 7](https://www.debian.org/releases/wheezy/), nous avons [installé manuellement](http://www.webupd8.org/2012/06/how-to-install-oracle-java-7-in-debian.html) Oracle Java 7, car c'est la version recommandée.

Le reste est détaillé dans cet [excellent tutoriel](https://www.digitalocean.com/community/tutorials/how-to-use-logstash-and-kibana-to-centralize-and-visualize-logs-on-ubuntu-14-04).

## Personnalisation

La stack ELK transforme des flux de données brutes en un ensemble de données structurées. Cela inclut donc bien plus que des logs d'erreurs : on peut aussi l'utiliser pour vérifier le bon fonctionnement de son application en analysant ses [propres fichiers de logs](http://highscalability.com/log-everything-all-time).

Les différentes étapes de la transformation par le serveur sont :

1. la réception du **flux** d'informations brutes provenant des fichiers de logs,
2. l'analyse du flux à l'aide d'un **filtre** présent sur le serveur,
3. le découpage de chaque ligne selon un **patterns grok** défini dans le filtre,
4. le stockage des informations structurées dans Elasticsearch.

Chaque filtre contient donc :

* soit un pattern spécifique non réutilisable,
* soit un pattern réutilisable se trouvant dans `/opt/logstash/patterns`,
* soit un pattern parmi ceux [fournis nativement](https://github.com/elasticsearch/logstash/tree/v1.4.2/patterns) par logstash.

Les patterns peuvent s'imbriquer les uns dans les autres pour élaborer des mappings de plus en plus complexes. Logstash fournit [un ensemble de patterns atomiques](https://github.com/elasticsearch/logstash/blob/v1.4.2/patterns/grok-patterns) permettant de faciliter la création d'autres patterns plus évolués.

### Création d'un filtre avec un pattern existant

Par exemple, pour créer un filtre structurant les données provenant des fichiers de log d'accès d'Apache, il est possible d'utiliser directement le pattern natif `COMBINEDAPACHELOG` :

```
# /etc/logstash/conf.d/12-apache-access.conf

filter {
  if [type] == "apache-access" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
}
```

### Création d'un filtre avec un patter personnalisé

Alors que pour créer un filtre structurant les données provenant des fichiers de logs d'erreurs d'Apache, il faut créer un tout nouveau pattern :

```
# /etc/logstash/conf.d/13-apache-error.conf

filter {
  if [type] == "apache-error" {
    grok {
      match => { "message" => "\[(?<timestamp>%{DAY:day} %{MONTH:month} %{MONTHDAY} %{TIME} %{YEAR})\] \[%{WORD:class}\] \[%{WORD:originator} %{IP:clientip}\] %{GREEDYDATA:errmsg}" }
    }
  }
}
```

Il est possible de déplacer ce pattern dans un nouveau fichier dans le répertoire `/opt/logstash/patterns` pour que les autres filtres puissent l'utiliser. Cela peut être très utile dans le cas où tous vos fichiers de logs applicatifs sont structurés de la même manière.

### Test des nouveaux patterns

Comme cela peut rapidement devenir assez illisible, un debugger est [disponible en ligne](http://grokdebug.herokuapp.com/) pour tester les patterns.

Ensuite, lorsque le filtre a été mis à jour avec un pattern valide, il peut être intéressant de le tester à son tour en [ajoutant la commande "configtest"](http://blog.stevenmeyer.co.uk/2014/06/add-configuration-test-to-logstash-service-configtest.html) au service `logstash` :

```
$ service logstash configtest
```


## Nettoyage des données

Il est possible d'effacer les données stockées dans Elasticseach avec [curator](https://github.com/elasticsearch/curator).

L'installation sur Debian 7 est simplissime :

```
$ sudo apt-get install python-pip
$ sudo pip install elasticsearch-curator
```

Ensuite, pour supprimer toute les données datant de plus d'un jour :

```
$ curator delete --older-than 1
```

Un ensemble d'options permet évidemment d'affiner le nettoyage.
