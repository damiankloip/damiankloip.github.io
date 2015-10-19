---
layout: post
title: Drupal AJAX return content
date: 2015-10-19
tags:
  - drupal
  - ajax
---

I recently wanted to populate some content on a page via AJAX in Drupal 7. It is easy enough to just
do something like this:

{% highlight js %}
$.get(path, function(data, textStatus, XHR) {
  some_callback(data);
};
{% endhighlight %}

Great. Everything is dandy, until the content you have `#attached` assets from the content you have rendered on the
server side (or anything added with `drupal_add_(js|css|library`) too). A simple AJAX request like this will give you the generated HTML
but means you cannot easily update the assets for the page and re-attach Drupal behaviors.

<!--more-->

To get around this the Drupal AJAX API can be used. However, we need to do a couple of things to be able to just get the content
from the AJAX request returned to us.

Define a simple ajax command for the `Drupal.ajax` object that just returns the htm from the response.

{% highlight js %}
Drupal.ajax.prototype.commands.returnContent = function(ajax, response, status) {
  ajax.returnContent(response.html);
};
{% endhighlight %}

Then instead of just doing a standard AJAX request, we create a Drupal.ajax instance, bind a `returnContent` function to it,
then invoke `eventResponse();` to actually make the AJAX request.

{% highlight js %}
var element_settings = {
  url : 'custom/ajax/node/1'
};

var ajaxObject = new Drupal.ajax(null, $(document), element_settings);

ajaxObject.returnContent = function(html) {
  some_callback(html);
};

ajaxObject.eventResponse(this, {});
{% endhighlight %}

Yes, this is very unintuitive and clunky but that's the AJAX API we have to work with. This will attach any new javascript assets from
the rendered content and re-attach behaviors. Nice! As the `returnContent` method is created and bound in place, the calling code you can
easily control what happens in that closure, and which variables are available in that scope etc..

That covers the jQuery side of things, but the returnContent command will not yet get invoked when the response is received from the
server. For that to happen we need to return a JSON array of ajax commands (like any regular Drupal AJAX request), including our
new `returnContent` command. E.g. Using this as a delivery callback is useful, as any render array can be used and the rest will be
taken care of.

{% highlight php %}
<?php

function custom_return_delivery_callback($page_callback_result) {
  $html = drupal_render($page_callback_result);

  $commands = [[
    'command' => 'returnContent',
    'html' => $html,
  ]];

  print ajax_render($commands);
}
{% endhighlight %}

So, an example hook_menu item would look like this, similar usage to the `ajax_deliver` delivery callback:

{% highlight php %}
<?php

/**
 * Implements hook_menu().
 */
function custom_menu() {
  $items = array();

  $items['custom/ajax/node/%node'] = [
    'title' => 'Custom content',
    'page callback' => 'node_view',
    'page arguments' => [3],
    'delivery callback' => 'custom_return_delivery_callback',
    // ...
  ];

  return $items;
}
{% endhighlight %}
