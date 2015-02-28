---
layout: post
title: "Real time comments stream using Rails 4, SSE and Redis"
modified:
categories: ['ruby']
tags: []
image:
  feature:
date: 2013-06-28T19:21:20-08:00
comments: true
---

I was eager to test out the new features of rails 4 so I decided to use the rails 4 streaming feature to create a live comment stream a la Facebook.

I know that the most common choice for this would be a node based stack but I feel like having a solution to accomodate a more general stack is great.

### TL;DR;

You can find the source for this project on my [Github repo](https://github.com/djpate/LiveCommentRails4).

### DEMO

Heroku doesnt play nice with SSE so setting up a live demo is painfull so if you really need to see it to believe it here is a video of the thing in action. I Know it sucks.

<iframe width="560" height="315" src="https://www.youtube.com/embed/1JJpAEClUQc" frameborder="0" allowfullscreen></iframe>

### REQUIREMENTS

* Rails 4.0+
* Redis
* A server that accept concurent connections (here we’ll use Puma)


### LET’S START WITH A BASIC COMMENT SYSTEM

*app/controllers/comments_controller.rb*
{% highlight ruby %}
class CommentsController < ApplicationController

  def index
    @comments = Comment.order('id desc').limit(5).all.reverse
  end

  def create
    comment = Comment.new
    comment.title = params[:title]
    comment.content = params[:content]
    comment.save!
    redirect_to comments_path
  end

end
{% endhighlight %}


*app/models/comment.rb*
{% highlight ruby %}
class Comment < ActiveRecord::Base
end
db/migrate/xxxxx_create_comments.rb
class CreateComments < ActiveRecord::Migration
  def change
    create_table :comments do |t|
      t.string :title
      t.string :content
      t.timestamps
    end
  end
end
{% endhighlight %}

*config/routes.rb*
{% highlight ruby %}
LiveCommentRails4::Application.routes.draw do
  resources :comments, :only => [:index, :create]
end
app/views/comments/index.html.erb
<% @comments.each do |comment| %>
  <h1><%=comment.title%></h1>
  <p><%=comment.content%></p>
<% end %>

<h3>Add comment</h3>

<\%= form_tag(comments_path, method: "post") do %>
  <input type="text" name="title" placeholder="Title" />
  <textarea name="content" placeholder="Comment"></textarea>
  <input type="submit" value="Save" />
<% end %>
{% endhighlight %}

See, I wasn’t lying when I said basic!

### NOW LET’S DISCUSS WHAT WE WANT TO ACHEIVE AND HOW

So at this point the only way to see new comments would be to refresh the page, the idea here is that new comments get added realtime to the page without any page reload or xhr polling.

In order to do that we are going to use Server-Sent Events a cool feature of HTML5.

Server-Sent Events are real-time events emitted by the server and received by the browser. They’re similar to WebSockets in that they happen in real time, but they’re very much a one-way communication method from the server.

On top of that we are going to take advantage of redis pub/sub feature so publish events to our realtime controller.

Ok so let’s add some stuff to our project.

*Gemfile*
{% highlight ruby %}
#add those gems
gem 'redis'
gem 'puma'
{% endhighlight %}

*app/models/comment.rb*
{% highlight ruby %}
class Comment < ActiveRecord::Base

  after_save :notify_new_comment

  def notify_new_comment
    redis = Redis.new
    event_hash = {:title => self.title, :content => self.content}
    redis.publish("new_comment", event_hash)
  end

end
{% endhighlight %}

*lib/sse/writer.rb*

**this is a helper I found on another blog about SSE and basicly takes care of setting up the correct format for SSE data.**
{% highlight ruby %}
module Sse
  class Writer
    #http://tenderlovemaking.com/2012/07/30/is-it-live.html
    def initialize io
      @io = io
    end

    def write object, options = {}

      options.each do |k,v|
        @io.write "#{k}: #{v}\n"
      end

      @io.write "data: #{object}\n\n"
    end

    def close
      @io.close
    end
  end
end
{% endhighlight %}

*app/controllers/realtime.rb*

**the important stuff here is the include ActionController::Live. This a rails 4 helper that enables the stream mechanism.
Other than that it’s pretty straight forward. We listen for a new_comment event from redis and when it happens we write to the stream.**

{% highlight ruby %}
require 'sse/writer'
class RealtimeController < ApplicationController

  include ActionController::Live

  before_action :setup_stream

  def comments
    begin
      @redis.subscribe('new_comment') do |on|
        on.message do |event, data|
          @stream.write(JSON.parse(data).to_json, :event => :new)
        end
      end
    ensure
      @stream.close
    end
  end

  private

  def setup_stream
    response.headers['Content-Type'] = 'text/event-stream'
    @redis  = Redis.new
    @stream = Sse::Writer.new(response.stream)
  end

end
{% endhighlight %}

*config/routes.rb*
{% highlight ruby %}
LiveCommentRails4::Application.routes.draw do
  resources :comments, :only => [:index, :create]
  get '/realtime/comments' => 'realtime#comments', :as => :realtime_comments
end
{% endhighlight %}

*app/assets/javascripts/comments.js*

**Here we are using the basic implementation of EventSource to log everytime we have an event from the server.**
{% highlight js %}
jQuery(document).ready(function() {
  var source = new EventSource('/realtime/comments');
  source.addEventListener('new', function(e) {
   console.log(e);
  });
});
{% endhighlight %}

Try it now. Open 2 browsers and post a comment in one and look at the console on the other one. you should see a console.log with some data. (don’t forget to migrate…)

From there I’m sure you get it and you can play with it the way you want however if you want to stick around you can see the full js implementation below.

### LET’S MAKE IT SEXIER

*app/assets/stylesheets/comments.css.scss*
{% highlight scss %}
#comment_app{
  padding: 5px;
  #comments_container{
    .comment{
      background-color: #EFEFEF;
      border:1px solid black;
      padding: 5px;
      margin-bottom: 5px;
      h1{
        padding: 0;
        margin: 0;
        font-size: 15px;
      }
      p{
        font-size: 12px;
      }
    }
  }
  #see_newer{
    display: none;
    border:1px solid black;
    padding: 5px;
    background-color: #E9e9e9;
  }
}

