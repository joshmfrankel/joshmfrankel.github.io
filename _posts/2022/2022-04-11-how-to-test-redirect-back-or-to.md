---
layout: post
title: How to test redirect_back_or_to
categories:
- today-i-learned
tags:
  - rails
  - testing
---

Have you ever successfully used `redirect_back_or_to` in your application only to find out that testing was a challenge? Both methods rely on browser history which is generally not available within controller test and request specs. Recently I ran into this and after digging in came up with a simple solution.
<!--excerpt-->

## First what is redirect_back_or_to

`redirect_back_or_to` and its deprecated alias `redirect_back` both have the goal
of request flow control. Depending on what information is available in browser history they can either redirect to the previously visited page or to a specified fallback location. In both cases, the fallback location is required. This was the primary issue I ran into while testing. 

Let's take a look at an example. Note: We'll be looking at Minitest as a testing framework but the concepts explored here should function for RSpec as well.

## Quick example

Let's say you have the following Minitest controller test. Our desired behavior
is that when a user updates their password but does not include a number it should
redirect them and display a error message.

We're using `redirect_back_or_to` in our code with a fallback location of `root_path`

```ruby
redirect_back_or_to root_path
```

Our controller test looks like the following:
```ruby
  context "PATCH #update" do
    should "be invalid with password in wrong format" do
      sign_in @user

      patch user_passwords_path, params: { id: @user.id, user: { password: "invalid without numbers" } }

      assert_redirected_to edit_user_passwords_path
    end
  end
```

Now the above looks accurate but the problem with controller tests there is no
context as the user's last visited location. We're essentially sending a request directly to an endpoint above without first visiting the `edit_user_passwords_path` which is typically how a user would interact. The result of the above is a failing test as instead of being redirected to the edit page we end up on the `root_path`. Not ideal.

We know the issue is a lack of browser history for `redirect_back_or_to` to utilize. This is the first nod to how the method works. Also, the documentation gives a great hint as well:

{% include blockquote.html quote="Redirects the browser to the page that issued the request (the referrer) if possible, otherwise redirects to the provided default fallback location. The referrer information is pulled from the HTTP Referer (sic) header on the request. This is an optional header and its presence on the request is subject to browser security settings and user preferences. If the request is missing this header, the fallback_location will be used." source_link="https://api.rubyonrails.org/classes/ActionController/Redirecting.html#method-i-redirect_back" source_text="api.rubyonrails.org" %}

Basically the only way that `redirect_back_or_to` can reliably navigate the user back in history is to look at the `Referer` header. With that knowledge we've got a solution.

## The solution

We only need to adjust the original test slightly to get the desired result. Focusing on including the `referer` header when making the endpoint request ensures that we redirect to the correct location.

```ruby
patch user_passwords_path, params:
  {
    id: @user.id,
    user: {
      password: "invalid without numbers"
    }
  },
  headers: {
    referer: edit_user_passwords_url
  }
```

Note that instead of using `edit_user_passwords_path` we instead utilize the fully qualified url with the `_url` helper. This ensures that our test behaves the same way that the browser would.

With that everything is green and good to go.

Thanks for reading.
