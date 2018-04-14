---
layout: post
title: Configurer et créer dynamiquement des services dans une extension du container Symfony
description: Créer une configuration de bundle extensible permettant de créer des définitions de service dynamiques
date: 2014-12-20
lang: fr
tags:
    - symfony
---

## Introduction

En jouant avec les bundles [KnpGaufretteBundle](https://github.com/KnpLabs/KnpGaufretteBundle) et [LiipImagineBundle](https://github.com/liip/LiipImagineBundle), j'ai
remarqué qu'ils permettaient de créer dynamiquement des définitions de service. Les utilisateurs peuvent créer une nouvelle configuration qui sera
mappée sur un service et disponible dans le container.

```yaml
knp_gaufrette:
    adapters:
        foo:
            local:
                directory: /path/to/my/filesystem
        testamazon:
            amazon_s3:
                amazon_s3_id:   amazonS3
                bucket_name:    foo_bucket
                options:
                    directory:  foo_directory
```

Par exemple, ici Knp permet de créer 2 services de type adapter dans le container Symfony. Un premier foo de type local, un second testamazon de type amazon S3.

Cet article a pour but de décrire comment mettre en place ce type de configuration dans son bundle.
Pour cet exemple, je supposerais qu'on a besoin de pouvoir instancier rapidement plusieurs types de resolver.
Par exemple, on peut vouloir traduire un nom de fichier en url. L'url changera en fonction du type de service : amazon s3, liipimagine, assets, ...

## 1. Définir l'extension de configuration

Cette extension devra être capable de :

* D'ajouter de nouveau type de resolvers. Nous aurons donc besoin d'une factory de service par type de resolver
* Permettre à un utilisateur de configurer plusieurs définition de resolvers. Nous aurons besoin d'une configuration permettant à un utilisateur d'ajouter des services.

La gestion des factory de service est assez classique. Il s'agira de services implémentant une classe ResolverFactoryInterface et taggués example.resolver.factory.
La différence avec une pass de compilateur classique est qu'elle sera réalisée dans un container temporaire pour permettre leur utilisation dans le containeur du bundle concerné.

MyBundle\DependencyInjection\ResolverFactoryConfiguration.php :

```php
class ResolverFactoryConfiguration implements ConfigurationInterface
{
    /**
     * @return TreeBuilder
     */
    public function getConfigTreeBuilder()
    {
        $treeBuilder = new TreeBuilder();

        $treeBuilder
            ->root('example')
                ->ignoreExtraKeys()
                ->fixXmlConfig('resolver_factory', 'resolver_factories')
                ->children()
                    ->arrayNode('resolver_factories')
                        ->defaultValue(array())
                        ->prototype('scalar')->end()
                    ->end()
                ->end()
            ->end()
        ;

        return $treeBuilder;
    }
}
```

Il s'agit ici d'une configuration classique permettant à un utilisateur de référencer une liste de service dans la clé resolver_factories de la configuration du bundle.

MyBundle\DependencyInjection\MyBundleExtension.php :

```php
public function load(array $configs, ContainerBuilder $container)
{
    $processor = new Processor();
    $config    = $processor->processConfiguration($new ResolverFactoryConfiguration(), $configs);
    $factories = $this->createResolverFactories($config, $container);

    // Continuer le chargement d'une extension de containeur classique
    ...
}

protected function createResolverFactories(array $configs, ContainerBuilder $container)
{
    if (null !== $this->factories) {
        return $this->factories;
    }

    // load bundled resolver factories
    $tempContainer = new ContainerBuilder();
    $parameterBag  = $container->getParameterBag();
    $loader        = new Loader\YamlFileLoader($tempContainer, new FileLocator(__DIR__.'/../Resources/config'));
    $loader->load('resolver_factories.yml');

    // load user-created resolver factories
    foreach ($configs['resolver_factories'] as $factory) {
        $loader->load($parameterBag->resolveValue($factory));
    }

    $services  = $tempContainer->findTaggedServiceIds('example.resolver.factory');

    $factories = array();
    foreach (array_keys($services) as $id) {
        $factory = $tempContainer->get($id);
        $factories[str_replace('-', '_', $factory->getKey())] = $factory;
    }

    return $this->factories = $factories;
}
```

Que se passe-t-il ici ?

Avant de faire le chargement classique d'une configuration de bundle, nous validons la clé resolver_factories configurée dans la classe précédente et
nous rentrons dans la méthode createResolverFactories.

Cette méthode crée un container temporaire et charge d'abord le contenu d'un fichier resolver_factories.yml dans le bundle. Ce fichier référence la liste
des factory de définition de service resolver fournies nativement par le bundle.

Exemple :

```yaml
services:
    example.resolver_factory.asset:
        class: "%example.resolver_factory.asset.class%"
        tags:
            - { name: example.resolver.factory }

    example.resolver_factory.imagine:
        class: "%example.resolver_factory.imagine.class%"
        tags:
            - { name: example.resolver.factory }
```

Il ajoute ensuite le contenu de la configuration resolver_factory définie par l'utilisateur (permettant ainsi à ce dernier de rajouter manuellement
de nouvelles factory).

Une fois ces dernières chargées, il cherche tous les services taggués example.resolver.factory et liste ces dernières dans un tableau.

## 2. Valider des configurations de service dynamiquement

Nous voulons que notre utilisateur puisse configurer plusieurs resolver de différents types. Par exemple, un resolver de type assets aurait un
paramètre de configuration directory tandis qu'un autre de type amazon s3 aurait un paramètre de configuration bucket.