#comment_form{
  input,textarea{
    padding: 5px;
    border:1px solid #EFEFEF;
    margin-bottom: 10px;
    width: 500px;
  }

  textarea{
    height: 200px;
  }

  input[type=submit]{
    display: block;
    width: 100px;
  }
}
{% endhighlight %}

*app/assets/javascripts/comments.js*
**I use backbone here so don’t forget to add it to your application.js**
{% highlight js %}
jQuery(document).ready(function() {

  //definitions

  var source = new EventSource('/realtime/comments');

  var template = _.template($("#comment_template").html());

  var App = Backbone.View.extend({

    events: {
      'click #see_newer': 'seeNewComments'
    },

    new_comment: function(){
      this.$el.find('#see_newer').show().html('See '+this.collection.length+' new comment(s)')
    },

    seeNewComments: function(){
      this.collection.each(_.bind(function(model){
        var html = template(model.toJSON());
        this.$el.find('#comments_container').append(html);
      },this));
      this.clear();
    },

    clear: function(){
      this.collection.reset();
      this.$el.find('#see_newer').hide();
    }

  });

  //instances

  var newCommentCollection = new Backbone.Collection([]);
  var appInstance = new App({el: $('#comment_app'), collection: newCommentCollection})

  //listeners

  newCommentCollection.on('add', function(model){
    appInstance.new_comment();
  });

  source.addEventListener('new', function(e) {
    data = $.parseJSON(e.data);
    var comment = new Backbone.Model({title: data.title, content: data.content});
    newCommentCollection.add(comment);
  });

});
{% endhighlight %}

and finally the view

*app/views/comments/index.html.erb*

{% highlight ruby %}
<div id='comment_app'>

  <div id="comments_container">
    <% @comments.each do |comment| %>
      <div class='comment'>
        <h1><%=comment.title%></h1>
        <p><%=comment.content%></p>
      </div>
    <% end %>
  </div>
  <div id='see_newer'>
  </div>
</div>

  <script type="text/template" id="comment_template">
    <div class="comment">
      <h1><%%- title %></h1>
      <p><%%- content %></p>
    </div>
  </script>

<div id='comment_form'>
  <h3>Add comment</h3>
  <%= form_tag(comments_path, method: "post") do %>
    <input name='title' placeholder='Title'><br />
    <textarea name='content' placeholder='Comment'></textarea>
    <input type='submit' value='Save' />
  <% end %>
</div>
{% endhighlight %}
### ADDITIONAL INFO

This is not a production ready implementation. you need to deal with reconnects and so on but this should give you a good starting point.

Happy Coding!