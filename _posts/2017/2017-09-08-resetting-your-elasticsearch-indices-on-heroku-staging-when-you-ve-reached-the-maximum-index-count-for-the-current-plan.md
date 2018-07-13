---
layout: post
title: Resetting your Elasticsearch indices on heroku staging when you've reached
  the maximum index count for the current plan
category:
- fixes
tags:
- elasticsearch
- searchkick
- heroku
---

If you've ever used Heroku's low cost staging plan for Bonsai Elasticsearch you may have
ran into the following error: <code>reached maximum index count for current plan</code>.

Since in most cases your staging indices aren't as critical as say production (don't use
the fix below in production, please!), you can safely follow the procedure below for
removing and reindexing your search enabled models.
<!--excerpt-->

## Find your Elasticsearch URL
First you'll need to find your Elasticsearch url. If you are using the Bonsai
addon from Heroku you can do so in your terminal with: 

{% highlight ruby %}
# Where BONSAI_IVORY_URL is an environment variable set on heroku that
# points at your Elasticsearch url
# Note: Your specific plan might be different than IVORY
heroku config:get BONSAI_IVORY_URL -a your-staging-application
{% endhighlight %}

If you don't know what your environment variable is for BONSAI then you'll need
to go into Heroku's web UI and look at the config variables for it. It should
be in the format of <code>BONSAI_PLANNAME_URL</code>.

## Delete all available indices
The next step is to remove all the available Elasticsearch indices. 

**WARNING** DO NOT RUN THIS IN PRODUCTION! PLEASE, PLEASE, PLEASE DON'T DO THIS.

<blockquote>We offer a low cost staging plan for development and testing - Heroku</blockquote>

You can read more on this in [Elasticsearch's documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-delete-index.html). 
This can be accomplished via cURL, which you'll also need installed, from the following:

{% highlight terminal %}
# _all - refers to all available indices
# * - also works like _all
curl -XDELETE 'https://[bonsai_url_goes_here]/_all'
{% endhighlight %}

## Reindex all search enabled models
Lastly, we need to reindex all of your available models. If you are using Searchkick you can accomplish this
by running the following command. This will run through all of your searchkick
models 1-by-1 and reindex them. Otherwise, you'll need to reindex your content
based on whatever method you have configured.

{% highlight ruby %}
# All models with searchkick enabled
Searchkick.models.each do |model|
  model.reindex
end
{% endhighlight %}

That's it! You now have removed all the current indices and reindexed with new ones. Start searching!
