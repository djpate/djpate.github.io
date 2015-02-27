---
layout: post
title: "Add CSRF Header to All AJAX Request With jQuery"
modified:
categories: ['javascript']
tags: []
image:
  feature:
date: 2013-05-14T22:48:14-08:00
---

If you are using rails CSRF protection feature, you might have ran into the issue that you need to add a csrf header to all your ajax requests.

Fortunately if you are using jQuery there is an easy way to do it automatically.

<!--more-->

First of all use the csrf helper to add the csrf token in the meta header of your layout.

{% highlight html %}
<head>
  <%= csrf_meta_tags %>
</head>
{% endhighlight %}

This will create those two meta information in your head.

{% highlight html %}
<meta content="authenticity_token" name="csrf-param">
<meta content="someRandomString" name="csrf-token">
{% endhighlight %}

Finally you can use the ajaxSetup method from jQuery to automatically add this header to all your requests.

{% highlight js %}
(function($) {

  $.ajaxSetup({
    headers: {
      'X-CSRF-Token' : $('meta[name="csrf-token"]').attr('content')
    }
  });

}(jQuery));
{% endhighlight %}