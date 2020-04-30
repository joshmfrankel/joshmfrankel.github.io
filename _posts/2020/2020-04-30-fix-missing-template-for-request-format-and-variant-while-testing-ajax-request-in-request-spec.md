---
layout: post
title: Fix "missing template for request format and variant" while testing Ajax request in a Request Spec
date: 2020-04-30 16:58 -0400
categories:
- today-i-learned
tags:
- RSpec
---

Request specs have a lot of benefits over traditional controller specs. They allow
for full stack testing at the controller level. One thing I recently ran
into was how to write this for AJAX requests.
<!--excerpt-->

[Take me to the solution](#solution)

If you're familiar with controller specs then you might have seen the following
format before:

``` ruby
  xhr :get, :new
```

For an endpoint that uses AJAX, I first tried the following request spec style:

``` ruby
  get users, params: { name: 'Geralt' }, as: :js
```

Knowing that this was a javascript request seemed logical to me. Unfortunately,
I was met with the following error:

```
Failure/Error: @app.call(env)

ActionController::UnknownFormat:
  MembershipsController#create is missing a template for this request format and variant.

  request.formats: ["text/html"]
  request.variant: []
```

Ok... So that message above tells us that the requested format was `text/html`. But
`as: :js` was specified since the template is an `.js.erb`.

## Solution

Coming full circle we need to specify that the request being made is an AJAX or
XHR style request. Adding the option `xhr: true` solves this issue.

``` ruby
  get users, params: { name: 'Geralt' }, xhr: true
```
