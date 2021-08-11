---
layout: post
title: Don't let the null bytes bite
categories:
- articles
series:
- prevent invalid request params in Rack
tags:
- Rack
- Ruby
- RSpec
---

Have you ever encountered `Argument error "string contains null byte"`? What
this means is that a null byte character `\u0000` was sent as a part of the request
body. This can indicate a malicious request from someone trying to probe your
application for vulnerabilities. So how can you avoid the stinging byte of
invalid characters? Rack to the rescue!
<!--excerpt-->

## So, what are null bytes?

The super abridged version...

Ruby is compiled to C. C uses null bytes to delimit strings. When crossing between
the two languages null byte injection vulnerabilities can be encountered. Basically
bad people trying to exploit application params. 

For a more thorough understanding here's an [excellent post on the subject](https://www.more-magic.net/posts/lessons-learned-from-nul-byte-bugs.html).
It's how I got to the simplified explanation above.

## First let's recreate the error

Usually this error occurs when a param gets passed to a ActiveRecord model
from a controller. This makes it easy to reproduce the issue. Let's assume you
have a model called <strong>User</strong> that contains a single attribute name.
Let's see what happens if we try to create a new User with a name containing the
invalid character.

``` ruby
User.create(name: "Bad Person \u0000")
# ArgumentError: string contains null byte
```

ActiveRecord is protecting us here but invalid characters
are still making their way from our controller to our model. We'd like to avoid
these characters from interacting with the model layer and more so the controller
layer. So how can we prevent characters from reaching these layers?

I've listed out several options that could work:

1. Sanitize params into empty strings using `gsub` but we'd rather have
the request fail if malicious activity is going on. 
2. We could handle them on a case-by-case basis within controller endpoints 
but we'd have to remember to do that everywhere. 
3. **ApplicationController** could allow for checking the request object for all controllers (which
is closer to what we want) but really we don't even want the request to make it this far. 
4. **Rack** provides the most ideal answer as it allows for intercepting requests before they make it into the application layer.

## How Rack works

![Rack Layer Diagram](/img/2019/rack-layer-diagram.png "Rack Layer Diagram")

Rack sits in between the Server and the application. This allows for injecting
bits of functionality in the form of Middleware. This Middleware accepts the application
and environment allowing specific logic before loading a page. These execute in
a specified order meaning that one Middleware's sideeffect could become multiplicative
if order isn't taken into account.

In others words think of Rack as a stack of exams that need to be graded. You 
start at the top grading them one-by-one until you're ready to present grades
to the class.

## Using Rack middleware to intercept requests

Since Rack sits between the controller and the server, we can use it to intercept
requests. In our case we want to pre-validate requests against invalid characters. 
Specifically, if a request body contains any invalid characters we want it to fail 
with a 400 user error. The response should contain little detail as normal
users won't encounter it (don't want to give more information than we have to).

So here's our requirements:

1. Create new middleware that intercepts client made requests
2. If the request body contains invalid characters then immediately respond with
a 400 failure

First we'll create a new file called **validate_request_params.rb** within the **app/middleware**
directory.

``` ruby
class ValidateRequestParams
  def initialize(app)
    @app = app
  end

  def call(env)
    # Load the current request based on the request environment
    request = Rack::Request.new(env)

    # Ending with this line basically says continue executing the application
    # request as normal
    @app.call(env)
  end
end
```

Middleware must define a **#call** method which accepts a parameter that represents
the request environment. It also needs to define the **#initialize** method which
accepts a parameter for the application. Usually the **#call** method ends with calling
`@app.call(env)` which continues the application execution with whatever side effects
have occurred within the middleware. I've done that above.

That's great and all but we need to add this somewhere in order for it to load. The
place to put this is within `application.rb`

``` ruby
module MyApplication
  class Application < Rails::Application
    # Require our file
    require "./app/middleware/validate_request_params.rb"

    # Insert our middleware into the stack before Rack::Head
    config.middleware.insert_before Rack::Head, ValidateRequestParams
  end
end
```

The above loads and inserts our middleware into the stack so that it loads before
**Rack::Head**. This theoretically could work in a different order but I've found
the above to suit my needs. Like I mentioned above, more complicated Rack setups
might require deeper thinking in terms of ordering.

Alright, so we've got a basic setup but nothing is happening yet. It's time to write
some tests.

## Rack-test

Rack-test provides a way to test the Rack layer of an application. It's pretty 
similar to a controller or request spec in RSpec.

