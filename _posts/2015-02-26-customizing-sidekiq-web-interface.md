---
layout: post
title: "Customizing Sidekiq Web Interface"
modified:
categories: ['ruby']
tags: []
image:
  feature:
date: 2015-02-26T22:29:09-08:00
---

I've been using Sidekiq for a while now and I really love it but I've always found the Web ui to not suit my need perfectly.


For example, We use the [sidekiq-limit_fetch](https://github.com/brainopia/sidekiq-limit_fetch), which enables us to pause queues but we needed to go in the console everytime we needed to pause/unpause a queue.

Turns out adding customizing the UI to help me do that more easily was pretty simple.

First thing you want to do, is copy all the views from web/views to a folder in your app. For this example, I'll use app/views/sidekiq.

Now we just need to tell sidekiq where the views are.

{% highlight ruby %}
require 'sidekiq/web'
Sidekiq::Web.set 'views', File.join(Rails.root, 'app', 'views', 'sidekiq')
{% endhighlight %}

At this point, you can modify the views to your liking. If you need to add extra actions you can monkey path the Sidekiq::Web class. Let see what it looks like for pause/unpause.

{% highlight ruby %}
class Sidekiq::Web
  post "/queues/:name/pause" do
    Sidekiq::Queue[params[:name]].pause
    redirect_with_query("#{root_path}queues")
  end
  post "/queues/:name/unpause" do
    Sidekiq::Queue[params[:name]].unpause
    redirect_with_query("#{root_path}queues")
  end
end
{% endhighlight %}

Happy hacking!
