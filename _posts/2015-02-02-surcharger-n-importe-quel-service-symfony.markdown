---
layout: post
title: Comment surcharger n'importe quel service du containeur Symfony
description: Utiliser les pass de compilations pour surcharger les services rendus non extensibles dans la configuration du bundle
date: 2015-02-02
lang: fr
tags:
    - symfony
    - php
---

## Introduction

Dans les bundles communautaires, plusieurs bonnes pratiques permettent d'assurer que son code sera facilement extensible.

Par exemple, les noms de classe sont définis dans des paramètres :

```yml
parameters:
    my_service.class: My\Service

services:
    my_service:
        class: '%my_service.class%'
```

Ou alors, les services injectés/utilisés que l'on souhaite rendre extensibles sont configurables par la configuration syntaxique du bundle :

```yml
my_bundle:
    injectable_service_id: my_service
```

```php
class MyController extends Controller
{
    public function indexAction()
    {
        $this->container->get($this->container->getParameter('my_bundle.injectable_service_id'));
    }
}
```

Ou encore, une configuration syntaxique propose un mécanisme de factory de services. J'ai écrit un billet de blog à ce sujet récemment :
[Service factory dans une extension du containeur](http://blog.bouzekri.net/2014/12/20/service-factory-extension-container-symfony/)

Mais il peut arriver que vous tombiez sur un bundle ne proposant pas ces mécanismes. Après avoir fouillé dans le code du bundle, vous faites
une pull request mais cette dernière prend du temps à être acceptée et la deadline de votre projet approche ... Pas d'inquiétude, tous les
services de tous les bundles sont modifiables dans Symfony.

## Le chargement du container Symfony

Pour bien comprendre cette technique de surcharge, attardons nous sur le fonctionnement du container d'injection de dépendance de Symfony
et plus particulièrement sa compilation. Je vous invite à lire la documentation officielle pour plus de
détail : [http://symfony.com/fr/doc/current/components/dependency_injection/index.html](http://symfony.com/fr/doc/current/components/dependency_injection/index.html)

Nous parlons de compilation du container lorsque ce dernier agrège les données de tous les bundles du projet, les valide et les met en cache.

La première étape de cette compilation est la lecture des extensions enregistrées. Dans 99% des cas, vous déclarez vos services dans
des fichiers de configuration que vous référencez dans une extension du container. Ce dernier charge chaque extension les unes
après les autres dans l'ordre dans lequel vous les avez enregistré (dans le fichier AppKernel). Dans ces extensions, le scope
de vos services est restreint à celui du bundle. Vous ne pourrez pas agir sur des services externes à votre extension.

Puis vient la phase des phases de compilation, le container applique les passes de compilation qui ont été référencées. A cette étape,
la totalité des services de votre application et du framework sont accessibles. Cette étape est classiquement utilisée pour lister les
services taggés. Mais vous remarquerez que ces services sont positionnés dans une multitude de bundle et vous avez accès à leur configuration.
Du coup, rien ne vous empêche dans ces passes de compilation de modifier leur définition à la volée et ainsi de pouvoir :

* changer la classe instanciée
* Ajouter, modifier ou retirer des arguments
* ajouter des calls à l'instanciation
* …

## Un exemple de surcharge changeant la classe d'un service

Supposons que vous chargiez un bundle OtherBundle proposant le service other_bundle.service mais sans class parameter vous permettant
de surcharger la classe instanciée.

Cependant, une méthode de cette classe ne fait pas exactement le traitement souhaité dans votre cas. Vous pourriez copier la totalité du
bundle pour changer juste cette classe mais en prenant 5 minutes vous pourrez changer la classe instanciée à la volée avec une passe de
compilation personnalisée.

Tout d'abord, créez votre passe de compilation. Je ne rentrerais pas dans les détails, il s'agit du même type de classe que vous instanciez lorsque
vous utilisez des services taggés :
[http://symfony.com/fr/doc/current/components/dependency_injection/tags.html#creer-une-compilerpass-passe-de-compilateur-en-francais](http://symfony.com/fr/doc/current/components/dependency_injection/tags.html#creer-une-compilerpass-passe-de-compilateur-en-francais)

Dans votre passe de compilation, cherchez le service concerné et modifiez la classe à instancier :

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\Reference;

class CustomCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container)
    {
        if (!$container->hasDefinition('other_bundle.service')) {
            return;
        }

        $definition = $container->getDefinition(
            'other_bundle.service'
        );

        $definition->setClass('MyBundle\\Service\\CustomClass');
    }
}
```

Votre CustomClass peut hériter de l'ancienne pour changer le fonctionnement de la méthode souhaitée par héritage.

En ayant accès à la définition du service, vous pouvez bien sûr changer la totalité de cette dernière. Cette page de la documentation
vous expliquera comment modifier les arguments, ajouter des calls à l'instanciation, ... :
[http://symfony.com/fr/doc/current/components/dependency_injection/definitions.html#travailler-avec-une-definition](http://symfony.com/fr/doc/current/components/dependency_injection/definitions.html#travailler-avec-une-definition)
