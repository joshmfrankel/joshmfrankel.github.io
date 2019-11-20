---
layout: post
title: Recursively validate application requests with Rack
categories:
- articles
series:
- prevent invalid request params in Rack
tags:
- Rack
- Ruby
- RSpec
---

Previously, I described a process of using Rack to validate request objects
for malicious formats (specifically null bytes). However, request params come
in many different formats from arrays to hashes to arrays containing hashes, there
are a lot of different cases to cover. Fortunately using recursion allows for
an elegant solution that covers each scenario.

<!--excerpt-->

If you haven't read the first post in this series it lays a lot of the groundwork
we're going to go over in this post. If you're interested you can find it here: ["Don't let the null bytes bite"](http://joshfrankel.me/blog/don-t-let-the-null-bytes-bite/).

Onward!

### Adding test support for request param formats

First let's list out all the possible permutations that a request object value can be formatted in:

1. Just a plain String value (We've already supported this)
2. An array of strings values
3. A JSON key value pair where the value is a string
4. A nested variation of 2 and 3 above (example: An array with JSON key value pairs that contain string values)

It's time again to write some more tests:

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

    # Newly added
    it "responds with 400 BadRequest for hashes with strings" do
      post "/users", name: { inner_key: "I am #{null_byte} bad" }

      expect(last_response.bad_request?).to eq true
    end

    # Newly added
    it "responds with 400 BadRequest for arrays with strings" do
      post "/users", name: ["I am #{null_byte} bad"]

      expect(last_response.bad_request?).to eq true
    end

    # Newly added
    it "responds with 400 BadRequest for arrays containing hashes with string values" do
      post "/users", name: [
        {
          inner_key: "I am #{null_byte} bad"
        }
      ]

      expect(last_response.bad_request?).to eq true
    end
  end

  context "WITHOUT invalid characters" do
    it "responds with a 200 ok" do
      post "/users"

      expect(last_response.ok?).to eq true
    end

    it "responds with a 200 ok for strings" do
      post "/users", name: "safe name"

      expect(last_response.ok?).to eq true
    end

    # Newly added
    it "responds with a 200 ok for hashes with strings" do
      post "/users", name: { inner_key: "safe name" }

      expect(last_response.ok?).to eq true
    end

    # Newly added
    it "responds with a 200 ok for arrays with strings" do
      post "/users", name: ["safe name"]

      expect(last_response.ok?).to eq true
    end

    # Newly added
    it "responds with a 200 ok for arrays containing hashes with string values" do
      post "/users", name: [{ inner_key: "safe name" }]

      expect(last_response.ok?).to eq true
    end
  end
end
```

The above just continued the previous format we established for testing request values that are strings. Now it just supports a couple more valid formats. Onto making all this pass.

### Adding logic to validate multiple request param formats

Once I have all my test cases written out the next step I like to take is writing the simplest implementation to get all of them passing. For this, the plan is to check the type of the request param and navigate to
any contained string values.

Building on our existing implementation above we can adjust the **#call** method to call out to a private method which allows us to better organize the upcoming logic. We'll start by abstracting the current String logic.

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
      if value.respond_to?(:match)
        string_contains_invalid_character?(value)
      end
    end
  end

  def string_contains_invalid_character?(string)
    invalid_characters_regex = Regexp.union(INVALID_CHARACTERS)

    string.match?(invalid_characters_regex)
  end
end
```

Now checking for invalid characters in a String is contained within its
own method. Additionally, the boolean check within **#call** now has
its own method dedicated to determining this value.

Moving onto adding support for arrays we can add the following code to the **#has_invalid_character?** private method:

``` ruby
def has_invalid_character?(param_values)
  param_values.any? do |value|
    if value.respond_to?(:match)
      string_contains_invalid_character?(value)
    elsif value.is_a?(Array)
      value.any? do |array_value|
        string_contains_invalid_character?(value) if array_value.respond_to?(:match)
      end
    end
  end
end
```

Started to get a bit verbose above but stick with me. I've got a plan to clean it up. Adding support for JSON params follows a similar format as arrays:

``` ruby
def has_invalid_character?(param_values)
  param_values.any? do |value|
    if value.respond_to?(:match)
      string_contains_invalid_character?(value)
    elsif value.is_a?(Array)
      value.any? do |array_value|
        string_contains_invalid_character?(array_value) if array_value.respond_to?(:match)
      end
    elsif value.respond_to?(:values)
      value.values.any? do |hash_value|
        string_contains_invalid_character?(hash_value) if hash_value.respond_to?(:match)
      end
    end
  end
end
```

So that's a rats nest of code now isn't it? There are a couple problems with it aside from the fact it's hard to read.

1. It has nested conditional logic
2. There are repeated conditionals. The code isn't confident into what value it is receiving
3. Arrays and JSON logic only supports 1 level of nesting. What if you had something like `key: [{ another_key: { super_inner_key: ["bad \u0000"] }}]`
4. Our test that checks, "responds with 400 BadRequest for arrays containing hashes with string values" is failing because the request is coming back successful

Given all of that we can determine a solution by looking for commonalities between the various branching conditionals. So what can we identify about them?

1. All formats eventually nest down into a string value
2. Formats can be multi-level nested

That's it! Each needs to determine its lowest level string value to compare to our created regular expression above. Here's where recursion is our friend.

## What is recursion?

Recursion is when you define a method that calls itself until a stop condition is
reached. Essentially, its an infinite loop with a conditional return at some point.

<blockquote class="Info Info-right">
"The Fibonacci sequence [consists of a sequence where] each number is the sum of the two preceding ones, starting from 0 and 1"
<cite><a href="https://en.wikipedia.org/wiki/Fibonacci_number">- Wikipedia</a></cite>
</blockquote>

A classic example of a problem easily solved by recursion is determine the nth number
in the Fibonacci sequence. The sequence goes like: 0 + 1 = 1 + 1 = 2 + 1 = 3 + 2 = 5. So if we wanted to know the 10th number in the sequence we can write a recursive method that stops after 10 iterations.

``` ruby
def fibonacci(ending_index, current_number = 1, previous_number = 0, current_index = 0)
  if current_index == ending_index
    return previous_number
  else
    current_index += 1
    next_number = current_number + previous_number
    fibonacci(ending_index, next_number, current_number, current_index)
  end
end

fibonacci(10) #=> 55
```

So starting from 1 the tenth number in the sequence is 55. How this works is that
fibonacci calls itself within the else condition above while passing along previous
values and indexes. When the current index is equal to the ending index (or stop
condition) we return the previous number which happens to be the nth number in the 
sequence. So writing the above out in full:

``` ruby
1 + 0 - First number is 1
1 + 1 - Second number is still 1
2 + 1 - Third is 2
3 + 2 - Fourth is 3
5 + 3 - Fifth is 5
8 + 5 - Sixth is 8
13 + 8 - Seventh is 13
21 + 13 - Eighth is 21
34 + 21 - Ninth is 34
55 + 34 - And tenth is 55
=> 55
```

That's pretty much it to recursion. A method that calls self with a stop condition. One thing to note is that since the above could easily become an infinite loop defining a stop condition early prevents stack level too deep errors.

This is a pretty simple example of it so let's go back to a more practical application of it.

## Recursively validate request params

Looking back at our requirements above:

1. All formats eventually nest down into a string value
2. Formats can be multi-level nested

We can use recursion to shape the method signature. Additionally, since we built the supporting unit tests changing our logic becomes easy to know if we've got it right or not.

Converting our current implementation to recursion looks like the following:

``` ruby
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

    string.match?(invalid_characters_regex)
  end
```

Breaking this down step by step, we've converted the previous **#has_invalid_character?** method to instead call out to a new recursive method called **#check_for_invalid_characters_recursively**. The allows the new method to focus on recursion while the **#has_invalid_character?** can just loop through values returning early as soon as one is true.

``` ruby
def has_invalid_character?(param_values)
  param_values.any? do |value|
    check_for_invalid_characters_recursively(value)
  end
end
```

The recursive method shouldn't look much different than the previous implementation of **#has_invalid_character?** (since we moved the logic out of it). We still have the `if...elsif...else` conditional based on type. The notable change here occurs for Hashes and Array in that they now call **#check_for_invalid_characters_recursively** with the current value in the Hash or Array and the current depth of iteration (more on this in a minute).

### Breaking down the logic flow step-by-step

So if we assume that the request param looks like `params: { safe: "nothing dangerous", sketchy: { devious: "evil \u0000 string" } }` the program flows looks like this:

``` ruby
# request.params.values #=> ["nothing dangerous", {:devious=>"evil \u0000 string"}]
if has_invalid_character?(request.params.values)

#...
def has_invalid_character?(param_values)
  param_values.any? do |value|
    # First iteration:
    # value => "nothing dangerous"
    check_for_invalid_characters_recursively(value)
  end
end

def check_for_invalid_characters_recursively(value, depth = 0)
  # First iteration:
  # value => "nothing dangerous"
  # depth => 0
  if value.respond_to?(:match)
    # Returns false because there is not an invalid string present in "nothing dangerous"
    string_contains_invalid_character?(value)
  elsif # ...
  # ...
end

# Second Iteration
# {:devious=>"evil \u0000 string"}

def has_invalid_character?(param_values)
  param_values.any? do |value|
    # Second iteration:
    # value => {:devious=>"evil \u0000 string"}
    check_for_invalid_characters_recursively(value)
  end
end

def check_for_invalid_characters_recursively(value, depth = 0)
  # Second iteration:
  # value => {:devious=>"evil \u0000 string"}
  # depth => 0

  depth += 1 # depth => 1
  if value.respond_to?(:match)
    # ...
  
  # {:devious=>"evil \u0000 string"} responds to .values
  elsif value.respond_to?(:values)
    # value.values converts it back to an array
    # #=> ["evil \u0000 string"]
    value.values.any? do |hash_value|
      # hash_value => "evil \u0000 string"
      # depth => 1
      # below re-calls the current method recursively
      check_for_invalid_characters_recursively(hash_value, depth)
    end
  elsif value.is_a?(Array)
    # ...
  end
end

# Third iteration
# We've now drilled down through the hash to extract the lowest level
# value that is a string. We're ready to validate it
def check_for_invalid_characters_recursively(value, depth = 0)
  # Third iteration:
  # value => "evil \u0000 string"
  # depth => 1
  if value.respond_to?(:match)
    # Returns true because "evil \u0000 string" contains a null byte
    string_contains_invalid_character?(value)
  elsif # ...
  # ...
end

# The true return above kicks us back up into the parent iterator
def has_invalid_character?(param_values)
  param_values.any? do |value|
    # Since the below line returned true the above .any? call is also true.
    # This indicates that the request contained invalid characters and we
    # should immediately reject it with a 400 error
    check_for_invalid_characters_recursively(value)
  end
end

# Back to the #call method we returned as soon as a value contained
# something invalid. Take that malicious requests!
if has_invalid_character?(request.params.values)
  return [400, {}, ["Bad Request"]]
end
```

So a rough outline of the above could look like:

1. Given a request send through all params as an Array
2. Continue to iterate recursively until the value is a string and check it
for invalid characters.
3. As soon as one value contains something malicious stop execution as .any? becomes
true.

You might still be wondering with the depth variable is included above. The reason
for this is to avoid performance issues with malicious requests. Our above strategy
relies on recursion to detect invalid characters but if an attacker instead crafted
a request with 100 keys that each contained 100 keys where each had a value the recursion
above would be forced to traverse all the params. This could lead to memory issues and/or
timeouts. Using the depth variable acts as safeguard in that we only check request params
2 levels deep. This is reasonable to most request formats and should catch most issues.
2 is an arbitrary value as I could have also said 5 here. 

This is all accomplished with the following code:

``` ruby
def check_for_invalid_characters_recursively(value, depth = 0)
  return false if depth > 2

  depth += 1

  # ... some conditional
    check_for_invalid_characters_recursively(hash_value, depth)
```

Each recursive loop, increments a counter that prevents recursion from traversing
to far down the request object. The depth variable is always passed along anytime
an Array or Hash is encountered as those indicate that we need to traverse inside
of them. Strings on the other hand are the ending state we wish to have values
be formatted in.

## Wrap up

So with all of that we now have a way to determine malicious values within a request
object regardless of it being a String, Array, or Hash. Avoiding the sting of null bytes one request at a time.

But wait! What about if a user's session contains null bytes in it? Have no fear as the [next post in this series
covers how to handle it](http://joshfrankel.me/blog/invalidate-requests-when-a-user-session-contains-null-bytes-in-the-rack-layer/).

What's a fun way you've used recursion to solve a complex programming problem? I'd love to discuss what the outcome was below in the comments.

Thanks for reading.
