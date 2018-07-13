---
layout: post
title: How to call Pry from within a SimpleDelegator Decorator
category:
- fixes
tags:
- ruby
- gems
---

I'm by nature a kinesthetic learning or someone that learns by doing. One of the 
ways I learn new Ruby codebases or techniques is to dive into them and see how
they work during every step of their processing. Pry is a great tool for allowing me to
dig directly into code. While, really useful for debugging and understanding, one
thing that has bothered me about using Pry is that it doesn't work properly with classes
that inherit from SimpleDelegator. Instead it ends up in a method_missing block. Luckily, 
I recently learned a great workaround for this issue which we'll get into below.
<!--excerpt-->

## What happens when calling Pry within a Decorator?

Let's say you have the following Decorator that inherits from SimpleDelegator:

{% highlight ruby %}
class UserDecorator < SimpleDelegator

  # @return [String] formats an email address with the User's name
  def format_email
    binding.pry # I want to know what's going on here

    "{name} <{email}>" #=> returns Josh <josh@example.com>
  end
end
{% endhighlight %}

You'd typically expect to see the context be within the <code>UserDecorator</code>
above. However, what actually happens is that we end up in the <code>Delegator#method_missing</code>
magic method. It looks like the below code:

{% highlight ruby %}
   78: def method_missing(m, *args, &block)
   79:   r = true
   80:   target = self.__getobj__ {r = false}
   81: 
   82:   if r && target.respond_to?(m)
   83:     target.__send__(m, *args, &block)
   84:   elsif ::Kernel.respond_to?(m, true)
=> 85:     ::Kernel.instance_method(m).bind(self).(*args, &block)
   86:   else
   87:     super(m, *args, &block)
   88:   end
   89: end
{% endhighlight %}

## SimpleDelegator workaround

<blockquote class="Info Info-right"><strong>Pro Tips <sub>[2]</sub></strong><br>
  <ul>
      <li><strong>ls</strong> - lists all available variables</li>
      <li><strong>next</strong> - execute the next line of code</li>
      <li><strong>ls -grep [keyword]</strong> - lists all available variables matching a pattern</li>
      <li><strong>whereami</strong> - shows the current code context in reference to Pry location</li>
      <li><strong>!!!</strong> - exits out of current and future Pry debugger commands</li>
      <li><strong>help</strong> - list of all available commands within a Pry session</li>
  </ul>
</blockquote>

Now, technically at this point you can still check variables as if you were in the
<code>UserDecorator</code> above with commands like <code>ls</code> to list all the 
accessible variables, but you really can't see where you are in the code. This isn't super
helpful for times when you want to <code>next</code> your way through the code.

However, there is a trick to getting around this which is hinted at in the method_missing 
response above. Instead of using the vanilla `binding.pry` use the classifed name of `::Kernel.binding.pry`<sup>[1]</sup>. Now you'll be within your method's context and be able to see your current breakpoint position.

{% highlight ruby %}
   83: def method_to_pry_into
=> 84:   ::Kernel.binding.pry
   85:   # more logic here
   86: end
{% endhighlight %}

Nice! Know any other cool tricks you've learned while using Pry? Maybe you know
about another tool that has been useful to your learning? I'd love to hear about
it in the comments below.

Thanks for reading.

### Additional Resources
1. [Original Pry Github Issue & Fix](https://github.com/pry/pry/issues/1462)
2. [Pry Command System](https://github.com/pry/pry/wiki/Command-system)
