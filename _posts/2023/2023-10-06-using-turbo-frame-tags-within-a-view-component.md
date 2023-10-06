---
layout: post
title: "Using Turbo Frame tags within a ViewComponent"
categories:
- articles
tags:
  - hotwire
  - ViewComponent
---

ViewComponents and Hotwire are incredible for orchestrating a beautiful, reusable View layer. While working with the two, I came across a case where I wanted a component to update via a Turbo Frame but was unable to utilize the **turbo-rails** gem's built-in `turbo_frame_tag`. Let's get into this simple fix.
<!--excerpt-->

What we need here is access to the `turbo_frame_tag` from all ViewComponent html.erb files. We can 
easily enable this by including the following module into the base ApplicationComponent. Here's
an example of a working version below:

``` ruby
class ApplicationComponent < ViewComponent::Base
  include Turbo::FramesHelper

  # ...
end
```

With the above code we can now access the frame tag within any of our templates:

``` html
<!-- my_component.html.erb -->
<%= turbo_frame_tag ...
```

I discovered this not through the documentation (as I couldn't find it) but using
Github's find symbol in project functionality. This lead me to the [definition location](https://github.com/hotwired/turbo-rails/blob/main/app/helpers/turbo/frames_helper.rb)
for `turbo_frame_tag` which gave me the proper module to include.

Good luck and start making those ViewComponents even snappier.
