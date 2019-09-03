---
layout: post
title: Modifying strong parameter values after a request
category: today-i-learned
tags:
- Ruby on Rails
---

Strong parameters are a great way of guarding against unexpected request params. They allow you to specify the names
of keys that are accepted from a given request. I've found working between a JavaScript
front-end and Rails back-end sometimes you need to adjust request values (especially when at an
intermediary step during a refactor). However, strong parameter values can't be modified which
makes this a bit more complex.
<!--excerpt-->

## Strong Parameters are Immutable

You may have heard the term immutable before in functional programming languages. This is essentially a fancy
way of saying that an object cannot be changed. You could also refer to this object as static. For more details, [here's a great
post on the subject](https://lispcast.com/what-is-immutability/){:target="_blank"}.

In other words the following controller code will not work for changing an
integer (department_id) to a string (department):

``` ruby
class UserController < ApplicationController
  def create
    user_params[:department] = Department.find(params[:department_id]).name
    user = User.new(user_params)

    if user.save
      # success
    else
      # failure
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :department_id)
  end
end
```

It looks like we should be setting a new key `:department` based on using the request param
`:department_id` to look up the corresponding department name. However, in practice this only
returns the original object.

``` ruby
user_params #=> <ActionController::Parameters {"name"=>"Rick Grimes", "department_id"=>"2"} permitted: true>
user_params[:department] = Department.find(params[:department_id]).name #=> "Leader"

# Nothing changes
user_params #=> <ActionController::Parameters {"name"=>"Rick Grimes", "department_id"=>"2"} permitted: true>
```

This is because [ActionController::Parmeters#permit](https://edgeapi.rubyonrails.org/classes/ActionController/Parameters.html#method-i-permit) is designed to return a new instance of the object everytime it is called. Looking at the above implementation of `#user_params` we see that everytime we call `#user_params` it
in turn calls `#permit`. This means that each time we call it we get a new object. Here's the internal implementation for `ActionController::Parameters#permit` I'm talking about.

``` ruby
# File actionpack/lib/action_controller/metal/strong_parameters.rb, line 554
def permit(*filters)
  params = self.class.new # Always returns a new instance

  # more implementation code
```

So modifying strong parameters directly will never work if there is a call to `#permit`
anywhere in the chain. Ok, so now we understand why this is happening we need to side step the issue.

<blockquote class="Info Info-right">
  <strong>Note:</strong> If <code>#require</code> is being used WITHOUT <code>#permit</code> you CAN modify
strong parameters directly.
</blockquote>

## How to modify Strong Parameters

There are a couple of different approaches to fixing this issue. I'll list them out below
for clarity.

* **IDEAL:** Refactor logic so that you don't need to change request parameters in a controller action
* Modify request params before Strong Parameters
* Use a temporary variable to save the current state of strong parameters and then update it
* Use Hash#merge while using the params
* Memoize (or cache) strong parameters return value
* Don't use strong parameters

We'll run through these below

### Refactoring logic

This one is self-explanatory. If you're needing to mutate request parameters after they
pass through strong parameters this is likely a code smell. Ideally you should fix this
and have the front-end send the proper data format that the back-end expects. If you can't
(which can happen in legacy code) then try another option.

### Modify request params before Strong Parameters

This is one I've used in the past with great success. Basically, you inject logic in between
the request and the call to Strong Parameters by manipulating the `params` object.

``` ruby
class UserController < ApplicationController
  def create
    params[:department] = Department.find(params[:department_id]).name
    user = User.new(create_params)

    if user.save
      # success
    else
      # failure
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :department_id, :department)
  end
end
```

Because `params[:department]` is on the request params and not Strong Parameters,
we can safely add the value before the `User.new(create_params)` call. One thing
to note with this approach is that we need to add the `:department` key to the
user_params permit listing in order to properly use the new value without removing
it.

### Using a temporary variable

Ole' faithful. Not elegant but certainly clear in its purpose. Using a temporary variable
allows us to save the parameters into a variable that we can then modify. In the case
below we use `create_params` and assign it to the Strong Parameters object `user_params`. This way
we aren't calling `#permit` each time we access the object.

``` ruby
class UserController < ApplicationController
  def create
    create_params = user_params # temporary variable
    create_params[:department] = Department.find(params[:department_id]).name
    user = User.new(create_params)

    if user.save
      # success
    else
      # failure
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :department_id)
  end
end
```

### Use Hash#merge while using the params

Another method is to use `Hash#merge` to add another key-value pair at the time
of parameter usage. Merge's update won't persist so you have to use it immediately
where you want the additional value.

It becomes a one-liner below but there is a bit more mental juggling involved
to understand it. Also there is the hidden gotcha of if you use user_params
after the `#merge` it **WILL NOT** contain the previously merged in values.

``` ruby
class UserController < ApplicationController
  def create
    user = User.new(
      user_params.merge(department: Department.find(params[:department_id]).name)
    )

    # user_params no longer contains the :department key-value pair

    if user.save
      # success
    else
      # failure
    end
  end

  private

  def user_params
    params.require(:user).permit(:name, :department_id)
  end
end
```

### Memoize strong parameters

Memoize is another way of saying store the result into this variable so that I can
refer back to it in the future without re-calculating it. Or just caching. Ruby achieves
this with the `||=` operator.

``` ruby
class UserController < ApplicationController
  def create
    user_params[:department] = Department.find(params[:department_id]).name
    user = User.new(user_params)

    if user.save
      # success
    else
      # failure
    end
  end

  private

  def user_params
    @_user_params ||= params.require(:user).permit(:name, :department_id)
  end
end
```

One nice thing about this approach is that the original implementation within `#create`
now works. The downside here is that Strong parameters works differently now that the
default. Though I'd say this is a small price to pay.

### Don't use strong parameters

This is always an option. If a request is complicated enough and the code is unable to
be refactored crafting a custom workflow to deal with it might be the best approach.
Use sparingly so you don't end up with a codebase that has a custom parameters implementation
on every Controller.

For requests that only require a single parameter you can safely skip Strong parameters
altogether and instead just use the specific value you expect. (i.e. `User.new(params[:name])`)

## Wrapping up

With that we've discussed several different approaches to dealing with modifying Strong Parameter
values. I was reading up a lot about this the other day and stumbled across a [StackOverflow posting](https://stackoverflow.com/questions/18369592/modify-ruby-hash-in-place-rails-strong-params){:target="_blank"}
that contained many of the methods I had tried out in the past which directly inspired this article.

Got any additional tricks with Strong parameters? I'd love to hear about them in the comments.

Thanks for reading.
