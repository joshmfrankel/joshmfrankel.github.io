---
layout: post
title: Using attr_accesor for Class level methods
category:
- tutorials
---

The beginning steps to writing a new class usually involve writing a quick
<code>def initialize</code> method. The purpose of these type of methods is to
setup any dependencies or configurations necessary for the object to fulfill its
responsibility. With instance variables, a common paradigm is to use an attr_reader
or attr_writer to read/write values. However, these methods specifically work
off of instance variables.
<!--excerpt-->

attr_reader :element

## The difference between instance variables and class variables

An <code>attr_</code> is basically a shortcut for getter and setter methods. For
example, if you have the following initializer method, and wanted to access the 
instance variable <code>@element</code> in a private method, you might do something
like this:

{% highlight ruby %}
class SiteBuilder
  attr_accessor :element

  def initialize(element)
    @element = element
    add_timestamp_to_element
    @element
  end

  private

  # Notice we no longer need to reference the instance variable but can instead
  # call element which is now part of the attr_accessor reader portion
  def add_timestamp_to_element
    element.update(timestamp: DateTime.now)
  end
end
{% endhighlight %}

This approach works because you reference the set instance variable @element. However,
for class methods you don't have access to instance level variables so something
like the following won't work:

{% highlight ruby %}
class SiteBuilder
  attr_accessor :element

  def self.format(element)
    @element = element
    append_timestamp
  end

  def self.add_timestamp_to_element
    element.update(timestamp: DateTime.now)
  end
end
{% endhighlight %}

The reason the instance variable isn't available is that the context that the
above is ran in is the class context. In other words it runs directly off the class
like a method while instances of that class represent an object.

{% highlight ruby %}
SiteBuilder.new(element) #=> Returns an element with the updated timestamp
SiteBuilder.format(element) #=> Explodes horribly in a deadly fire tornado
{% endhighlight %}

For SiteBuilder.new(element), we are creating a new object instance of the SiteBuilder
class. The element becomes part of its properties or what describes its data and features.

For SiteBuilder.format(element), we are calling a method that works off of whatever
the input is. It isn't a new object but rather a way to perform an operation on
something that doesn't need an object.

## Using an attr_accessor method with class level methods

Still want to have the benefits of attr_reader, attr_writer, and attr_accessor? We
can implement another approach that allows the usage of these instance variables in
a class level context. Basically what it does is create an instance variable in
the class context to be used elsewhere.

{% highlight ruby %}
class SiteBuilder

  # Creates an instance variable in the class context
  class << self
    attr_accessor :element
  end

  def self.format(element)
    self.element = element # We need to set the variable directly on the class
    append_timestamp
  end

  # We can now use the following shorthand without the @ symbol
  def self.add_timestamp_to_element
    element.update(timestamp: DateTime.now)
  end
end
{% endhighlight %}
