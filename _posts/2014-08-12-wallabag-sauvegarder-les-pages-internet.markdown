---
layout: post
title: Wallabag - Sauvegarder les pages internet qui vous intéressent
description: Découvrez wallabag, une solution opensource PHP vous permettant de sauvegarder et conserver les pages internet
date: 2014-08-12
lang: fr
tags:
    - software
---

## Introduction

En lisant le linux pratique de juillet 2014, je suis tombé sur un article présentant [Framasoft](http://www.framasoft.net/) et son initiative de promotion des
logiciels libres. Un de ces logiciels m'a tout de suite attiré [Wallabag](https://www.wallabag.org/).

Je suis abonné à plusieurs mailing lists regroupant des articles de blog techniques que je bookmarke régulièrement pour une lecture future.
J'ai donc des dizaines de bookmarks dans mon navigateur vers ce type de lien sans aucun rangement ni aucune garantie qu'au moment de la lecture, ils
seront toujours disponibles. Ici intervient Wallabag. Ce logiciel permet de sauvegarder sans informations superflus le contenu des pages web que
vous saisissez. Je le considère un peu comme un mini archive.org à usage personnel.

Je vais présenter dans cet article comment installer wallabag, le configurer simplement et son utilisation de base.

## Installation et configuration

Téléchargez la dernière version stable depuis le site officiel : [téléchargez wallabag](https://www.wallabag.org/downloads/). Au moment de l'écriture de cet article
la dernière version stable est 1.7.2.

Décompressez l'archive et déployez les sources (Wallabag utilise composer pour la gestion des dépendances) :

```shell
unzip wallabag-1.7.2.zip
mv wallabag-1.7.2 /var/www/wallabag
cd /var/www/wallabag
curl -s http://getcomposer.org/installer | php
php composer.phar install
```

Afin d'autorisez le téléchargement des images en local lorsque vous sauvegardez une page, il faut modifier un paramètre de configuration. Editez le fichier
inc/poche/config.inc.php :

```php
@define ('DOWNLOAD_PICTURES', TRUE);
```

Donnez les droits à apache et assurez vous qu'il a les droits d'écriture dans les dossiers suivants :

```shell
chown -R www-data .
chmod -R u+w assets cache db
```

Configurez un VirtualHost apache avec comme DocumentRoot /var/www/wallabag. Voici un exemple fonctionnant pour moi dans le fichier /etc/apache2/site-enabled/wallabag.conf :

```apache
<VirtualHost *:80>
    ServerAdmin youremail
    ServerName hostname

    DocumentRoot /var/www/wallabag

    <Directory />
        Options FollowSymLinks
        AllowOverride None
    </Directory>

    <Directory /var/www/wallabag>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error-wallabag.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access-wallabag.log combined
</VirtualHost>
```

Redémarrez apache :

```shell
service apache2 restart
```

En vous rendant sur le nom de domaine configuré, vous devriez voir l'écran suivant :

![](/assets/img/wallabag/install.png)

Vous pouvez vous rendre sur la page suivante pour vérifier la configuration et la compatibilité du système http://hostname/wallabag_compatibility_test.php?from=install

Pour ma part, j'ai dû installer (j'ai décidé d'utiliser sqlite car je ne souhaitais pas installer mysql sur mon serveur juste pour wallabag) :

```shell
aptitude install php5-tidy php5-gd php5-sqlite php5-curl
service apache2 restart
```

Sur l'écran d'installation, renseignez le SGDB souhaité et configurez un login et mot de passe pour accéder à votre instance de wallabag et cliquez sur "Install wallabag"
Si tout s'est bien passé, une boîte d'alerte verte apparaît en haut de la page avec un lien "access it". Cliquez sur ce lien pour commencer à utiliser wallabag.

## Utilisation

Une fois authentifié, vous vous retrouvez sur l'écran suivant :

![](/assets/img/wallabag/home.png)

En colonne de gauche, cliquez sur le lien "Save a link". Une popin avec un formulaire apparaît :

![](/assets/img/wallabag/save_link.png)

Saisissez un lien (pour mes tests, j'ai utilisé : http://datacenteroverlords.com/2012/03/01/creating-your-own-ssl-certificate-authority/) et cliquez sur "Save link!"
Une fois l'article sauvegardé, le texte "Done!" apparaît sous la fenêtre. Vous pouvez la fermer et cliquez sur "Home" pour mettre à jour la liste d'article.

L'article apparaît maintenant dans votre wallabag et sera conservé indéfiniment.

## Conclusion

Cet article ne couvre que les bases des fonctionnalités fournies par wallabag :

* Vous pouvez configurer la langue
* Organiser vos pages par un mécanisme de tags
* Archiver les articles lus
* Chercher parmi vos articles
* etc ...

Bien sûr, Wallabag est totalement libre. Vous pouvez contribuer au projet directement sur le [repository Github](https://github.com/wallabag/wallabag).
