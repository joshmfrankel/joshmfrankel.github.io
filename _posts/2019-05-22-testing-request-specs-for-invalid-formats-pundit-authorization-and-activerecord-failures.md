---
layout: post
title: Testing request specs for invalid formats, Pundit authorization, and ActiveRecord failures
categories:
- today-i-learned
tags:
- ruby on rails
- RSpec
- ActiveRecord
- Pundit
---

Request formats should match their endpoints. Authorization adds a split between authorized and unauthorized requests. ActiveRecord means we're at times making an unnecessary database calls (which I like to avoid). One way to minimize such calls is to stub interactions between objects using stubbing.

For the remainder of this post, we're going to dig into how to properly test invalid request formats, authorization logic, and ActiveRecord failures. Let's get to it!

<!--excerpt-->

### Sections

If you're just looking to understand one part of this post I've listed out some
section shortcuts below:

1. [Testing request format](#when-format-is-not-json)
2. [Testing unauthorized request via Pundit](#when-the-current-user-is-unauthorized)
3. [Testing authorized request via Pundit](#for-authorized-requests)
4. [Testing ActiveRecord update failure](#when-activerecord-fails)

## First a little bit about Request Specs
With the advent of RSpec 3.5, request specs were noted as being the standard going forward. They 
allow for better testing of functionality by providing a more realistic environment 
than controller specs. Also they're apparently really fast. 

![...But why request specs?](/img/2019/why-request-specs.jpg "Zoolander meme with request specs")

I'll let the RSpec team take over here:

<blockquote class="Info">The official recommendation of the Rails team and the RSpec core team is to write request specs instead. Request specs allow you to focus on a single controller action, but unlike controller tests involve the router, the middleware stack, and both rack requests and responses. This adds realism to the test that you are writing, and helps avoid many of the issues that are common in controller specs.
<cite><a href="http://rspec.info/blog/2016/07/rspec-3-5-has-been-released/">RSpec 3.5 has been released</a></cite></blockquote>

## Our Controller and Request spec

We'll start out with a simple controller action that is used for a JSON endpoint. It has a few responsibilities that it needs to support:

* Find an object based on incoming params
* Perform authorization on the object (via Pundit)
* Attempt to update the object
* Return a JSON response

The first thing I like to do is write out a skeleton of contexts that I want to test. With that knowledge, the corresponding `/spec/requests/users_spec.rb` might look like the following:

``` ruby
# /spec/requests/users_spec.rb
require "spec_helper"

describe UsersController do
  let(:current_user) { create(:user) }
  let(:user_object_to_authorize) { create(:user) }

  before do
    # Does the necessary login logic for our application
    login_to_application(user: current_user, password: password)
  end

  describe "PATCH#update" do
    context "WHEN format is not JSON" do
      pending "raises an ActionController::RoutingError exception"
    end

    context "WHEN format is JSON" do
      context "WHEN unauthorized" do
        pending "raises a Pundit::NotAuthorizedError exception"
      end

      context "WHEN authorized" do
        context "AND the update fails" do
          pending "doesn't update the user and responds with a 422 status"
        end

        context "AND the update succeeds" do
          pending "updates the user and responds with a 200 status"
        end
      end
    end
  end
end
```

I'm using RSpec's `pending` syntax to stub 🙄... tests I'd like to write. I've found this as a great way to detail all the various logic branches. This way I can be confident when I've finished the pending tests that the underlying code is properly covered. Note, that this method only works if you know
what functionality you'd like to build next. If while developing, I come up with
additional contexts I can add them to the skeleton as pending while working on
the previously known expectations.

Normally, at this point I'd use test-driven development. For the sake of understanding, I've written out the final `UsersController#update?` action and corresponding `routes.rb` file will look like before writing tests:

``` ruby
# /app/controllers/users_controller.rb
class UsersController < ApplicationController
  def update
    @user = User.find(params[:id])
    authorize @user

    # Just a simple update to the User's name string
    if @user.update(params[:name])
      render json: { success: true }, status: :ok
    else
      render json: { success: false }, status: :unprocessable_entity
    end
  end
end

# config/routes.rb
Rails.application.routes.draw do
  resources :users, only: :update, constraints: -> request { request.format == :json }
end
```

If you're unfamiliar with the **constraints** attribute in the **routes.rb** file above, I recommend you read my previous post, ["Using request-based constraints to only accept JSON formats for endpoints"](http://joshfrankel.me/blog/using-request-based-constraints-to-only-accept-json-formats-for-endpoints/)

Now that we've got a baseline example, we'll work our way from the top-down of the request.

## Working from the the top-down

The term top-down is a bit loaded, so here's what I mean. 

Given a request, start testing at the highest level of the stack and work your way down to the lowest. What this means is that we'll start with the routing, then the authorization, and finally the **ActiveRecord** call. If you look back at the **users_spec.rb** file you'll notice that's exactly how I organized the tests. `"WHEN format is not JSON"` is a context that checks to make sure that the incoming request is using the proper format. This is higher level than the authorization check `authorize @user`. An invalid format will trigger an `ActionController::RoutingError` exception before authorization is even called, so testing it first is logical. I also find that this way of organizing tests organically builds nested contexts to place tests within.

Ok, enough talking. Moar coding! 

### When format is not JSON
First we'll add supporting coverage for non-JSON formatted requests.

``` ruby
require "spec_helper"

describe UsersController do
  let(:current_user) { create(:user) }
  let(:user_object_to_authorize) { create(:user) }

  before do
    # Does the necessary login logic for our application
    login_to_application(user: current_user, password: password)
  end

  describe "PATCH#update" do
    context "WHEN format is not JSON" do
      it "raises an ActionController::RoutingError exception" do
        expect do
          patch user_path(user_object_to_authorize), params: { format: :html }
        end.to raise_error ActionController::RoutingError
      end
    end
    ...
  end
```

I'll break this down a bit. From above we're calling the named route `user_path(user_object_to_authorize)` with the **PATCH** update action which translates to the route `patch "/users/:id"`. 

We're also passing `format: :html` to specify the request is asking for HTML. 

Finally, we wrap the entire request in an `expect do ... end` block, to capture the exception that is raised. If we were instead to write the test like this:

``` ruby
it "raises an ActionController::RoutingError exception" do
  patch user_path(user_object_to_authorize), params: { format: :html }
  expect(response).to raise_error ActionController::RoutingError
end
```

It wouldn't work properly. This is because as the `ActionController::RoutingError` is raised it would halt execution and trigger a spec failure. That's why wrapping the patch call in a block allows us to capture the exception and match it to the expectation. 

Expectation meet exception.

### When the current user is unauthorized

<blockquote class="Info Info-right">
  <strong>A little more about Pundit</strong><br>
  Essentially, it distills down business logic for questions like, "Can user X do action Y?" or "Can User X access record Z?". Here's the gem's homepage in case you haven't used it before: 
  <cite><a href="https://github.com/varvet/pundit">https://github.com/varvet/pundit</a></cite>
</blockquote>

Alright, now that we're onto our first authorization stub things start to get interesting. We want the `@user` object to fail authorization. What we don't want to do is actually create a validly authorized user for the test but rather just have authorize return false. This avoids database interaction which is 👍 for speedy specs.

With Pundit, we have a corresponding **UserPolicy** class that implements the **update?** method. All that this method does is check to see if the **current_user** can perform the **update?** action. Our request spec doesn't care about the implementation details of the `UserPolicy#update?` method, all it needs is for it to throw an exception if the **current_user** is unauthorized. 

Behind the scenes what `authorize @user` does is call `UserPolicy.new(current_user, @user).update?`. Where **current_user** is the currently logged in user and `@user` is the object we want to perform authorization for the **update?** action.

With the above we're also going to assume that the `UserPolicy#update?` method has extensive test
coverage. So why re-test it in a request spec? Stubbing the UserPolicy like we're about to do is also known as stubbing an outgoing message. Or messages sent to external objects. For more details, I've [previously written a post](http://joshfrankel.me/blog/what-it-means-to-stub-the-system-under-test/#what-to-stub) that describes what to stub and what not to as well as the difference between stubs and mocks.

Ok, so with that out of the way here's the resulting spec coverage:

``` ruby
  describe "PATCH#update" do
    context "WHEN format is not JSON" do
      # already tested
    end
    
    context "WHEN format is JSON" do
      context "WHEN unauthorized" do
        it "raises a Pundit::NotAuthorizedError exception" do
          mock_policy = instance_double(UserPolicy, update?: false)
          expect(UserPolicy).to receive(:new)
            .with(current_user, user_object_to_authorize)
            .and_return(mock_policy)

          expect do
            patch user_path(existing_user), params: { format: :json }
          end.to raise_error Pundit::NotAuthorizedError
        end
      end
    end
  end
```

There's a lot going on above let's step through it.

``` ruby
mock_policy = instance_double(UserPolicy, update?: false)
```

`instance_double` specifies that we want a partial double of the **UserPolicy**. This
object should stand in for **UserPolicy** acting just like it. Since it is imitating the object it will only have access to the same implemented methods as the **UserPolicy**. This is the major difference between **double** and **instance_double** 

The second parameter `update?: false` stubs the implemented `UserPolicy#update?` method with a return value of **false**. This will come into play in a second.

``` ruby
expect(UserPolicy).to receive(:new)
  .with(current_user, user_object_to_authorize)
  .and_return(mock_policy)
```

With this line, we're expecting the original call to `UserPolicy.new(current_user, @user).update?` to occur with the slight twist of instead returning the previously defined **mock_policy**. The above reads like: 

* When **UserPolicy** has the **new** method called 
* **With** the parameters **current_user** and **user_object_to_authorize**
* Return the **mock_policy** object

Because the **mock_policy** defined a return value of `update?: false` directly on it, the authorization check
will fail. This in turn raises a **Pundit::NotAuthorizedError** exception. Because of this there's now no need to make the authorization test setup accurate as we've just ensured that this test always fails authorization checks.

The remaining code should look familiar to the previous non-JSON format coverage.

``` ruby
expect do
  patch user_path(existing_user), params: { format: :json }
end.to raise_error Pundit::NotAuthorizedError
```

Again, we're just capturing the error during execution of the controller action. The difference is that the exception is now Pundit specific and we're passing `format: :json` which uses the expected format.

Now onto an authorized request

### For authorized requests

Now that we've detailed how to specify a failure to authorize, we can do the same thing in the opposite polarity to simulate an authorized request.

``` ruby
  describe "PATCH#update" do
    context "WHEN format is not JSON" do
      # already tested
    end
    
    context "WHEN format is JSON" do
      context "WHEN unauthorized" do
        # For reference I left this previous spec implementation
        it "raises a Pundit::NotAuthorizedError exception" do
          mock_policy = instance_double(UserPolicy, update?: false)
          expect(UserPolicy).to receive(:new)
            .with(current_user, user_object_to_authorize)
            .and_return(mock_policy)

          expect do
            patch user_path(existing_user), params: { format: :json }
          end.to raise_error Pundit::NotAuthorizedError
        end
      end

      context "WHEN authorized" do
        context "AND the update fails" do
          # We'll come back to this one next
        end

        context "AND the update succeeds" do
          it "updates the user and responds with a 200 status" do
            # The major difference here is that update?: returns true
            mock_policy = instance_double(UserPolicy, update?: true)
            expect(UserPolicy).to receive(:new)
              .with(current_user, user_object_to_authorize)
              .and_return(mock_policy)

            patch user_path(existing_user), params: { 
              format: :json,
              name: "Elliot Alderson"
            }

            # Response expectations
            expect(response).to have_http_status(:ok)
            expect(json_response).to match_json_expression(success: true)

            # Object was updated expectation
            expect(user_object_to_authorize.reload).to have_attributes(
              id: user_object_to_authorize.id,
              name: "Elliot Alderson"
            )
          end
        end
      end
    end
  end
```

The major difference in the test setup is specifying `update?: true` on the **mock_policy**. This ensures that the current request will pass the authorization check.

We'll skip down to the actual request line

``` ruby
patch user_path(existing_user), params: { 
  format: :json,
  name: "Elliot Alderson"
}
```

<blockquote class="Info Info-right"><strong>match_json_expression</strong>:<br> I'm using the <a href="https://github.com/chancancode/json_expressions">json_expressions gem</a> for the `match_json_expression` matcher. It is super handy
  when dealing with JSON payloads.
</blockquote>

Pretty straightforward here. We're just passing `name: "Elliot Alderson"` which becomes **params[:name]** in the controller to be used to
update the **User** with a new name.

Next we make some expectations to assert that the request is a **200 :ok** as well as a json payload of `success: true`

``` ruby
expect(response).to have_http_status(:ok)
expect(json_response).to match_json_expression(success: true)
```

Finally, we check to make sure that the update actually took place. 

``` ruby
expect(user_object_to_authorize.reload).to have_attributes(
  id: user_object_to_authorize.id,
  name: "Elliot Alderson"
)
```

<blockquote class="Info Info-right"><strong>Happy path?</strong><br>
  "[A] happy path is a default scenario featuring no exceptional or error conditions".<br><br>
  Oppositely, the sad path is one where execution is expected to
  encounter an error condition.
<cite><a href="https://en.wikipedia.org/wiki/Happy_path">Wikipedia</a></cite></blockquote>

One thing to note here is that because **user_object_to_authorize** is defined before the request is made we have to call `user_object_to_authorize.reload` to ensure we have the latest object from the database. We could possibly take this a step further and stub the update but I usually like to leave at least the happy path calling **ActiveRecord**. This is just a preference of mine so feel free to keep on down the stubbing path if it feels right.


### When ActiveRecord fails

Without specific model validations or other direct ways to force a failure of `@user.update`, it might be easy to just skip testing the failure case. I'd recommend against it. I've had several headaches around failures cases not having matching test coverage while refactoring existing code. This usually leads to thoughts like, "Why the hell isn't this working! It has tests... oh wait, actually it's missing coverage".

For a successful **ActiveRecord** update we went ahead and made it a real call to the database. However, since we don't have a way to make this test fail let's simulate it again with some mocking.

``` ruby
  describe "PATCH#update" do
    context "WHEN format is not JSON" do
      # already tested
    end
    
    context "WHEN format is JSON" do
      context "WHEN unauthorized" do
        # already tested
      end

      context "WHEN authorized" do
        context "AND the update fails" do
          it "doesn't update the user and responds with a 422 status" do
            mock_policy = instance_double(UserPolicy, update?: true)
            
            # Inject our user_object_to_authorize as what ActiveRecord returns
            # from the find method
            allow(User).to receive(:find).and_return(user_object_to_authorize)
            expect(UserPolicy).to receive(:new)
              .with(current_user, user_object_to_authorize)
              .and_return(mock_policy)

            # Stub the ActiveRecord update method to return false indicating a 
            # failure.
            allow(user_object_to_authorize).to receive(:update).and_return(false)

            patch user_path(existing_user), params: { 
              format: :json,
              name: "Elliot Alderson"
            }

            expect(response).to have_http_status(:unprocessable_entity)
            expect(json_response).to match_json_expression(success: false)
          end
        end

        context "AND the update succeeds" do
          # already tested
        end
      end
    end
  end
```

This was is a bit more specific than the last context; as we're injecting an
object where normally `User.find` would return one from the database. Essentially,
we're getting in front interaction between controller and model to use our
own object. This allows for the necessary stubbing of a failed **ActiveRecord** action.
A side effect of the above stubbing setup is that we never touch the database.

Some of the above is identical to the successful request.

``` ruby
mock_policy = instance_double(UserPolicy, update?: true)
...
expect(UserPolicy).to receive(:new)
  .with(current_user, user_object_to_authorize)
  .and_return(mock_policy)
```

This is just making sure that the authorization check returns true for successfully authorized.

``` ruby
# Inject our user_object_to_authorize as what ActiveRecord returns
# from the find method
allow(User).to receive(:find).and_return(user_object_to_authorize)
...

# Stub the ActiveRecord update method to return false indicating a 
# failure.
allow(user_object_to_authorize).to receive(:update).and_return(false)
```

Above, we inject our `user_object_to_authorize` test object as what **ActiveRecord** returns from the call to **find**. Later on, we stub that same user object to
respond to the update message with the value of false. By doing this we simulate
what would happen in the event that `@user.update(params[:name])` would fail.

One thing you might notice is the order of stubbing above. I find that stubbing is most understandable when you perform it in order of code execution. This make looking back at what an old test was accomplishing intuitive. Quick refresher on our controller action shows the following order:

1. Load a User object from a request param `@user = User.find(params[:id])`
2. Authorize the loaded user object `authorize @user`
3. Update the authorized user object `@user.update(params[:name])`

That's why the order of stubbing looks like:

``` ruby
# Step 1
allow(User).to receive(:find).and_return(user_object_to_authorize)

# Step 2
# We use expect here because we really want to be sure that the proper objects
# are being sent to the authorization check
expect(UserPolicy).to receive(:new)
  .with(current_user, user_object_to_authorize)
  .and_return(mock_policy)

# Step 3
allow(user_object_to_authorize).to receive(:update).and_return(false)
```

Another difference above is the usage of **allow** and **expect**. Using **expect**, will fail the test if the object does not receive the method and parameters specified. Using **allow**, simply stubs the method but will continue to work even if that method is never called. I'm using allow above for the ActiveRecord calls as I don't care as much about what they are performing outside of ensuring they return a specific value. For the **UserPolicy** I want the extra confidence that not only is it calling the **new** method but also it is passing the proper parameters of **.with(current_user, user_object_to_authorize)**. This is mostly a preference of mine as you could use allow / expect here interchangeably.

Now, by stubbing both the authorization as well as the return value from ActiveRecord, we now can test what happens when the update fails. Notably that it returns a
status code of **422 (unprocessable_entity)** and the **JSON** payload contains `success: false`.

``` ruby
expect(response).to have_http_status(:unprocessable_entity)
expect(json_response).to match_json_expression(success: false)
```

## Conclusion

We've written coverage for an invalid request format, stubbed Pundit authorization for a user object, and simulated an ActiveRecord failure. With this, we have a well tested endpoint that doesn't test more than it has to. We can not be confident that future changes don't impact this endpoint along with keeping the spec efficient.

Here's the spec in its full form:

``` ruby
require "spec_helper"

describe UsersController do
  let(:current_user) { create(:user) }
  let(:user_object_to_authorize) { create(:user) }

  before do
    login_to_application(user: current_user, password: password)
  end

  describe "PATCH#update" do
    context "WHEN format is not JSON" do
      it "raises an ActionController::RoutingError exception" do
        expect do
          patch user_path(user_object_to_authorize), params: { format: :html }
        end.to raise_error ActionController::RoutingError
      end
    end

    context "WHEN format is JSON" do
      context "WHEN unauthorized" do
        it "raises a Pundit::NotAuthorizedError exception" do
          mock_policy = instance_double(UserPolicy, update?: false)
          expect(UserPolicy).to receive(:new)
            .with(current_user, user_object_to_authorize)
            .and_return(mock_policy)

          expect do
            patch user_path(existing_user), params: { format: :json }
          end.to raise_error Pundit::NotAuthorizedError
        end
      end

      context "WHEN authorized" do
        context "AND the update fails" do
          it "doesn't update the user and responds with a 422 status" do
            mock_policy = instance_double(UserPolicy, update?: true)
            
            # Inject our user_object_to_authorize as what ActiveRecord returns
            # from the find method
            allow(User).to receive(:find).and_return(user_object_to_authorize)
            expect(UserPolicy).to receive(:new)
              .with(current_user, user_object_to_authorize)
              .and_return(mock_policy)

            # Stub the ActiveRecord update method to return false indicating a 
            # failure.
            allow(user_object_to_authorize).to receive(:update).and_return(false)

            patch user_path(existing_user), params: { 
              format: :json,
              name: "Elliot Alderson"
            }

            expect(response).to have_http_status(:unprocessable_entity)
            expect(json_response).to match_json_expression(success: false)
          end
        end

        context "AND the update succeeds" do
          it "updates the user and responds with a 200 status" do
            # The major difference here is that update?: returns true
            mock_policy = instance_double(UserPolicy, update?: true)
            expect(UserPolicy).to receive(:new)
              .with(current_user, user_object_to_authorize)
              .and_return(mock_policy)

            patch user_path(existing_user), params: { 
              format: :json,
              name: "Elliot Alderson"
            }

            # Response expectations
            expect(response).to have_http_status(:ok)
            expect(json_response).to match_json_expression(success: true)

            # Object was updated expectation
            expect(user_object_to_authorize.reload).to have_attributes(
              id: user_object_to_authorize.id,
              name: "Elliot Alderson"
            )
          end
        end
      end
    end
  end
end
```

We had to dig into the internals of Pundit a bit to properly stub UserPolicy. What did
you think of this approach? Did it couple the test stubs to closely to a third-party
implementation? Do you have an alternative you know about?

Is there a way we could refactor or consolidate the test logic above? (I can think
of two options 😉)

I would love to hear your thoughts below. As always, thanks for reading.
