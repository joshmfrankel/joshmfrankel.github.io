---
layout: post
title: Expecting Perfection from ActionController::Parameters
categories:
  - today-i-learned
tags:
  - ruby on rails
---

Today, I learned that Rails 8 introduced a new **ActionController::Parameters** method called `expect`. This allows for cleaner and safer parameter permission and requiring. I must have just missed this in the documentation, but what a nice quality-of-life feature to add.

<!--excerpt-->

**Before Rails 8**

Previously, you would have to require the top level key **:user** in this case and then permit its
keys with a secondary method call.

```ruby
params.require(:user).permit(:name)
```

**After Rails 8**

With Rails 8+, you can now use a simpler syntax to achieve the same results.

> expect is the preferred way to require and permit parameters. It is safer than the previous recommendation to call permit and require in sequence, which could allow user triggered 500 errors.
> expect is more strict with types to avoid a number of potential pitfalls that may be encountered with the .require.permit pattern.
>
> - [ActionController::Parameters#expect ](https://api.rubyonrails.org/classes/ActionController/Parameters.html#method-i-expect)

The above **require** to **permit** are now contained within a single method call with well-structured
arguments.

```ruby
params.expect(user: [:name])
```
