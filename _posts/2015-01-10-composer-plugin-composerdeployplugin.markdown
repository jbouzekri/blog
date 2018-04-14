---
layout: post
title: Composer plugin et mon plugin ComposerDeployPlugin
description: Créer un plugin pour composer et présentation de mon plugin pour aider au déploiement
date: 2015-01-10
lang: fr
tags:
    - composer
    - php
---

## 1.Introduction

Tous les développeurs Symfony ont utilisé au moins une fois la commande assets:install. Cette dernière permet de déployer dans le dossier publique accessible par le serveur web les fichiers
statiques (assets) disponibles dans les extensions du développeurs. (afin de rendre les fichiers accessibles par les internautes).

Dans un projet, j'ai eu besoin de faire exactement la même opération avec une librairie installée à l'aide de composer. Afin de réaliser une commande réutilisable,
je me suis tourné vers le système de plugin composer.

Dans ce post, je vais décrire le système de plugin composer puis présenter mon plugin de déploiement.

## 2. Le système de plugin de composer

Ce chapitre est quasiment une traduction de la [documentation officielle des plugins composer](https://getcomposer.org/doc/articles/plugins.md).

Composer propose un mécanisme de plugin permettant d'altérer ou d'étendre ses fonctionnalités. Un plugin n'est rien de plus qu'un package installé via votre
fichier composer.json.

### 2.1 composer.json du plugin

Le fichier composer.json du plugin est quasiment identique à un fichier classique. Les seules contraintes sont :

* `type` doit avoir la valeur `composer-plugin`
* `extra` doit contenir une clé `class` qui contient le nom de la classe du plugin (avec son namespace complet)
* `require` doit inclure le package spécial `composer-plugin-api` à sa version `1.0.0`

### 2.2 La classe du plugin

La classe du plugin référencée dans la clé `extra.class` de votre composer.json doit implémenter l'interface `Composer\Plugin\PluginInterface`. Cette
interface vous donne la méthode `activate` qui est appelée lorsque le plugin est chargé. Elle reçoit une instance de `Composer\Composer` et de `Composer\IO\IOInterface`.
A l'aide de ces 2 objets, vous avez accès à toute la configuration chargée et les objets internes instanciés. Leurs états respectifs peuvent bien sûr être manipulés.

```php
<?php

namespace phpDocumentor\Composer;

use Composer\Composer;
use Composer\IO\IOInterface;
use Composer\Plugin\PluginInterface;

class TemplateInstallerPlugin implements PluginInterface
{
    public function activate(Composer $composer, IOInterface $io)
    {
        $installer = new TemplateInstaller($io, $composer);
        $composer->getInstallationManager()->addInstaller($installer);
    }
}
```

*Note : La méthode activate est appelée au chargement du plugin. Ce dernier intervient quasiment au début du flow d'exécution de composer. A cet instant, l'autoloader php proposé par composer n'a pas encore été exécuté. Vous ne pourrez donc pas instancier de classes provenant des dépendances de votre plugin sans inclures les fichiers sources correspondants.*

Je souhaite mettre l'accent sur 2 fonctionnalités intéressantes :

* Configurer son plugin
* Enregistrer des listeners d'évènement dans son plugin

Configurer son plugin est extrèmement simple, `Composer\Composer` vous donne accès à toute la configuration de composer. Vous pouvez donc accéder à
la clé `extra` du fichier `composer.json` de votre projet.

Je vous conseille de définir un namespace :

```json
{
    "extra": {
        "my-composer-plugin": {
            "key1": "value1",
            "key2": "value2"
        }
    }
}
```

Vous pouvez ensuite accéder à ces valeurs à l'aide de l'objet Composer :

```php
$key1 = $composer->getPackage()->getExtra()['my-composer-plugin']['key1'];
```

Un plugin peut de plus être utilisé pour écouter sur certains évènements lancés par composer. Par exemple, vous pouvez exécuter du code après chaque update ou chaque install.

Pour ce faire, votre classe de plugin doit implémenter `Composer\EventDispatcher\EventSubscriberInterface`. Elle pourra ainsi enregistrer des handlers d'évènement sur :

* COMMAND : évènement lancé au début de toutes les commandes
* PRE_FILE_DOWNLOAD : évènement lancé avant le téléchargement de fichiers
* [Tous les évènements de scripts](https://getcomposer.org/doc/articles/scripts.md#event-names)

Exemple :

```php
<?php

namespace Naderman\Composer\AWS;

use Composer\Composer;
use Composer\EventDispatcher\EventSubscriberInterface;
use Composer\IO\IOInterface;
use Composer\Plugin\PluginInterface;
use Composer\Plugin\PluginEvents;
use Composer\Plugin\PreFileDownloadEvent;

class AwsPlugin implements PluginInterface, EventSubscriberInterface
{
    protected $composer;
    protected $io;

    public function activate(Composer $composer, IOInterface $io)
    {
        $this->composer = $composer;
        $this->io = $io;
    }

    public static function getSubscribedEvents()
    {
        return array(
            PluginEvents::PRE_FILE_DOWNLOAD => array(
                array('onPreFileDownload', 0)
            ),
        );
    }

    public function onPreFileDownload(PreFileDownloadEvent $event)
    {
        $protocol = parse_url($event->getProcessedUrl(), PHP_URL_SCHEME);

        if ($protocol === 's3') {
            $awsClient = new AwsClient($this->io, $this->composer->getConfig());
            $s3RemoteFilesystem = new S3RemoteFilesystem($this->io, $event->getRemoteFilesystem()->getOptions(), $awsClient);
            $event->setRemoteFilesystem($s3RemoteFilesystem);
        }
    }
}
```

## 3. Le plugin ComposerDeployPlugin

En utilisant les informations de cette page, j'ai créé un plugin permettant de déployer (copier ou créer des liens symboliques) depuis le dossier vendor
vers un dossier de votre choix. Ce plugin est disponible sur [github](https://github.com/jbouzekri/ComposerDeployPlugin) et sous license [MIT](https://github.com/jbouzekri/ComposerDeployPlugin/blob/master/LICENSE).

La documentation sur github est complète, je ne présenterais donc qu'un use case.

Voici une configuration simple pour un projet dépendant d'une librairie proposant des fichiers css ou js (supposons que cette librairie ne dispose pas de bower.json) :

```json
{
    "require": {
        "librairie/test": "~2.2,>=2.2.3",
        "jbouzekri/composer-deploy-plugin": "~1.0"
    },
    "extra": {
        "jb-composer-deploy": {
            "target-dir": "web/composed",
            "folders": [
                "js",
                "css"
            ],
            "symlink": true,
            "relative": false
        }
    }
}
```

Cette configuration permet de créer 2 liens symboliques absolus dans le dossier `web/composer/librairie-test` :

* `web/composer/librairie-test/js` -> `vendor/librairie/test/js`
* `web/composer/librairie-test/css` -> `vendor/librairie/test/css`

Ainsi vos fichiers statiques peuvent être rendus accessibles à vos internautes en les déployant automatiquement à chaque composer update or install dans le
DocumentRoot de votre projet.