```yaml
example:
    resolvers:
        upload:
            assets:
                directory: uploads
        crop:
            assets:
                directory: uploads/croped
        distant:
            aws3:
                bucket: mybucket
```

Chaque type de service aura une configuration différente, il faut donc que notre validateur/normaliseur de configuration soit dynamique.
Nous disposons déjà d'un tableau de factory de service avec une factory pour chaque type. Alors pourquoi ne pas ajouter une méthode à chaque factory
définissant la configuration acceptée ?

Exemple pour assets :

```php
public function addConfiguration(ArrayNodeDefinition $node)
{
    $node
        ->children()
            ->scalarNode('directory')->isRequired()->cannotBeEmpty()->end()
        ->end()
    ;
}
```

Il faut maintenant ajouter ces configurations à celle du bundle :

```php
public function load(array $configs, ContainerBuilder $container)
{
    // Traitement des factory de service resolver
    ...

    // Chargement et validation classique de la configuration du bundle
    $processor = new Processor();
    $processor->processConfiguration($new MainConfiguration($factories), $configs);
}
```

Notez que nous passons le tableau de factories de service resolver à la configuration du bundle.

```php
class MainConfiguration implements ConfigurationInterface
{
    /**
     * @var array
     */
    protected $factories;

    /**
     * Constructor
     *
     * @param array $factories
     */
    public function __construct(array $factories)
    {
        $this->factories = $factories;
    }

    public function getConfigTreeBuilder()
    {
        $treeBuilder = new TreeBuilder();
        $rootNode = $treeBuilder->root('example');

        $this->addResolversSection($rootNode, $this->factories);

        // Autre configuration du bundle
        ...
    }

    protected function addResolversSection(ArrayNodeDefinition $node, array $factories)
    {
        $resolverNodeBuilder = $node
            ->fixXmlConfig('resolver')
            ->children()
                ->arrayNode('resolvers')
                    ->useAttributeAsKey('name')
                    ->prototype('array')
                    ->performNoDeepMerging()
                    ->children()
        ;

        foreach ($factories as $name => $factory) {
            $factoryNode = $resolverNodeBuilder->arrayNode($name)->canBeUnset();

            $factory->addConfiguration($factoryNode);
        }
    }
```

Ce bout de code permet donc de charger pour chaque factory la configuration autorisée.
Notez l'appel à canBeUnset. Ainsi un utilisateur peut se contenter de configurer les informations réellement au type concerné sans information superflue.

## 3. Créer les définitions de service dynamique configurées par l'utilisateur

Une fois les factory de service configurées, on peut traiter la configuration des services eux mêmes.

Reprenons, la configuration de service précédente, toujours dans l'extension du containeur du bundle, on traite ces configurations de resolvers :

```php
public function load(array $configs, ContainerBuilder $container)
{
    // Traitement des factory de service resolver
    ...

    // Chargement et validation classique de la configuration du bundle
    ...

    $resolvers = array();
    foreach ($config['resolvers'] as $name => $resolver) {
        $resolvers[$name] = $this->createResolver($name, $resolver, $container, $factories);
    }
}

protected function createResolver($name, array $config, ContainerBuilder $container, array $factories)
{
    foreach ($config as $key => $resolver) {
        if (array_key_exists($key, $factories)) {
            $id = sprintf('example.%s_resolver', $name);
            $factories[$key]->create($container, $id, $resolver);

            return $id;
        }
    }

    throw new \LogicException(sprintf('The resolver \'%s\' is not configured.', $name));
}
```

Pour chaque resolver configuré, on cherche une factory de resolver et on appelle la méthode create de cette dernière.
L'objectif étant d'obtenir un tableau listant des ids de service resolver associé à un alias configuré par l'utilisateur.
Pour le premier de notre exemple, l'alias serait upload et l'id du service serait example.upload_resolver.

Chaque factory implémente donc la méthode create qui permet de créer dynamiquement la définition de service configurée par l'utilisateur :

```php
public function create(ContainerBuilder $container, $id, array $config)
{
    $container
        ->setDefinition($id, new DefinitionDecorator('example.resolver.asset.prototype'))
        ->setScope('request')
        ->addArgument($config['directory'])
    ;
}
```

Qui serait équivalent à un utilisateur créant un service :

```yaml
example.upload_resolver:
    parent: example.resolver.asset.prototype
    scope: request
    arguments:
        - 'uploads'
```

## 4. On référence toutes les définitions dans un service chain

Afin de permettre leur utilisation dans le code de notre bundle, le plus simple est de les ajouter tous à un service chain comme le fait le code
d'exemple de la [documentation symfony sur les services taggués](http://symfony.com/fr/doc/current/components/dependency_injection/tags.html).

```php
public function load(array $configs, ContainerBuilder $container)
{
    // Traitement des factory de service resolver
    ...

    // Chargement et validation classique de la configuration du bundle
    ...

    // Création des définition de service resolver
    ...

    $loader = new Loader\YamlFileLoader($container, new FileLocator(__DIR__.'/../Resources/config'));
    $loader->load('services.yml');
    ...

    $resolverChain = $container->findDefinition('example.resolver_chain');

    foreach ($resolvers as $name => $resolver) {
        $resolverChain->addMethodCall('addResolver', array(new Reference($resolver), $name));
    }
```

Chaque resolver est donc ajouté avec l'alias configuré par l'utilisateur au service example.resolver_chain pour utilisation dans le code du bundle.

## Conclusion

Cette structure de configuration du containeur Symfony est assez longue à mettre en place mais permet de créer des bundles extensibles. Vos utilisateur
auront la possibilité d'utiliser les fonctionnalités proposées par vos bundles sans coder une ligne de code en utilisant simplement la configuration du bundle.
