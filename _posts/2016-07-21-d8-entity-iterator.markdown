---
layout: post
title: Iterating Entities in Drupal 8
date: 2016-07-21
tags:
  - drupal
---

What do you do if you need to iterate over a lot of loaded entities during a single
request or background process, like a Drush command? I thought it would be as easy as
[this](https://www.drupal.org/node/2577417) ...

<!--more-->

... until I found out that the `ContentEntityStorageBase` class (which accounts for most
entities you would ever need this for) resets both the persistent cache and static cache
in it's `resetCache` method (despite the point of that method being for static cache clearing).

This leaves us in a sticky situation, as you don't want to flush the persistent cache for
all of your entities just to iterate over them.

This is a modified version of the core patch I submitted above:

{% highlight php %}
<?php

use Drupal\Core\Entity\EntityStorageInterface;

/**
 * Provides an Iterator class for dealing with large amounts of entities
 * but not loading them all into memory.
 */
class ChunkedIterator implements \IteratorAggregate, \Countable {

  /**
   * The entity storage to load entities.
   *
   * @var \Drupal\Core\Entity\EntityStorageInterface
   */
  protected $entityStorage;

  /**
   * An array of entity IDs to iterate over.
   *
   * @var array
   */
  protected $entityIds;

  /**
   * The size of each chunk of loaded entities.
   *
   * This will also be the amount of cached entities stored before clearing the
   * static cache.
   *
   * @var int
   */
  protected $chunkSize;

  /**
   * @var \Closure
   */
  protected $closure;

  /**
   * Constructs an entity iterator object.
   *
   * @param \Drupal\Core\Entity\EntityStorageInterface $entity_storage
   * @param array $ids
   * @param int $chunk_size
   */
  public function __construct(EntityStorageInterface $entity_storage, array $ids, $chunk_size = 50) {
    // Create a clone of the storage controller so the static cache of the
    // actual storage controller remains intact.
    $this->entityStorage = clone $entity_storage;
    // Make sure we don't use a keyed array.
    $this->entityIds = array_values($ids);
    $this->chunkSize = (int) $chunk_size;
  }

  /**
   * Implements \Countable::count().
   */
  public function count() {
    return count($this->entityIds);
  }

  /**
   * Implements \IteratorAggregate::GetIterator()
   */
  public function getIterator() {
    foreach (array_chunk($this->entityIds, $this->chunkSize) as $ids_chunk) {
      foreach ($this->loadEntities($ids_chunk) as $id => $entity) {
        yield $id => $entity;
      }
    }
  }

  /**
   * Loads a set of entities.
   *
   * This depends on the cacheLimit property.
   */
  protected function loadEntities(array $ids) {
    // Reset any previously loaded entities then load the current set of IDs.
    $this->resetCache();
    return $this->entityStorage->loadMultiple($ids);
  }

  /**
   * Resets the entity storage static cache.
   */
  protected function resetCache() {
    $closure = $this->getClosure();
    $closure();
  }

  /**
   * Gets the closure bound to the entity storage.
   *
   * @return \Closure
   */
  protected function getClosure() {
    if (!isset($this->closure)) {
      $this->closure = function () {
        // Reset the protected entities property, which is the static cache for
        // entity instances. Could only clear our IDs but don't think we need it.
        $this->entities = [];
      };

      $this->closure->bindTo($this->entityStorage);
    }

    return $this->closure;
  }

}
{% endhighlight %}

Did you see what is happening?

In order to just clear the static cache for entity storage, we can reset the `$entities`
property. However, this is a protected property! [Closure::bindTo()](http://php.net/manual/en/closure.bindto.php) the rescue.
Using `Closure::bindTo()` we can create a closure that resets `$this->entities` - we can then bind this to the entity
storage class itself to clear the entities property in the context of that class. Pretty cool. Somewhat a hack, but needs
must 'n' all that.