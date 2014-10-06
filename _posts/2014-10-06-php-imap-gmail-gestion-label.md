---
layout: post
title: Gérer les labels Gmail en PHP avec l'extension IMAP
author: florent
---

A Xotelia nous avons mis en place un système qui est capable d'analyser des mails, de détecter des reservations et de les enregistrer dans notre système. Pour garder une boîte mail propre et s'y retrouver, nous avons décider de ranger les mails traités dans des labels Gmail en fonction des étapes de traitement. Après plusieurs essais, j'ai enfin trouvé le bon moyen d'assigner facilement des labels a ses mails au travers d'une connxion IMAP.

Tout d'abord pour se connecter à un compte mail en IMAP, on utilise l'extension PHP IMAP, au travers d'une petite lib `tedivm/fetch` qui permet de travailler en mode objet.

Je ne vais pas rentrer dans les détails de l'implémentation du protocol IMAP en PHP, qui est certes assez limitée par rapport à ce que permet le protocol, mais on s'en sort très bien avec fonctions qu'on a à disposition. Le point principal c'est que cette extension permet de travailler avec les mails de deux façons différentes. Soit avec le numéro de `sequence`, qui correspond à l'ordre du mails dans la boîte. Soit avec l'`UID`, qui est un identifiant unique définit par le serveur IMAP pour la boîte courante.

Le développeur de la lib qu'on utilise a fait le choix de travailler uniquement avec l'`UID`. C'est plus prudent car quelque soit l'ordre du mail dans la boîte, son `UID` reste identique. Mais seulement pour chaque boîte. C'est à dire que si on déplace ou on copie ce mail dans une autre boîte, il n'aura plus le même `UID`.

Au début des développements de ce projet, je pensais que cet `UID` était unique par rapport au compte et pas seulement unique par boîte. De plus suivant les implémentations du serveur IMAP, les dossiers peuvent être soit physique. C'est à dire qu'on se connectant à un compte UNIX, on peut se rendre physiquement dans le dossier à l'aide de la commande `cd` et lister simplement les messages via la commande `ls`. Mais ces dossiers peuvent aussi être virtuels, c'est à dire qu'ils sont gérés uniquement par le serveur IMAP. C'est par exemple le cas du dossier `Sent Mails`, ou sur Gmail, du dossier `All Mail`, qui en IMAP s'appelle `[Gmail]/All Mail`. Suivant la configuration du serveur ces dossiers peuvent contenir tout ou partie des mails présents.

Lors de la première implémentation de notre projet, une fois le mail récupérer sur la boîte mail, on l'archivait tout de suite. C'est à dire en IMAP, qu'on le déplaçait du dossier `INBOX` vers le dossier `[Gmail]/All Mail`. Le problème c'est que ce dossier d'archive, contient tous les mails présents sur le serveur dans toutes les boîtes. Et du coup son l'`UID` de chaque mail n'est pas le même dans la boîte de réception que dans le dossier d'archives. Du coup lorsque dans notre processus on était ammené à ajouter un label à un mail, on avait l'`UID` donné par IMAP venant du dossier `INBOX` pour aller chercher ce même mail dans le dossier d'archives. Et donc on se retrouvait à rajouter un label au mauvais mail.

Pour corriger ce problème, on ne lit désermais plus que les mails marqués comme non lus dans le dossier `INBOX`.

```php
<?php

$server = new \Fetch\Server($serverPath, $port, 'imap');

$messages = $server->search('UNSEEN', $limit);

foreach ($messages as $message) {
    // do stuff
}
```

Au début du traitement on les marque comme lu, mais ils restent dans le même dossier.

```php
<?php

$message->setFlag('seen');
```


Et seulement à la fin du traitement, on déplace ce mail dans le dossier d'archive.

```php
<?php

$message->moveToMailBox('[Gmail]/All Mail');
```

L'autre problème est que notre projet est découpé en plusieurs petit processus, qui font chacun une partie du traitement. Et la communication entre ces processus se fait via RabbitMQ. Un des processus s'occupe de lire la boîte mail, et de passer les mails aux processus concernés pour la suite du traitement. Lors de l'ajout de tag ou à la fin du traitement, lorsqu'il faut ajouter un label à un mail ou l'archiver, c'est un autre processus avec une autre connexion IMAP qui s'en occupe.

Chacun des processus est lancé par une tâche CRON. Et pour les processus qui ont besoin d'une connexion IMAP, elle est établie lors de l'initialisation du processus. De coup il se peut qu'un des processus qui doit rajouter un label par exemple à sa conenxion IMAP ouverte à un instant _t_, alors que le processus qui à récupérer le mails en cours de traitement s'est exécuté à l'instant _t_ + 1. Du coup lors de l'ajout de l'ajout du label ou de l'archivage le message a traiter n'existent pas à cet instant. Il faut donc forcer une réouverture de la connexion pour que le processus ait connaissance de ce message.

```php
<?php

imap_reopen($stream, 'INBOX');
```

Ainsi, lors de l'ajout de label, le processus à maintenant connaissance des messages dans la boîte et peut donc le récupérer via son `UID`. Pour rajouter un label, il suffit simplement de copier ce message dans le dossier du label. Car en IMAP, une boîte (INBOX), un label Gmail ou encore un dossier virtuel (Sent Mails) sont tous des dossiers.

```php
<?php

imap_mail_copy($stream, $message->getUid(), $label, CP_UID);
```

Voilà pour les gestion des mails et des labels IMAP en PHP. Une évolution possible de notre process de traitement serait de séparer les processus par métier (un pour la connexion IMAP, etc).
