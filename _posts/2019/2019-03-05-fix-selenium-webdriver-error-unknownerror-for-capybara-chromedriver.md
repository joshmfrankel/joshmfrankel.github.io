---
layout: post
title: Fix Selenium::WebDriver::Error::UnknownError for Capybara chromedriver-helper
category: today-i-learned
tags:
- Capybara
- Chrome
---

Do you use the chromedriver-helper gem? How about Capybara? Have you seen 
this error `Selenium::WebDriver::Error::UnknownError: unknown error: call function result missing`?
If so then keep reading for how I fixed it for my local development.
<!--excerpt-->

chromedriver-helper enables a dead simple process for using headless Chrome
with Capybara. Recently, we upgraded to using it for our feature spec suite.
Unfortunately, after switching away from Poltergeist I started noticing errors
the looked like this:

{% highlight ruby %}
Selenium::WebDriver::Error::UnknownError:
  unknown error: call function result missing 'value'
    (Session info: headless chrome=72.0.3626.119)
    (Driver info: chromedriver=2.31.488763 (092de99f48a300323ecf8c2a4e2e7cab51de5ba8),platform=Linux 4.4.0-142-generic x86_64)
{% endhighlight %}

So what the heck does this mean?

My first thought was because I'm running Elementary OS which is based off of Ubuntu 16.04, that
I had come up against an incompatibility between my OS and the gem. Luckily, my second thought
was to check the different versions listed in the error message.

<blockquote>
headless chrome=72.0.3626.119<br>
chromedriver=2.31.488763
</blockquote>

Checking [chrome://settings/help](chrome://settings/help) told me that I was indeed
running **72.0.3626.119**. Which brings me to the first action I took.

## Update Chrome

I found out after downloading the 64-bit .deb package that the latest version of
chrome was **72.0.3626.121**. Just a few versions out of date between **.121** and **.119**. Still worth a shot. 

So step 1, I updated Chrome from the following [link](https://www.google.com/chrome/?brand=CHBD&gclid=CjwKCAiA2fjjBRAjEiwAuewS_YqearMUGJBpamtaMEsrbswwld6fR1Lro7LMjerLelq31yyHVdVJbxoCaVYQAvD_BwE&gclsrc=aw.ds). Next I relaunched and double checked [Chrome's settings](chrome://settings/help) to confirm
the update had been made. 

Unfortunately, the same spec failure persisted.

## Update chromedriver

I use asdf locally as my package version manager. So when I checked `which chromedriver` it
told me that it lived within the `/home/josh/.asdf/shims/chromedriver directory`. At this point
I tried a couple different things to install the newest version but all of them failed. I was
able to determine that the version I was using **2.31.488763** was out of date with the latest
**2.46** by running the following cURL command: `curl -sS chromedriver.storage.googleapis.com/LATEST_RELEASE`

Alright, so we're getting somewhere here. Looks like our version is still out of date. Hmm... That's when
I thought, "maybe I should read the documentation for chromedriver-helper". Which led to the following section
[_Updating to latest chromedriver_](https://github.com/flavorjones/chromedriver-helper#updating-to-latest-chromedriver). Duh.

Basically the way to update chromedriver helper is via a command:

<blockquote>
  chromedriver-update
</blockquote>

This finally upgraded my chromedriver version to `ChromeDriver 2.46.628388 (4a34a70827ac54148e092aafb70504c4ea7ae926)`. Going back to the feature spec I tried running them again and success! They worked! Well,
the specs still failed but chromedriver was no longer to blame. Time for some red-green-refactor
