---
layout: post
title: Prevent CSS transitions on page load with ES6
category: articles
tags:
- JavaScript
- CSS
---

I recently noticed on my blog that sometimes the CSS hover transitions for `font-size`, `border-color`, and
`color` animate on page load. While this doesn't impact the final design, it does make the
initial page load a bit of a stylistic mess. 

<p itemprop="description">I stumbled upon a <a href="https://css-tricks.com/transitions-only-after-page-load/">great post on css-tricks</a> that outlined a fix for this issue using <strong>jQuery</strong>. However, my goal for this site has always been keeping it as lean as possible. That's why I use Jekyll for static pages and minimal scripting (outside Disqus and Google Analytics). So of course I dug into how we can skip <strong>jQuery</strong> and just use <strong>ES6</strong>.</p>
<!--excerpt-->

Here's what the issue looks like:

{% include rich-snippets/article-image src="2019/page-load-transition.gif" caption="Page loading with CSS transition" width="1005" height="410" %}

The fix discussed in the above article goes like this. First we add a new class to the body (or could be html) element. This can be whatever you want. I called mine `.preload-transitions` for extra readability.

{% highlight html %}
<body class="preload-transitions">
  <!-- more elements -->
</body>

<!-- Alternative for html -->
<html class="preload-transitions">
 <!-- more elements -->
</html>
{% endhighlight %}

Next we write a **SCSS** rule that ensures that all elements beneath the class above have their transitions disabled. And for extra assurance drop an `!important` on them. `!important` just means that we really, really want this value to be used over other inherited rules.

{% highlight scss %}
// Note the * after the class name. This just selects everything after the initial
// class definition
.preload-transitions * {
  // Dry this up a bit with SCSS variable
  $null-transition: none !important;

  -webkit-transition: $null-transition;
  -moz-transition: $null-transition;
  -ms-transition: $null-transition;
  -o-transition: $null-transition;
  transition: $null-transition;
}
{% endhighlight %}

Lastly, we diverge from using **jQuery** and instead lean on **ES6**. First we'll add an
event listener to the `DOMContentLoaded` event (also known as page load). Next we
find the CSS class that disables the transitions `.preload-transitions`. Lastly, once
we know the DOM element, we remove the class from it. 

{% highlight html %}
<script type="text/javascript">
  document.addEventListener("DOMContentLoaded",function(){
    let node = document.querySelector('.preload-transitions');
    node.classList.remove('preload-transitions');
  });
</script>
{% endhighlight %}

That's it we're done! 

As far as browser support goes here's some general notes with links to 
caniuse.com entries:

* `addEventListener` works on all browsers except IE < 9 ([link](https://caniuse.com/#search=addEventListener))
* `querySelector` has similar support to `addEventListener` with FireFox 2-3 also not supported ([link](https://caniuse.com/#search=querySelector))
* `classList` has partial support in IE and Opera ([link](https://caniuse.com/#search=classList))

Have another CSS transition trick or hack? Did this solution work for you? Tell me about it below.
