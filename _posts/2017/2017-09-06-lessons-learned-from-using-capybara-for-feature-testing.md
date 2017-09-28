---
layout: post
title: 'Lessons learned from using Capybara for feature testing'
categories:
- articles
tags:
- capybara
- rspec
---

Having confidence in your application helps with the aid of great interation tests.
For Rails, that means utilizing the Capybara gem for testing features from end-to-end.

While Capybara is a great gem for accomplishing this goal, it can be at time difficult, frustrating, and
nuanced in its implementation. I've been keeping track of all the tricks I use reguarly and compiled a
list of the best ones. Hopefully, this helps give some clarity to a few best practices I use on a daily basis.
<!--excerpt-->

## Find element attribute
You can use the hash symbol syntax to return the full value of any attributes from a captured element.

{% highlight ruby %}
have_css(".PathContentOptions .NewButton[disabled]")
{% endhighlight %}

The additional syntax of <code>find(".PathContentOptions .NewButton")[:disabled]</code> is also available but
not recommended due to its inability to wait for the element to load fully with the specific
attribute.

## Find the parent of an element
Sometimes it might be nearly impossible to grab a specific container element. Whether that is due to multiple existing on the page or insufficient selector specificity, it can be difficult to grab ambigous elements.

However, if you can find an element that is a child of the element you want to write expectations against you can use xpath to find its parent.

{% highlight html %}
<div>
  <p class="target">
</div>
<div>
  <p>Excluded</p>
</div>
{% endhighlight %}
{% highlight ruby %}
node = page.find(".target")
parent = node.find(:xpath, "..")
parent["innerHTML"] #=> <p>Excluded</p>
{% endhighlight %}

## Set element value without an ID or name
Using the keyword <code>set</code> you can mimic the fill_in shortcut for elements that are difficult to find by id or name.

{% highlight ruby %}
find(".AssignModal-recurringAmount").set(2)
{% endhighlight %}

You can also select options from a drop down in a similiar fashion with the <code>select_option</code>
keyword.

{% highlight ruby %}
find(".AssignModal-recurringUnit select").first(:option, "Weeks").select_option
{% endhighlight %}

## Mimic user typing directly with send_keys
While the previous example gives you a way to directly set what value is within
an element, sometimes you want to actually mimic the user's interaction. This is 
especially important with things like <strong>autocomplete</strong> or <strong>fuzzy searches</strong>.

{% highlight ruby %}
find(".someElement").native.send_keys("User typing each character")
{% endhighlight %}

<code>Native</code> returns the element directly from the driver allowing for more control of
elements that is closer to the real implementation of the browser.

<code>Send_keys</code> mimics a user typing each character one at a time into whatever element
it is pointed at.

## Assert that the button is disabled
This allows you to match only disabled buttons on the page. Without this line Capybara 
defaults to marking elements that are disabled as not being part of the content.

{% highlight ruby %}
expect(page).to have_button("Review and Assign", disabled: true)
{% endhighlight %}

Furthermore, you can also check for visible buttons or elements with the visible attribute.

{% highlight ruby %}
expect(page).to have_button("Review and Assign", visible: true)
{% endhighlight %}

## Switch your test context to a popup
By switching to the latest browser window as the current page window you can seamlessly 
test a popup's interactions and styles.

{% highlight ruby %}
popup = page.driver.browser.window_handles.last
page.driver.browser.switch_to_window(popup)
{% endhighlight %}

## Ensure specific counts for element matches
Allows for matching and expecting a certain number of elements. Fails if there are
more or less than the specified amount.

{% highlight ruby %}
expect(page).to have_selector(".Icon", count: 4)
{% endhighlight %}

## Expect selector with text
Matching a css class or ID can sometimes not be specific enough. Especially, in
cases where a class appears more than once further matching the selector based on 
the text within it can help increase expectation specificity.
{% highlight ruby %}
expect(page).to have_selector(".LearningContent-actionButton", text: "Start Lesson")
{% endhighlight %}

Additionally, text can accept regexp syntax so you can use the <code>/i</code> case insensitive matcher

{% highlight ruby %}
expect(page).to have_selector(".LearningContent-actionButton", text: /Start Lesson/i)
{% endhighlight %}

## Waiting for ajax? Sleep no more!

Waiting for ajax can be a pain. How long should sleep be set for? A quarter-second? 1 second? Luckily, there's a way to properly wait for ajax to complete without using sleep.

{% highlight ruby %}
# Put this in your javascript helpers or other helper location you use for
# your feature specs
def wait_for_ajax
  Timeout.timeout(Capybara.default_max_wait_time) do
    loop until page.evaluate_script("jQuery.active").zero?
  end
