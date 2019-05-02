---
layout: post
title: Using request-based constraints to only accept JSON formats for endpoints
date: 2019-04-22 13:50 -0400
categories:
- today-i-learned
tags:
- ruby on rails
---

I prefer doing things The Rails Way<sup>tm</sup> whenever possible. Oftentimes when you are working
with web requests, your controllers expect to respond to specific content formats. Formats that are
outside of this expected format are generally handled within the controller
layer. There's nothing wrong with this approach BUT we can do better here by using 
request-based constraints in the Routing layer.
<!--excerpt-->

## A simple JSON endpoint

Here's a typical controller setup where we want to accomplish the following things:

* users#index endpoint
* return a relation of users in json format
* return a 200 success status

{% highlight ruby %}
class UsersController < ApplicationController
  def index
    users = User.all

    render json: users, status: :ok
  end
end
{% endhighlight %}

So this endpoint will only ever care about JSON responses. What happens if a client
requests HTML?

The request succeeds and returns the jsonified user relation above. What should have
been a failed request actually succeeds. Certainly not ideal but what if we use respond_to with a specific format?

## Using respond_to to enforce format

{% highlight ruby %}
class UsersController < ApplicationController
  def index
    users = User.all

    respond_to do |format|
      format.json { render json: users, status: :ok }
    end
  end
end
{% endhighlight %}

The result here is more expected. The above raises a `ActionController::UnknownFormat` when we make an HTML request to an endpoint that
expects only JSON. This makes sense because HTML isn't a handled format and therefore the controller doesn't know what to do.

We could stop here (and that would be a perfectly acceptable way of crafting the
controller action) but we've added 2 lines of code plus two blocks for what the original `render json: users` does in one. So how can we have a simple controller while still enforcing format?

## Request-based constraints for the routing layer

<blockquote class="Info Info-right"><strong>What is a Lambda?</strong><br />
A lambda is equivalent to an anonymous function in other languages. This means that you can specific logic within one without providing it a name.
</blockquote>

By specifying the valid formats directly in our routes.rb file, we ensure that at
the routing layer that using an unsupported format will respond with a failed status. This is accomplishing by using a lambda to specify valid request formats using the following syntax `lambda { |request| request.format == :json }`

So let's adjust our **routes.rb** file to add the new constraint and revert our UsersController to use the original `render json` format:

{% highlight ruby %}
# config/routes.rb
Rails.application.routes.draw do
  resources :users, only: :index, constraints: lambda { |request| request.format == :json }
end

# app/controllers/users_controller.rb
class UsersController < ApplicationController
  def index
    users = User.all

    render json: users, status: :ok
  end
end
{% endhighlight %}

Now when we make HTML requests to the endpoint the we recieve a routing error
that looks like: `ActionController::RoutingError: No route matches [GET] "/users"`. Basically this is saying that the above route doesn't even exist which is true; the HTML version of the above route isn't defined. 

In addition to using the simpler render syntax in the controller we no longer need to handle invalid request formats directly inside the controller.

Want to get even fancier? We can use the shorthand "stabby" lambda syntax to achieve the same results with:

{% highlight ruby %}
# config/routes.rb
Rails.application.routes.draw do
  resources :users, only: :index, constraints: -> request { request.format == :json }
end
{% endhighlight %}

So, now you're thinking, "This is great and all, but I have 200 other json only endpoints. Am I going to need to repeat this pattern for each of them?". Of course not! Rails doesn't leave you hanging on this.

## Advanced Constraints

For cases where we want to apply the same constraint to several routes, we can use a dedicated class. This class must respond to the <code>#matches?</code> message. With this we have ourselves a highly reusable format constraint. 

For the below example I've added a couple extra routes for demonstration purposes.

{% highlight ruby %}
# config/initializers/routes/format_constraints.rb
module Routes
  class FormatConstraints
    attr_reader :formats

    def initialize(formats)
      # This coerces formats into an array
      @formats = Array(formats)
    end

    def matches?(request)
      # This checks to see the request format matches the array
      # Useful for multi formats like Routes::FormatConstraints.new([:html, :json])
      formats.include?(request.format.symbol)
    end
  end
end

# config/routes.rb
Rails.application.routes.draw do
  resources :users, only: :index, constraints: Routes::FormatConstraints.new(:json)
  resources :posts, constraints: Routes::FormatConstraints.new(:json)
  resources :comments, constraints: Routes::FormatConstraints.new(:json)
end
{% endhighlight %}

We're still repeating ourselves a bit with the above example. Luckily, the <code>constraints</code> syntax also has a block form which makes
it even easier to group your routes by JSON only endpoints.

{% highlight ruby %}
# config/routes.rb
Rails.application.routes.draw do
  constraints Routes::FormatConstraints.new(:json) do
    resources :users, only: :index
    resources :posts
    resources :comments
  end
end
{% endhighlight %}

I've also built the above to allow for arrays of formats to match against. This is
useful for grouping routes that have multiple formats.

{% highlight ruby %}
# config/routes.rb
Rails.application.routes.draw do
  constraints Routes::FormatConstraints.new(:json) do
    resources :users, only: :index
    resources :posts
    resources :comments
  end

  # This allows requests from html, json, and csv to the 
  # tags resource below
  constraints Routes::FormatConstraints.new([:html, :json, :csv]) do
    resources :tags
  end
end
{% endhighlight %}

Now we're grouping all of our JSON routes behind a formatting constraint. Nice!

<blockquote class="Info"><strong>Acknowledgments</strong><br><br>
  <a href="https://twitter.com/stevegrossi" title="@SteveGrossi">Steve Grossi</a> for the idea of using a dedicated constraint class<br>
  <a href="https://twitter.com/unixmonkey" title="@unixmonkey">Dave Jones</a> for pointing me at the concept wrapping routes within a constraint
  block.
</blockquote>

## Wrapping Up

This is just one type of routing constraint that Rails allows for. If you'd like to
learn more about request-based constraints [here's the documentation on the subject](https://guides.rubyonrails.org/routing.html#request-based-constraints).

What did you think about this method? Does it make controller code cleaner at the price of hiding formatting logic? Tell me about it below.
