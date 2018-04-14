---
layout: post
title: 'Symfony Tips : Find form theme override block name'
description: Put a var_dump to display almost all block name you can use when theming your form
date: 2015-02-06
tags:
    - symfony
    - php
    - tips
---

I don't know if you are like me but I have a very hard time remembering how symfony and twig build the block name used in form theming. I often have
to read again the documentation : [How to customize form rendering](http://symfony.com/doc/current/cookbook/form/form_customization.html). And it has not
been enough for some specific use cases in my project.

So last time, I decided to track how Symfony build these names and find a way to display them in order to help me when theming my forms.

I think I found a pretty neat solution which could be useful for a lot of Symfony developer.

Just add a var_dump (temporary) in the `loadResourceForBlockName` method in the `TwigRendererEngine` class :

``` php
/**
* Loads the cache with the resource for a given block name.
*
* This implementation eagerly loads all blocks of the themes assigned to the given view
* and all of its ancestors views. This is necessary, because Twig receives the
* list of blocks later. At that point, all blocks must already be loaded, for the
* case that the function "block()" is used in the Twig template.
*
* @see getResourceForBlock()
*
* @param string   $cacheKey  The cache key of the form view.
* @param FormView $view      The form view for finding the applying themes.
* @param string   $blockName The name of the block to load.
*
* @return bool True if the resource could be loaded, false otherwise.
*/
protected function loadResourceForBlockName($cacheKey, FormView $view, $blockName)
{
    var_dump($blockName);

    ...
}
```

The file should be located in `vendor/symfony/symfony/src/Symfony/Bridge/Twig/Form/TwigRendererEngine.php`.

This is what you should see :

![](/assets/img/symfony/tips_form_block_name.png)

Now you can use any of these block names in your form theme twig file to style your forms.
