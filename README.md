# pagelet_rails

This is example project of using pagelets in Rails

[Demo](https://polar-river-18908.herokuapp.com)

# Why?

* Do you have pages which show a lot of information at once? 
* The pages where you need to get data from 5 or 10 different sources? 
* What if one of them is slow?
* Does this mean your users have to wait?

Don't make your users wait for page to load.
 
# Example
 
![](https://camo.githubusercontent.com/50f4078cc4015e3df89afc753a5ff79828ac0e8e/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f662e636c2e6c792f6974656d732f303031323133314d324b3147335831483276314f2f313433303033383036373738372e6a7067)

For example let's take facebook user home page. It has A LOT of data, but it loads very quickly. How? The answer is [perceived performance](https://en.wikipedia.org/wiki/Perceived_performance). It's not about in how many milliseconds you can serve request, but how fast it **feels** to the user. 
    
The page body is served instantly and all the data is loaded after. Even for facebook it takes multiple seconds to fully load the page. But it feels instant, that it is all about.    

# Who is doing that?

Originally I saw such solution implemented at Facebook and Linkedin. Each page consists of small blocks, where each is responsible for it's own functionality and does not depend on the page where it's included. You can read more on that below.

* [BigPipe: Pipelining web pages for high performance](https://www.facebook.com/notes/facebook-engineering/bigpipe-pipelining-web-pages-for-high-performance/389414033919/)
* [Engineering the New LinkedIn Profile](https://engineering.linkedin.com/profile/engineering-new-linkedin-profile)

# What is Pagelet?

You can break a web page into number of sections, where each one is responsible for its own functionality. Pagelet is the name for each section. It is a part of the page which has it's own route, controller and view. 

The closest alternative in ruby is [cells gem](https://github.com/apotonick/cells). After using it for long time I've faced many limitations of its approach. Cells has a custom Rails-like syntax but not quite. That is frustrating as you have to learn and remember those differences. The second issue, and the biggest, cells are internal only and not designed to be routable. This stops many great possibilities for improving perceived performance, as request has to wait for all cells to render.  
 
Pagelet_rails is built on top of Rails and uses it as much as possible. The main philosophy: **Do not reinvent the wheel, build on shoulders of giants.**
 

 
# Usage
 
```ruby
# app/pagelets/current_time/current_time_controller.rb
class CurrentTime::CurrentTimeController < ::ApplicationController
  # Extend your normal Rails controller
  include PageletRails::Concerns::Controller  
  
  # `pagelet_resource` is shortcut for inline route `resource`
  pagelet_resource only: [:show] 

  def show
    # Normal Rails action
  end

end
``` 


```erb
<!-- Please note view path -->
<!-- app/pagelets/current_time/views/show.erb -->
<div class="panel-heading">Current time</div>

<div class="panel-body">
  <p><%= Time.now %></p>
  <p>
    <%= link_to 'Refresh', pagelets_current_time_path, remote: true %>
  </p>
</div>
```

And now use it anywhere in your view

```erb
<!-- app/views/dashboard/show.erb -->
<%= pagelet :pagelets_current_time %>
```
 
# Pagelet helper

`pagelet` helper allows you to render pagelets in views. Name of pagelet is his path. 

For example pagelet with route `pagelets_current_time_path` will have `pagelets_current_time` name.

## remote

Example
```erb
<%= pagelet :pagelets_current_time, remote: true %>
```

Options for `remote`:
* `true` - always render pagelet through ajax
* `:turbolinks`  - render pagelet throught ajax, but inline if it's a turbolinks page visit
* `false` or missing - render inline

## params

Example
```erb
<%= pagelet :pagelets_current_time, params: { id: 123 } %>
```

`params` are the parameters to pass to pagelet url. Same as `pagelets_current_time_path(id: 123)`

## html

```erb
<%= pagelet :pagelets_current_time, html: { class: 'panel' } %>
```

pass html attributes to pagelet

## placeholder

```erb
<%= pagelet :pagelets_current_time, placeholder: { text: 'Loading...', height: 300 } %>
```

Configuration for placeholder before pagelet is loaded.


## other

You can pass any other data and it will be available in `pagelet_options`

```erb
<%= pagelet :pagelets_current_time, title: 'Hello' %>
```

```ruby
# ...
  def show
    @title = pagelet_options.title
  end
#...
```


# Pagelet options

`pagelet_options` is similar to `params` object, but for private data and config. Options can be global for all actions or specific actions only.

```ruby
class CurrentTime::CurrentTimeController < ::ApplicationController
  include PageletRails::Concerns::Controller
  
  # Set default option for all actions
  pagelet_options remote: true 
  
  # set option for :show and :edit actions only
  pagelet_options :show, :edit, remote: :turbolinks 
  
  def show
  end
  
  def new
  end
  
  def edit
  end
  
end
```

```erb
<%= pagelet :new_pagelets_current_time %><!-- defaults to remote: true -->
<%= pagelet :pagelets_current_time %> <!-- defaults to remote: turbolinks -->

<%= pagelet :pagelets_current_time, remote: false %> <!-- force remote: false -->
```

# Inline routes

Because pagelets are small you will have many of them. In order to keep them under control pagelet_rails provides helpers. 

You can inline routes inside you controller.

```ruby
class CurrentTime::CurrentTimeController < ::ApplicationController
  include PageletRails::Concerns::Controller
  
  pagelet_resource only: [:show]
  # same as in config/routes.rb:
  #
  # resource :current_time, only: [:show]
  #
  
  pagelet_resources
  # same as in config/routes.rb:
  #
  # resources :current_time
  #
    
  pagelet_routes do
    # this is the same context as in config/routes.rb:
    get 'show_me_time' => 'current_time/current_time#show'
  end
  
end
```

# Pagelet cache

Cache of pagelet rails is built on top of [actionpack-action_caching gem](https://github.com/rails/actionpack-action_caching). 

Simple example

```ruby
# app/pagelets/current_time/current_time_controller.rb
class CurrentTime::CurrentTimeController < ::ApplicationController
  include PageletRails::Concerns::Controller
  
  pagelet_options expires_in: 10.minutes

  def show
  end

end
```

## cache_path

Is a hash of additional parameters for cache key. 

* `Hash` - static hash
* `Proc` - dynamic params, it must return hash. Eg. `Proc.new { params.permit(:sort_by) }`
* `Lambda` - same as `Proc` but accepts `controller` as first argument 

## expires_in

Set the cache expiry. For example `expires_in: 1.hour`. 

Warning: if `expires_in` is missing, it will be cached indefinitely.

## cache

This is toggle to enable caching without specifying options. `cache_defaults` options will be used (see below).

If any of `cache_path`, `expires_in` and `cache` is present then cache will be enabled.


## cache_defaults

You can set default options for caching.

```ruby
class PageletController < ::ApplicationController
  include PageletRails::Concerns::Controller

  pagelet_options cache_defaults: {
    expires_in: 5.minutes,
    cache_path: Proc.new {
      { user_id: current_user.id }
    }
  }
end
```

In the example above cache will be scoped per `user_id` and for 5 minutes unless it is overwritten in pagelet itself.


# Advanced functionality

## Partial update

```erb
<!-- app/pagelets/current_time/views/show.erb -->
<div class="panel-heading">Current time</div>

<div class="panel-body">
  <p><%= Time.now %></p>
  <p>
    <%= link_to 'Refresh', pagelets_current_time_path, remote: true %>
  </p>
</div>
```
Please note `remote: true` option for `link_to`. 

This is default Rails functionality with small addition. If that link is inside pagelet, than controller response will be replaced in that pagelet.

```ruby
# app/pagelets/current_time/current_time_controller.rb
class CurrentTime::CurrentTimeController < ::ApplicationController
  include PageletRails::Concerns::Controller
  
  pagelet_resource only: [:show]

  def show
  end

end
``` 

This will partially update the page and replace only that pagelet.
 
# Todo

* package as gem
* batch request
  * each pagelet makes a separate http call, it's very inefficient for pages with many pagelets. Goal is to group multiple pagelets into single http request. 
* streaming of components at the end of body
  * goal is to serve the page with placeholders but hold connection and render pagelets in the same request before `</body>` tag
* ~~partial updates~~
* ~~turbolinks support~~
* delay load of not visible pagelets (aka. below the fold)
  * do not load pagelets which are not visible to the user until user scrolls down. For example like Youtube comments.
