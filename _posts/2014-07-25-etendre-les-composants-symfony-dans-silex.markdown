---
layout: post
title: Etendre les composants de base Symfony dans Silex
description: Tutorial pour étendre certains composants de base Symfony à l'aide du container Pimple dans Silex
date: 2014-07-25
lang: fr
tags:
    - silex
    - symfony
    - pimple
---

Sur mon site personnel, je souhaitais ajouter un champ de formulaire de type captcha en utilisant la solution de Google Recaptcha. Malheureusement, le provider Silex
de la communauté n'utilisait pas les types de formulaire symfony ce qui ne me convenait pas. Au contraire, le Bundle Symfony Recaptcha les utilisait mais n'avait pas
de bridge pour Silex. J'ai donc décidé de construire ce bridge le plus proprement possible.

Ce bridge Silex pour Recaptcha est disponible dans le Bundle [EWZRecaptchaBundle](https://github.com/excelwebzone/EWZRecaptchaBundle). Vous pouvez accéder
directement au Bridge en cliquant sur [ce lien](https://github.com/excelwebzone/EWZRecaptchaBundle/tree/master/Bridge).

Dans ce post, je vais décrire pour chacun des composants rencontrés lors de la réalisation de ce bridge la façon dont je les ai étendu. Bien entendu, ces bouts
de code seront à adapter à vos projets (en particulier les noms de fichier, les chemins, ...).

De plus ces exemples impliquent que les composants Symfony concernés sont activés (leur provider Silex est appelé dans le bootstrap de votre projet).

## Etendre Twig

Si votre Bundle Silex fournit un dossier de views twig. Vous pouvez configurer à travers votre service provider le service twig pour qu'il charge le dossier de votre
Bundle.

```php
if (isset($app['twig'])) {
    $path = dirname(__FILE__).'/../Resources/views/Form';
    $app['twig.loader']->addLoader(new \Twig_Loader_Filesystem($path));

    $app['twig.form.templates'] = array_merge(
        $app['twig.form.templates'],
        array('ewz_recaptcha_widget.html.twig')
    );
}
```

Notez que ce bout de code enregistre également un template de formulaire global.

### Enregistrer un fichier de traduction

```php
 if (isset($app['translator'])) {
    $app['translator']->addResource(
        'xliff',
        dirname(__FILE__).'/../Resources/translations/messages.'.$app['locale'].'.xlf',
        $app['locale'],
        'validators'
    );
}
```

Ce bout de code enregistre un nouveau fichier de traduction en fonction de la locale contenue dans le container Pimple.

## Enregistrer un type de formulaire

```php
// Form type extension class
namespace EWZ\Bundle\RecaptchaBundle\Bridge\Form\Extension;

use Silex\Application;
use Symfony\Component\Form\AbstractExtension;
use EWZ\Bundle\RecaptchaBundle\Form\Type\RecaptchaType;

/**
 * Extends form to register captcha type
 */
class RecaptchaExtension extends AbstractExtension
{
    /**
     * Container
     *
     * @var \Silex\Application
     */
    private $app;

    /**
     * Constructor
     *
     * @param \Silex\Application $app container
     */
    public function __construct(Application $app)
    {
        $this->app = $app;
    }

    /**
     * Register the captche form type
     *
     * @return array
     */
    protected function loadTypes()
    {
        return array(
            new RecaptchaType(
                $this->app['ewz_recaptcha.public_key'],
                $this->app['ewz_recaptcha.enabled'],
                $this->app['ewz_recaptcha.locale_key']
            )
        );
    }
}

// Extending the extension of the form
if (isset($app['form.extensions'])) {
    $app['form.extensions'] = $app->share($app->extend('form.extensions',
        function($extensions) use ($app) {
            $extensions[] = new Form\Extension\RecaptchaExtension($app);

            return $extensions;
    }));
}
```

Cet exemple enregistre un nouveau type de formulaire dans le composant form Symfony. Vous pouvez ensuite utiliser le type de formulaire avec son nom (getName)
dans vos formulaires directement sans l'instancier.

2 étapes :

  * Créez une classe d'extension de type listant les types fournis par votre bundle
  * enregistrer votre extension dans le composant form.

## Enregistrer une contrainte de validation

```php
if (isset($app['validator.validator_factory'])) {
    $app['ewz_recaptcha.true'] = $app->share(function ($app) {
        $validator = new TrueValidator(
            $app['ewz_recaptcha.enabled'],
            $app['ewz_recaptcha.private_key'],
            $app['request_stack']
        );

        return $validator;
    });

    $app['validator.validator_service_ids'] = isset($app['validator.validator_service_ids']) ? $app['validator.validator_service_ids'] : array();
    $app['validator.validator_service_ids'] = array_merge(
        $app['validator.validator_service_ids'],
        array('ewz_recaptcha.true' => 'ewz_recaptcha.true')
    );
}
```

Dans cet exemple, j'enregistre une nouvelle contrainte de validation. C'est effectué en 2 étapes :

  * On crée un service définissant le validator
  * Dans le tableau validator.validator_service_ids, on référence ce service par le nom donné

La contrainte de validation peut ensuite être instanciée normalement dans votre formulaire. Le validator sera appliqué.

Exemple : on utilise la contrainte dans la configuration de la validation du formulaire qui déclenchera le validator configuré en service.
```php
$app['form.factory']
    ->createBuilder('form')
    ->add('captcha', 'ewz_recaptcha', array(
        'constraints' => new CaptchaTrue(),
        'attr' => array(
            'options' => array(
                'theme' => 'clean'
            )
        )
    ))
```
