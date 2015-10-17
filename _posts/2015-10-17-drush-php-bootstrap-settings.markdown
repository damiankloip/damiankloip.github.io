---
layout: post
title: Drush PHP Settings
date: 2015-10-17
tags:
  - drush
  - repl
---

My previous post gave an introduction to the `drush php` command and [PsySH][psysh].

I have not had a chance to look at integrating PsySH settings for this directly with drush settings yet, Hopefully sometime
soon I will open a pull request to integrate shell setting with drush better. So for now, I use the config file functionality
that comes bundled with PsySH to add a config file. See [here][psysh-config] for more details.

<!--more-->

All we need to do is create a file at `~/.config/psysh/config.php`. As you can see from the extension,
this is just a PHP file, so we can add whatever logic in here we need.

I crudely detect if this is being run through drush by checking for the `drush_drupal_major_version` function. If it
exists (and therefore drush has been bootstrapped) I add my drush specific file based on drupal version.

Here is my `config.php` file:

{% highlight php %}
<?php

$config = [];

// If the shell has been booted by drush check for a version file.
if (function_exists('drush_drupal_major_version')) {
  $drupal_version = drush_drupal_major_version();
  $include_path = __DIR__ . '/includes';
  $drupal_version_include = "$include_path/drupal-$drupal_version.php";

  if ($drupal_version && file_exists($drupal_version_include)) {
    $config['defaultIncludes'][] = $drupal_version_include;
  }
}

return $config;
{% endhighlight %}

From the sample config code above, you can see that I assume my drupal specific files will be in the `~/.config/psysh/includes/` directory.

So there should be a structure like...

{% highlight sh %}
$ tree ~/.config/psysh/
├── config.php
└── includes
    ├── drupal-7.php
    └── drupal-8.php
{% endhighlight %}

And finally, here is a breakdown of things I add to my Drupal 8 (`drupal-8.php`) file, as an example:

{% highlight php %}
<?php

// Add some frequently used services as variables to the shell session. These
// will get autocompleted too.
$em = \Drupal::entityManager();
$config_factory = \Drupal::configFactory();
$http_client = \Drupal::httpClient();
$module_handler = \Drupal::moduleHandler();
$token = \Drupal::token();
$csrf = \Drupal::csrfToken();
$url_generator = \Drupal::urlGenerator();
$link_generator = \Drupal::linkGenerator();
$module_handler = \Drupal::moduleHandler();

if ($module_handler->moduleExists('serialization')) {
  $serializer = \Drupal::service('serializer');
}

// Create an alias with no namespace for each entity type.
// E.g. 'Drupal\node\Entity\Node' gets an alias of 'Node', so 'User::load(123)'.
// This saves having to type full namespaces for entities or use the correct
// namespace each time.
foreach ($em->getDefinitions() as $entity_type) {
  $class = $entity_type->getClass();
  class_alias($class, (new \ReflectionClass($class))->getShortName());
}

// Handy lookup for services. This just prints out an array of all service ID's.
$services = \Drupal::getContainer()->getServiceIds();
{% endhighlight %}

Hopefully this helps give you some ideas about helpful things to help make using
the drush shell easier.

[psysh]: http://psysh.org/
[psysh-config]: http://psysh.org/#configure