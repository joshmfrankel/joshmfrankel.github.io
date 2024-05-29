---
layout: post
title: Destructuring the World (and any Object) in Ruby
categories:
- articles
tags:
- ruby
---

I've really been enjoying Ruby's destructuring syntax for Hashes. So much so that I
dug in a little further and figured out how to destructure pretty much everything in Ruby.
<!--excerpt-->

First off, if you aren't familiar with Ruby destructure syntax here's a quick
refresher:

``` ruby
my_hash = { first: "the first", second: "not first"}
my_hash => { first: }

# First is now a variable
first #=> "the first"
```

Now let's say you have an Object that you only want to pull certain attributes
out of. Additionally, you want to set these to temporary variables. Below I've
outlined how this would typically be accomplished.

``` ruby
class MyClass
  attr_reader :name

  def initialize
    @name = "deconstruct"
  end
end

new_object = MyClass.new
name = new_object.name #=> "deconstruct"
```

If we try to use the destructuring syntax on the object, the error message
gives us a clue as to how to make our object destructurable.

``` ruby
new_object = MyClass.new
new_object => { name: }
#=> `#<MyClass:0x00007f0e8823ae68 @name="hi"> does not respond to #deconstruct_keys (NoMatchingPatternError)
```

Our class does not respond to `#deconstruct_keys`, so let's implement it.

``` ruby
  def deconstruct_keys(keys)
    keys.each_with_object({}) do |key, hash|
      hash[key] = self.public_send(key)
    end
  end
```

What the above is doing, is building up a useful Hash for the object to respond with. This relies
on the object having a least a public method for the incoming key (e.g. `attr_reader :name`).
This fulfills the contract that destructuring expects when you use the `=>` syntax.

Now the above is just the basic implementation. If you need more specialization you can add conditional
logic for example to combine a value that contains an Array with `#join`. Additionally, you can allowlist
only certain keys for destructuring.

``` ruby
ALLOWED_KEYS = [:name, :email]

def deconstruct_keys(keys)
  keys.each_with_object({}) do |key, hash|
    next unless ALLOWED_KEYS.include?(key) # Allowlist for keys

    value = self.public_send(key)
    value = value.join if value.respond_to?(:join) # For Array values
    hash[key] = value
  end
end
```

The final result could look like the following. Even better as a module mixin.

``` ruby
class MyClass
  attr_reader :name

  def initialize
    @name = "deconstruct"
  end

  def deconstruct_keys(keys)
    keys.each_with_object({}) do |key, hash|
      hash[key] = self.public_send(key)
    end
  end
end

########################
# Or as a module mixin #
########################
module Deconstructable
  extend ActiveSupport::Concern

  included do
    def deconstruct_keys(keys)
      keys.each_with_object({}) do |key, hash|
        hash[key] = self.public_send(key)
      end
    end
  end
end

class MyClass
  include Deconstructable

  attr_reader :name

  def initialize
    @name = "deconstruct"
  end
end
```

And here's the usage:

``` ruby
new_object = MyClass.new
new_object => { name: }
name #=> "deconstruct"
```

That's it! Go destructure the world!
