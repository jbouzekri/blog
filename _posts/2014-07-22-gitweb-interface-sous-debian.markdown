---
layout: post
title: Gitweb interface sous debian
description: Tutorial pour ajouter une interface web simple fournie par le projet git
date: 2014-07-22
lang: fr
tags:
    - git
    - linux
---

Ce tutoriel a été écrit en suivant les instructions de la [documentation officielle de git](http://git-scm.com/book/en/Git-on-the-Server-GitWeb).

Si vous n'avez pas de serveur git d'installé, vous pouvez lire mon précédent post à ce sujet : [Mise en place d'un serveur git sous debian](https://blog.bouzekri.net/2014-07-20-mise-en-place-d-un-serveur-git-sous-debian.html).

## Compiler et déployer gitweb interface

Dans la suite du post, je supposerais que le répertoire d'installation des repositories de votre serveur git est /home/git et que le DocumentRoot de votre interface web
sera le dossier /var/www/gitweb

Sur votre serveur, installer gitweb :

```shell
$ cd /opt
$ git clone git://git.kernel.org/pub/scm/git/git.git
$ make GITWEB_PROJECTROOT="/home/git" gitweb
$ make gitwebdir=/var/www/gitweb install-gitweb
```

Vous devriez voir un répertoire /var/www/gitweb contenant les fichiers suivants :

```shell
$ cd /var/www/gitweb
$ ls
gitweb.cgi  static
```

## Configurer apache

Je suppose que vous utilisez apache en mode virtualhost.

Assurez vous que le mode cgi pour apache est activé :

```shell
a2enmod cgi
```

Ajouter un virtualhost pour votre interface gitweb :

```shell
$ vim /etc/apache2/site-available/gitweb.conf
```

```apache
<VirtualHost *:80>
    ServerName yourhostname

    DocumentRoot /var/www/gitweb

    <Directory "/var/www/gitweb">
        Options Indexes FollowSymlinks ExecCGI
        AddHandler cgi-script .cgi .pl
        DirectoryIndex gitweb.cgi
        AllowOverride None
        Order allow,deny
        Allow from all
    </Directory>

    ErrorLog /var/log/apache2/error-gitweb.log
    CustomLog /var/log/apache2/access-gitweb.log combined
</VirtualHost>
```

Activer le virtualhost et redémarrez apache.

```shell
$ cd /etc/apache2/site-enabled
$ ln -s ../site-available/gitweb.conf
$ service apache2 restart
```

Rendez vous avec votre navigateur sur votre hostname : http://yourgitwebhostname. Vous devriez voir l'interface gitweb avec la liste de vos projets.

## Troubleshooting

L'interface gitweb ne trouve pas vos projets :

* gitweb ne trouve pas l'exécutable git :
Editer /var/www/gitweb/gitweb.cgi et modifier:
```perl
our $GIT = "git"
```

* gitweb ne trouve pas le dossier contenant vos repositories :
Editer /var/www/gitweb/gitweb.cgi et modifier:
```perl
our $projectroot = "/home/git";
our $projects_list = $projectroot;
```

* gitweb exécuté par votre serveur web n'a pas les droits en lecture sur vos repositories.
Ajouter l'utilisateur www-data au groupe git (ou tout autre groupe du répertoire contenant vos dépôts).

```shell
$ usermod -aG git www-data
```

Attention à bien vérifier que vos repositories ont les bons droits :
```shell
$ chown git:git /home/git -R
$ chmod ug+rx /home/git -R
```

## Conclusion

Vous avez désormais une interface web permettant de naviguer rapidement parmi vos dépôts git. Elle reste très light. Si vous avez de plus gros besoins en administration
sous forme d'interfaces ou que vous trouvez cette interface trop légère graphiquement, plusieurs alternatives listées sur le site de git existent : [Liste des interfaces web git](https://git.wiki.kernel.org/index.php/InterfacesFrontendsAndTools#Web_Interfaces)

Notez que nous n'avons pas sécurisé cette interface, cette dernière à travers un vhost apache, vous êtes libre de rajouter une authentification et un certificat SSL.
