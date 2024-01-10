---
layout: post
title: "Assignment destructuring; and my other favorite Ruby 3 features"
categories:
- articles
tags:
  - ruby
---

With the advent of Ruby 3, we've got a list of notable features. From security and performance improvements to Static Analysis, Ruby continues to mature and truly stay an exciting ecosystem to work within. Let's dig into some of my favorite Ruby 3 features.
<!--excerpt-->

## Short-hand keyword arguments

I value intention revealing argument and method names. One pattern I've often run into is having to pass the same named keyword argument as the variable I am utilizing. For example:

``` ruby
def my_method(first:)
 puts first
end

# Some time later we have a variable
first = "the first"
my_method(first: first)
```

This ends up with the redundant syntax `(first: first)`.

Instead, with short-hand keyword arguments we can remove the passed in variable if that variable has the same name as the named argument. 

``` ruby
first = "the first"
my_method(first:) # Nice and clean
```

## Hash destructuring

This is a feature I've been wanting since I've used [JavaScript's destructuring assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment). It allows you to shorthand Hash structures into easily utilized variables.

``` ruby
my_hash = { first: "the first", second: "not first"}
my_hash => { first: }

# First is now a variable
first #=> "the first"
```

This syntax uses the hash rocket or rightward assignment operator in order to destructure the Hash. You can even take this one step further with a one-liner:

``` ruby
my_hash = { first: "the first", second: "not first" } => { first: }

# First is still a variable
first #=> "the first"

# So is my_hash
my_hash #=> {:first=>"the first", :second=>"not first"}
```

## Anonymous argument passing

Anytime you utilize the splat syntax `*` for positional arguments or `**`, if you had a child method call you'd have to specify the exact arguments. This is even when you just want to ferry the anonymous arguments onto the next method. With Ruby 3.2 you can now continually pass anonymous arguments just how you'd expect:

``` ruby
def my_method(**)
  my_child_method(**)
end

def my_child_method(**)
  puts(**) # Note: The parenthesis are important for utilizing anonymous arguments as it won't work otherwise
end

my_method(first: "the first", second: "not first")
#=> {:first=>"the first", :second=>"not first"}
```

## Looking ahead

I'd love to spend some time with TypeProf and rbs files for the newly added static analysis. Additionally, Ractor and Fiber would be great concepts to learn as a futher way of expanding different approaches to coding challenges in Ruby. 

Looking forward to what the future brings for Ruby!
