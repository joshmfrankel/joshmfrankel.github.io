---
layout: post
title: Override an ActiveRecord attribute value while using the same getter method
category: today-i-learned
tags:
- ruby on rails
---

Overriding methods as a Rubyist is extremely natural and powerful. Recently, I did run into a circumstance where I had an ActiveRecord object with a DateTime that I needed in a specific TimeZone (based on user input). If you ever have tried to override the getter for an attribute with the same name and have been stumped, this article details one way of making it work.
<!--excerpt-->

Alright, first off some example data to support our solution.

{% highlight ruby %}
=> Event
 id: 1
 name: "How to be a pro Rubyist?"
 start_time: Mon, 02 Jul 2018 14:11:17 EDT -04:00
 time_zone: "UTC"
{% endhighlight %}

With the above, anytime I wanted to output the Event's <code>start_time</code> to the front-end I would need to reach
for [in_time_zone](https://apidock.com/rails/DateTime/in_time_zone).

<blockquote class="Info Info-right"><strong>in_time_zone</strong><br />
  Returns the simultaneous time in Time.zone, or the specified zone.
  <cite>- apidock.com, Ruby on Rails</cite>
</blockquote>

Here's an example of outputting the Event object's <code>start_time</code> in the proper
time zone:

{% highlight ruby %}
<% event = Event.find(1) %>
<%= event.start_time.in_time_zone(event.time_zone) %>
{% endhighlight %}

Quite, verbose no?

Well what if we just override the <code>Event#start_time</code> getter to already
perform the conversion?

{% highlight ruby %}
class Event
  def start_time
    start_time.in_time_zone(time_zone)
  end 
end
{% endhighlight %}

Nope! This will result in the dreaded <code>SystemStackError: stack level too deep</code>, which basically translated to not having a proper stop condition in a recursion loop. Or in our case overriding a method and using the same method name within it.

But all hope is not lost. <code>read_attribute</code> to the rescue!

## Read Attribute

[read_attribute](https://apidock.com/rails/ActiveRecord/AttributeMethods/read_attribute) allows you to return the value from ActiveRecord before
it makes it to the getter method. Perfect! Here's the new working solution.

{% highlight ruby %}
class Event
  ...
  def start_time
    read_attribute(:start_time).in_time_zone(time_zone)
  end 
end
{% endhighlight %}

We've now successfully overridden a named attribute using the value from the same attribute. Neat!

Got any other Ruby method tricks? Is there a better way to accomplish the above? Leave me a comment below.