{% include blockquote.html quote="Rack::Test is a small, simple testing API for Rack apps. It can be used on its own or as a reusable starting point for Web frameworks and testing libraries to build on." source_link="https://github.com/rack-test/rack-test" source_text="Rack-test" %}

Assuming we have an endpoint for creating users **post "/users"** we can use it to
send invalid characters in the request payload for testing.

``` ruby
require "spec_helper"
require "rack/test"

describe ValidateRequestParams do
  include Rack::Test::Methods

  let(:app) { MyApplication::Application }

  context "WITH invalid characters" do
    let(:null_byte) { "\u0000" }

    it "responds with 400 BadRequest for strings" do
      post "/users", name: "I am #{null_byte} bad"

      expect(last_response.bad_request?).to eq true
    end
  end

  context "WITHOUT invalid characters" do
    it "responds with a 200 ok for strings" do
      post "/users", name: "safe name"

      expect(last_response.ok?).to eq true
    end

    it "responds with a 200 ok with no params" do
      post "/users"

      expect(last_response.ok?).to eq true
    end
  end
end
```

Let's break the above down a bit more.

`require "rack/test"` use Rack-test. That's it.

`include Rack::Test::Methods` includes some useful helper methods. [These can
all be found here](https://github.com/rack-test/rack-test/blob/master/lib/rack/test/methods.rb#L58). 
We'll be using the `last_response` and `post` helper methods which return
the last mock response from rack and perform a post HTTP call respectively.

`let(:app) { MyApplication::Application }` Rack test expects app to be a defined
method within the spec. We're using RSpec's `let` syntax to define the method but
we could have just as easily have written:

``` ruby
def app
  MyApplication::Application
end
```

App lets Rack know what is handling the mock request. From this we have three 
tests:

1. Ensure a 400 Bad Request is raised when a request contains a null byte
2. Ensure requests with params still return a 200 status code
3. Ensure requests without params still return a 200 status code

The second and third specs are important to ensure we didn't break basic
application functionality. With this we can begin writing out the necessary logic 
to get this passing.

## Preventing requests with invalid characters

Jumping back to our **validate_request_params.rb** file, we'll add the logic
necessary to prevent Strings containing invalid characters from being successful
requests.

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

    invalid_characters_regex = Regexp.union(INVALID_CHARACTERS)

    has_invalid_character = request.params.values.any? do |value|
      value.match?(invalid_characters_regex) if value.respond_to?(:match)
    end

    if has_invalid_character
      # Stop execution and respond with the minimal amount of information
      return [400, {}, ["Bad Request"]]
    end

    @app.call(env)
  end
end
```

Above we're using a constant **INVALID_CHARACTERS** to contain any potential invalid
characters (for now just null bytes). 

With that we're using `Regexp.union` to convert
it to a pipe delimited regular expression to allow for matching multiple invalid
characters. For example if we had `INVALID_CHARCTERS = ["\u0000", "Clarence"].freeze` defined as our constant
the regular expression would look like: `/\x00|Clarence/`. Basically, null bytes 
or Clarence. Poor Clarence.

Next we take the **Rack::Request** object defined as `request` and convert its params (which are
just all the available query params) into an array of values from the key value pairs. In our example `request.params` would look like: 

``` ruby
{ 
  name: "I am \u0000 bad" 
}
```

Add a call to `.values` converts it to a flattened array of string values `["I am \u0000 bad"]`.

We use Enumerable `.any?` because it returns as soon as a truthy value is encountered.
This is perfect as we want the request to fail if any user submitted value contains an
invalid character. This saves processing time and keeps the logic snappy.

Lastly, within the `.any?` block we match the string against the built regular expression
of invalid characters and return true/false for `.match?`. We use `.respond_to?(:match)` to
ensure that the value returned from the params array is an object the understands the match method. 
In our case **Strings** respond to `match?`. This prevents other types from throwing errors 
when they don't understand how to match regular expressions.

With that our tests pass and we call it a day...

## What about objects other than Strings?

Alright, so strings are only ONE type of object that can be passed as a user supplied
param. Technically, arrays and JSON could also be encountered in a request object.
In the next post in the series, [I'll cover how to handle these efficiently using recursion](http://joshfrankel.me/blog/recursively-validate-application-requests-with-rack/).

How has using Rack helped solve your problem? Found a particularly useful
Rack pattern? I'd love to discuss it in the comments below.

As always, thanks again for reading. 
