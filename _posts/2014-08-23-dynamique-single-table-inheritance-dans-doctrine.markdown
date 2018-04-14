---
layout: post
title: Dynamique single table inheritance dans Doctrine et Symfony2
description: Comment rendre les discriminator maps dynamiques dans Doctrine et Symfony2
date: 2014-08-23
lang: fr
tags:
    - symfony
    - doctrine
---

## Problématique

J'ai récemment eu besoin de rendre la configuration d'un héritage doctrine type [single table inheritance](http://doctrine-orm.readthedocs.org/en/latest/reference/inheritance-mapping.html#single-table-inheritance) paramétrable.
Ceci afin de respecter les principes [SOLID](http://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) pour permettre d'étendre le code sans le modifier.

## Solution

L'implémentation était en fait très simple. Dans une configuration sémantique d'un bundle, on propose à l'utilisateur du bundle de configurer le mapping souhaité :

Voici un extrait d'une configuration sémantique pouvant faire l'affaire :

```php
$treeBuilder = new TreeBuilder;
$rootNode = $treeBuilder->root('bundle_semantic_configuration_key');

$rootNode
    ->children()
        ->arrayNode('map')
            ->useAttributeAsKey('name')
            ->prototype('scalar')
            ->end()
        ->end()
    ->end()
;
```

Et dans l'extension de l'injecteur de dépendance :

```php
class BundleExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container)
    {
        $config = $this->processConfiguration(new Configuration(), $configs);
        $container->setParameter('bundle.map', $configs['map']);
    }
}
```

Maintenant, on crée un listeneur d'évènement sur le chargement des classmetadata doctrine.

D'abord le service :

``` yaml
bundle.map_discriminator.listener:
        class: Bundle\EventListener\MapDiscriminatorListener
        arguments:
            - '%bundle.map%'
        tags:
            - { name: doctrine.event_listener, event: loadClassMetadata }
```

Notez que ce que je décris ici peut être adapter à l'ODM Doctrine très rapidement pour fonctionner avec les documents MongoDB.

Ensuite le listener lui même :

```php
<?php

namespace Bundle\EventListener;

use Doctrine\ORM\Event\LoadClassMetadataEventArgs;

class MapDiscriminatorListener
{
    protected $map;

    public function __construct(array $map)
    {
        $this->map = $map;
    }

    public function loadClassMetadata(LoadClassMetadataEventArgs $event)
    {
        $metadata = $event->getClassMetadata();
        if ($classMetadata->getName() == 'MyClass/With/The/Single/Table/Inheritance') {
            $metadata->setDiscriminatorMap($this->map);
        }
    }
}
```

## Conclusion

Vous pouvez maintenant ajouter des lignes au discriminator sans modifier la classe de base.

Notez plusieurs éléments dans cette technique :

  * N'hésitez pas à modifier les noms indiqués ou la structure de la configuration sémantique
  * Un bundle existe implémentant cette solution [DCSDynamicDiscriminatorMapBundle](https://github.com/damianociarla/DCSDynamicDiscriminatorMapBundle)
  * Lorsqu'aucune DiscriminatorMap n'est configuré, Doctrine en génère une automatiquement.
  * Ici, nous écrason toute la configuration mais vous pouvez
utiliser addDiscriminatorMapClass pour ajouter des classes sans supprimer ce que vous avez configuré dans le yaml ou en annotation.
