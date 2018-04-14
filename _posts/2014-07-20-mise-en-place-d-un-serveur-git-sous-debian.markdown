---
layout: post
title: Mise en place d'un serveur git sous debian
description: Tutorial pour installer un serveur git pas à pas sous debian
date: 2014-07-20
lang: fr
tags:
    - git
    - linux
---

Ce tutoriel a été écrit en suivant les instructions de la [documentation officielle de git](http://git-scm.com/book/en/Git-on-the-Server-Setting-Up-the-Server).

## Préparer le serveur

Si git n'est pas installé sur votre serveur, vous pouvez l'installer en utilisant
 le gestionnaire de package en tant que root :

```shell
apt-get install git
```

Pour des raisons de sécurité, le serveur git aura un utilisateur dédié. Dans un premier
temps, nous utiliserons "git over ssh", vous devrez donc avoir une clé ssh publique de
disponible qui sera utilisée pour vous donner accès au serveur git.

Sur le serveur, ajouter l'utilisateur git. En tant que root :

```shell
adduser git
su git
cd
mkdir .ssh
```

Depuis votre poste, en étant authentifié avec l'un des utilisateurs autorisé à utiliser
votre serveur git, ajouter la clé publique ssh aux clés autorisées :

```shell
ssh-copy-id git@yourserverhostname
```

## Initialiser et manipuler votre premier repository

Sur le serveur, initialiser un repository vide :

```shell
su git
cd
mkdir myproject.git
cd myproject.git
git --bare init
```

Vous pouvez dès à présent depuis votre poste, cloner ce repository et faire vos premiers commits :

```shell
git clone git@yourserverhostname:/home/git/myproject.git
touch README
git add README
git commit -m "initial commit"
git push origin master
```

Note 1 : Pour tester que tout va bien, vous pouvez refaire un clone dans un autre dossier pour vérifier que votre
fichier README est bien présent.

Note 2 : Si vous utilisez un serveur SSH écoutant sur un port différent du port standard, la commande clone à utiliser
est la suivante : git clone ssh://git@yourserverhostname:yourport/home/git/myproject.git

Un peu de sécurité maintenant, votre utilisateur git est un utilisateur normal dans le sens UNIX. Il a donc accès à un
shell et à tous les exécutables qui vont avec. Git fournit un shell personnalisé qui restraint les commandes disponibles à celles
utiles pour gérer un serveur git.

Vérifier le path de l'utilitaire git-shell sur votre serveur :

```shell
which git-shell
```

En root, éditer le fichier des comptes locaux UNIX.

```shell
vim /etc/passwd
```

A la fin du fichier, vous devriez voir une ligne proche celle ci :

```shell
git:x:1003:1003:,,,:/home/git:/bin/bash
```

Remplacer /bin/bash par le résultat de la requête which git-shell. Pour moi : /usr/bin/git-shell. Ce qui devrait vous donner :

```shell
git:x:1003:1003:,,,:/home/git:/usr/bin/git-shell
```

Note : Pour tester que vous avez amélioré la sécurité de votre serveur, essayez de vous connecter en SSH avec l'utilisateur
git maintenant :

```shell
$ ssh git@yourserverhostname
fatal: Interactive git shell is not enabled.
hint: ~/git-shell-commands should exist and have read and execute access.
Connection to yourserverhostname closed.
```

## Conclusion

Vous avez dès à présent un serveur git prêt à l'emploi. Un clone d'un repository sera préconfiguré avec le bon remote.
Vous pouvez donc travailler en collaboration avec plusieurs utilisateurs pour peu que vous ayez uploadé leur clé publique ssh.

Notez que si vous avez besoin de rajouter un nouveau dépôt, vous devez quand même avoir un accès SSH à votre serveur.
De plus, vous ne pouvez plus utiliser l'utilisateur git directement. N'oubliez donc pas de faire un chown sur chaque nouveau repository pour
l'assigner à l'utilisateur git.

Pour information, j'ai créé un petit script que j'utilise pour initialiser mes dépôts avec les bons droits. Vous pouvez le télécharger sur ce lien [git-create-repo gist](https://gist.github.com/jbouzekri/35481a7c087bac61758c)