end

# Then use in your test like
it "performs an ajax request" do
  some_ajax_event

  wait_for_ajax

  more_expectations
end
{% endhighlight %}

The original solution (which the above is based on) was proposed by Thoughtbot and can be [found here on Coderwall](https://coderwall.com/p/aklybw/wait-for-ajax-with-capybara-2-0).

## Poltergeist web driver clicking elements
<code>Error message: Capybara::Poltergeist::MouseEventFailed:</code>
The element you are trying to interact with might be overlapping another element. If you don't care about this, using node.trigger('click') can ignore
this situation.
{% highlight ruby %}
find("some element").trigger("click")
{% endhighlight %}

## Search within particular element context
You probably know that you can change the page context with the <code>within(".selector") do</code> block. But did you know you can also 
select a specific element based on where it appears?

{% highlight ruby %}
within(first(".selector", match: :first))
within(first(".selector", text: "Some text here"))
{% endhighlight %}

**Note** Make sure you use an option or element waiting approach when using first()
since first() doesn't wait until all elements load.

## Use current_scope to debug the current context
If you wanted to debug the html from within a within block you unfortunately can't do something like:

{% highlight ruby %}
within(".selector") do
  puts page["innerHTML"] # => NoMethodError
end
{% endhighlight %}

Luckily, there is a keyword that allows you to access the current scope called current_scope

{% highlight ruby %}
within(".selector") do
  puts current_scope["innerHTML"] #=> Returns the html within the within block
end
{% endhighlight %}

## Directly debugging the current page
Sometimes using <code>save_and_open_page</code> isn't enough to determine what is going
on within a test. It doesn't load assets properly or interact well with user
input.

However, there is a trick to actually debugging the state of the current page within
a test. 

{% highlight ruby %}
puts current_url
binding.pry
{% endhighlight %}

You'll end up with a printed url within your test suite which can then be copy-pasted
into your browser where you can interact with the current page. Using binding.pry
pauses execution right where you want the state of the page to be currently. I can't take credit for this trick as the [excellent post on QuickLeft](https://quickleft.com/blog/five-capybara-hacks-to-make-your-testing-experience-less-painful/) directed me to it.

## Avoiding flaky tests

Another article by the Thoughtbot team, about all the ways of avoiding race condition
like tests. I've gleaned some of my information from this article which [can be found
here](https://robots.thoughtbot.com/write-reliable-asynchronous-integration-tests-with-capybara)

## Remove css animations for less flakes and a faster test suite

By default poltergeist, and potentially other js drivers, perform css animations.
Depending upon your expectations this might slow your test down in that expectations
need to wait until an element is fully loaded. Additionally, this could impact
the consistency of your tests by introducing flaky race conditions.

Instead, if we disable animations altogether we should in theory have a 
faster test suite that is more reliable. I found the basic idea for this on 
[StackOverflow](https://stackoverflow.com/questions/19662287/turn-off-animations-in-capybara-tests).

{% highlight ruby %}
# spec_helper.rb or wherever you've configured Capybara
Capybara.register_driver :poltergeist do |app|
  opts = {
    extensions: ["#{Rails.root}/features/support/disable_animations.js"], # Disable animations for capybara
    timeout: 2.minutes,
    js_errors: true
  }

  Capybara::Poltergeist::Driver.new(app, opts)
end
{% endhighlight %}

{% highlight javascript %}
// /features/support/disable_animations.js
/*
 * Disable transition, transform, and animation css rules
 * for all feature specs. This produces faster specs that are less prone
 * to flakes due to race conditions of element loading.
 */
var disableAnimationStyles = '-webkit-transition: none !important;' +
                             '-moz-transition: none !important;' +
                             '-ms-transition: none !important;' +
                             '-o-transition: none !important;' +
                             'transition: none !important;' +
                             '-webkit-transform: none !important;' +
                             '-moz-transform: none !important;' +
                             '-ms-transform: none !important;' +
                             '-o-transform: none !important;' +
                             'transform: none !important;' +
                             '-webkit-animation: none !important;' +
                             '-moz-animation: none !important;' +
                             '-ms-animation: none !important;' +
                             '-o-animation: none !important;' +
                             'animation: none !important;'

window.onload = function() {
  var animationStyles = document.createElement('style');
  animationStyles.type = 'text/css';
  animationStyles.innerHTML = '* {' + disableAnimationStyles + '}';
  document.head.appendChild(animationStyles);
};
{% endhighlight %}

Do you have an awesome Capybara trick I missed? Let me know with a comment below.

Thanks for reading.
