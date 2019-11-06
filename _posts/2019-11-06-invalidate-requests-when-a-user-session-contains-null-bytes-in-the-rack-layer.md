---
layout: post
title: Invalidate requests when a user session contains null bytes in the Rack layer
categories:
- articles
series:
- prevent invalid request params in Rack
tags:
- Rack
- Ruby
- RSpec
---

In addition to request params being sent with malicious characters, a user's session
can also contain them. If your website relies on session data (and most do) to determine
if a user is logged in, then you may experience the pain of seeing `ArgumentError: string contains null byte` appear in your logs. Luckily, building on the previous two posts we can quickly craft a mechanism to
check a users session and invalidate the request if necessary.
<!--excerpt-->

## First a rewind

Looking over our current implementation we've handled the validation processing of `request.params` efficiently using recursion:

``` ruby
class ValidateRequestParams
  INVALID_CHARACTERS = [
    "\u0000" # null bytes
  ].freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    request = Rack::Request.new(env)

    if has_invalid_character?(request.params.values)
      return [400, {}, ["Bad Request"]]
    end

    @app.call(env)
  end

  private

  def has_invalid_character?(param_values)
    param_values.any? do |value|
      check_for_invalid_characters_recursively(value)
    end
  end

  def check_for_invalid_characters_recursively(value, depth = 0)
    return false if depth > 2

    depth += 1
    if value.respond_to?(:match)
      string_contains_invalid_character?(value)
    elsif value.respond_to?(:values)
      value.values.any? do |hash_value|
        check_for_invalid_characters_recursively(hash_value, depth)
      end
    elsif value.is_a?(Array)
      value.any? do |array_value|
        check_for_invalid_characters_recursively(array_value, depth)
      end
    end
  end

  def string_contains_invalid_character?(string)
    invalid_characters_regex = Regexp.union(INVALID_CHARACTERS)
  end
end
```

This handles cases where nested data structures (arrays and hashes) containing
invalid characters like null bytes are invalidated at the Rack layer. We also
want to do this for user sessions. Since the infrastructure above is fairly flexible, we
can easily hook into this.

First though we'll need to access the user's session. Then we'll want to check it for
invalid characters. Finally like before return a `400 BadRequest` because hackers don't need any additional information.

## Setting a user's session in Rack::Test

A user's session is somewhat of a misleading phrase. This is because the concept
of session is something your framework provides. So what is a user session?

Using Rails as an example, a user session is built from existing cookies.
The cookies are then interpretting by your framework (Rails) or stored within an environment variable then translated to a session. [Justin Weiss has a superb article on the topic
that is definitely worth a read.](https://www.justinweiss.com/articles/how-rails-sessions-work/){:target="_blank"}

For the purpose of this article, we'll say that our cookie for storing a user's session
is called `my_session`. Using this we can write some tests that set a user's cookie
contain invalid characters:

``` ruby
require "spec_helper"
require "rack/test"

describe ValidateRequestParams do
  include Rack::Test::Methods

  let(:app) { MyApplication::Application }

  context "WITH invalid characters in `my_session` cookie" do
    let(:null_byte) { "%00" }

    it "responds with 400 BadRequest" do
      set_cookie "my_session=adfec7as9413db963b5#{null_byte}"

      get "/login"

      expect(last_response.bad_request?).to eq true
    end
  end

  context "WITH valid characters in `my_session` cookie" do
    it "responds with a 200 ok" do
      set_cookie "my_session=adfec7as9413db963b5"

      get "/login"

      expect(last_response.ok?).to eq true
    end
  end

  # All of our this series previous tests
```

I've used the `set_cookie` method above to prefill the cookie's value before the
request is made. [Here's the documentation on it](https://www.rubydoc.info/github/brynary/rack-test/Rack%2FMockSession:set_cookie). Basically it merges changes into the existing cookie jar.

In the first context, I've appended a null byte onto the end of
the value to ensure the request is invalid. This simulates making a request that
contains bad data.

``` ruby
context "WITH invalid characters in `my_session` cookie" do
  let(:null_byte) { "%00" }

  it "responds with 400 BadRequest" do
    set_cookie "my_session=adfec7as9413db963b5#{null_byte}"
```

As expected it fails since we haven't written the supporting code for it. Let's do
that now.

## Accessing cookies at the Rack middleware layer

Back in our **ValidateRequestParams** middleware, we can tap into our existing `#call` method
as an entry point for checking a user's cookie. Since we're already taking the current **env** and loading up the **Rack::Request** with `request = Rack::Request.new(env)`, most of the work is already done for us. **request** contains a method called **cookies** which does exactly what we need by listing out all available cookies. From this it's as easy as using the existing `#string_contains_invalid_character?` method to check the cookie's value against our accepted values regular expression.

``` ruby
def call(env)
  request = Rack::Request.new(env)

  if has_invalid_character?(request.params.values)
    return [400, {}, ["Bad Request"]]
  end

  if request.cookies["my_session"].present? && string_contains_invalid_characters?(request.cookies["my_session"])
    return [400, {}, ["Bad Request"]]
  end

  @app.call(env)
end
```

Then end result above has us passing the cookie into our string validation check with:

``` ruby
string_contains_invalid_characters?(request.cookies["my_session"])
```

We're also checking to ensure that the session is present before passing it into the regex matching method. Other than
that this does the trick in preventing malicious cookie values from causing exceptions.

## Wrap up

And that's it! We've set a cookie value for test coverage of the failing case, accessed a cookie via **Rack::Request**, and
returned a 400 BadRequest when a null byte is in the cookie's value.

What did you like about the above approach? Dislike? I'd love to discuss in the below comments and as always
thanks for reading.
