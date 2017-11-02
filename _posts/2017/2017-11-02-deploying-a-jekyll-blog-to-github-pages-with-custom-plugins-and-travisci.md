---
layout: post
title: Deploying a jekyll blog to github pages with custom plugins and TravisCI
category: 
- tutorials
tags:
- jekyll
- travis-ci
- github pages
---

Deploying a new blog with Github Pages is a breeze. All you need is a new repo named after your Github username <username.github.io> and a jekyll site on your master branch. 

Github also kindly provides several whitelisted plugins for ease-of-use which [can be found here](https://pages.github.com/versions/). Unfortunately, some of these are a bit outdated (I'm looking at you <mark title="NO LONGER UNDER ACTIVE DEVELOPMENT">jekyll-paginate</mark>). More bad news is that because the plugins are whitelisted you can't use any other plugins. 

We're developers though, like a whitelist will stop us...
<!--excerpt-->

## Basic Setup  

If you just want a super simple Github Pages site and don't care about custom plugins, the guide that Github provides [here](https://pages.github.com/) will be more than adequate.

If you're interested in a more streamlined process keep reading and I'll walk you through automating a build/deploy process.

## Automated Site Deploy with TravisCI

To circumvent the plugin restriction, we'll automatically build our site on TravisCI using <code>jekyll build</code> and then pushing the static site up to our master branch on Github. It's a straight-forward process but can become tedious if you have to do it everytime you write a new post. The workflow is described in the below flowchart.

![Github Pages automatic build process](/img/2017/github-pages-build-process.png)

## Github Branch Setup

The **first thing we'll want to do is create a new branch off of master called *source***. (The naming here isn't important but choose something you'll remember). Once you've created your *source* branch there will be no need to use the master branch anymore for this repo.

*Source* will become where you write your blogs posts while master will remain where Github Pages loads your static site from. For example master will only include the <code>site/</code> directory while *source* will have the entire jekyll project.

I would highly **recommend changing your default branch in your repository to *source***. Optionally, you can make the *source* branch protected to prevent it from being accidently deleted or overriden.

![Github Branch Settings](/img/2017/github-branch-settings.png)

## Github Personal Access Token

Next, we'll want to give TravisCI something to authenticate against to perform
git pushes for us. To do this let's create a personal access token. 

<blockquote class="Info Info-right"><strong>Note</strong><br> You can also pass your personal access token
  securely by encrypting it into your <code>.travis.yml</code> file.</blockquote>

First click the **Settings** menu, located under your profile in the top right of Github. 

Next look for **Developer Settings** on the left sidebar and click it. 

Once there navigate to **Personal access tokens** and click the button **Generate new token**. Name the token something memorable (I use Travis CI) and set the permission scopes to everything under **repo**.

![Github personal access token settings](/img/2017/github-travis-permissions.png)

Make sure to copy your new token to your clipboard as we're going to need it later on.

## Jekyll site configuration

We need to configure a few things for our Jekyll site to function appropriately with TravisCI.

First add the following to your <code>Gemfile</code>.

{% highlight ruby %}
source "https://rubygems.org"
ruby RUBY_VERSION

# We'll need rake to build our site in TravisCI
gem "rake", "~> 12"
gem "jekyll"

# Optional: Add any custom plugins here.
# Some useful examples are listed below
group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-sitemap"
  gem "jekyll-paginate-v2"
  gem "jekyll-seo-tag"
  gem "jekyll-compose", "~> 0.5"
  gem "jekyll-redirect-from"
end

{% endhighlight %}

Any plugins we're using above in our <code>Gemfile</code> we'll also want to list in our Jekyll site <code>_config.yml</code> file. 

Additionally, we want to exclude certain files and directory so that they don't end up in the *master* branch once TravisCI builds the *source* branch.

Using the above example your <code>_config.yml</code> might look like:

{% highlight yml %}
title: Your blog title
email: your.email@gmail.com

# many other settings
# ...

# Any plugins within jekyll_plugin group from Gemfile
plugins:
  - jekyll-feed
  - jekyll-sitemap
  - jekyll-paginate-v2
  - jekyll-seo-tag
  - jekyll-compose
  - jekyll-redirect-from

# Exclude these files from the build process results.
# Prevents them from showing up in the master branch which 
# is the live site.
exclude:
  - vendor
  - Gemfile
  - Gemfile.lock
  - LICENSE
  - README.md
  - Rakefile

{% endhighlight %}

Because we're using our *master* branch to display our statically generated site, we'll want to remove the <code>site/</code> directory from tracking.

Add this to your <code>.gitignore</code>.

{% highlight ruby %}
.sass-cache
.jekyll-metadata
_site
{% endhighlight %}


Next, and most importantly, we need a <code>.travis.yml</code> file to let TravisCI know how we want it to run. I'm going to run through it line-by-line with explanations of the settings.

{% highlight yml %}
language: ruby
rvm:
- 2.3.1
install:
- bundle install
deploy:
  provider: pages
  skip_cleanup: true
  github_token: $GITHUB_TOKEN
  local_dir: _site
  target_branch: master
  on:
    branch: source
{% endhighlight %}

<code>language: ruby</code> Use the ruby language

<code>rvm: - 2.3.1</code> Use RVM to set ruby version to 2.3.1

<code>install: - bundle install</code> Run bundle install to install all gems.

<code>provider: pages</code> Use TravisCI's Github Pages provider

<code>skip_cleanup: true</code> Preserve files created during build phase.

<code>github_token: $GITHUB_TOKEN</code> Our personal access token. This is currently a reference to an environment variable which will be added in the TravisCI setup section below.

<code>local_dir: _site</code> Use all files found in this directory for deployment.

<code>target_branch: master</code> Push resulting build files to this branch on Github.

<code>on: branch: source</code> Only run TravisCI for this branch.

You can find additional information from the [deployment documentation](https://docs.travis-ci.com/user/deployment/pages/).

All of this together basically says, "Using the source branch from this repo, push all the files found within the site directory to the master branch of the repo". This only works by using the following <code>Rakefile</code> to manually build the site.

{% highlight ruby %}
# filename: Rakefile
task :default do
  puts "Building Jekyll site..."

  # Runs the jekyll build command for production
  # TravisCI will now have a site directory with our
  # statically generated files.
  sh("JEKYLL_ENV=production bundle exec jekyll build")
  puts "Jekyll successfully built"
end
{% endhighlight %}

The <code>Rakefile</code> above is run on every build. Because of this you can add other checks to this process such as [html_proofer](https://github.com/gjtorikian/html-proofer). These will be required to pass without failure before TravisCI will deploy the build. 

Let's move onto the last step. Setting up TravisCI for Github Pages.

## TravisCI Setup

Now that we have our site setup we need to hook it into TravisCI.

First you'll want to sign into TravisCI using your github account. This will give you a listing of all of your repositories.

**Find your repository and enable it**. It should be in the format of **username.github.io**.

![Enable CI for Repo](/img/2017/enable-ci.png)

As seen above, **click the gear icon** next to the newly enabled repository to go to the setting page.

Remember when I said this?

> Make sure to copy your new token to your clipboard as we're going to need it later on.

Now's the time to dig up that token code again.

<blockquote class="Info Info-right"><strong>Note</strong><br><code>GITHUB_TOKEN</code> needs to match your <code>.travis.yml</code> file's line <code>github_token: $GITHUB_TOKEN</code> to properly authenticate usage.</blockquote>

Once you have the code create a new environment variable named <code>GITHUB_TOKEN</code> on the TravisCI settings page. 

![Environment Variables](/img/2017/environment-variable.png)

Here's a screenshot of the generic settings that I recommend using for your build process. 

![General TravisCI settings](/img/2017/travis-ci-setting.png)

Once, you've got everything set up try out pushing a new commit to your *source* branch. You should see the TravisCI build start, pass, and eventually if you navigate to [http://username.github.io](http://username.github.io) your site will be live! If for some reason your build fails look through the **job log** for any details on errors. I'd be happy to help troubleshoot them in the comments.

![A passing build](/img/2017/travis-ci-pass.png)

That's it! Start blogging. 

## Optional: Custom Domains

You can give your readers a more customized experience by giving them a customized
domain to navigate to. This is accomplished by pointing your <code>username.github.io</code> site at a domain registrar where you have purchased a domain name. (e.g. [joshfrankel.me](http://joshfrankel.me/))

Don't worry if this sounds scary. It is actually easy to setup. [Namecheap has an excellent article that guides you through the entire process](https://www.namecheap.com/support/knowledgebase/article.aspx/9645/2208/how-do-i-link-my-domain-to-github-pages).

What you need to do is first create a CNAME file in your Github repository. The CNAME file should just hold your domain name in it. For example mine is below:

{% highlight ruby %}
# filename: CNAME
joshfrankel.me
{% endhighlight %}

This tells Github Pages where you site is being published at. Below I've listed the basic setup for what you'll need to do for other domain registrars. (From the Namecheap documentation)

> - A record for @ pointing to 192.30.252.153
> - A record for @ pointing to 192.30.252.154
> - CNAME record for www pointing to your username.github.io (the username should be replaced with your actual GitHub account username):

Is there something I could explain more? Were the steps above easy to follow? Got a Jekyll plugin you are digging? I'd love to hear about it in the comments below.

Thanks for reading.

