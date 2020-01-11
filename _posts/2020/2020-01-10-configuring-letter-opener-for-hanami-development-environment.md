---
layout: post
title: Configuring Letter Opener for Hanami development environment
date: 2020-01-10 19:50 -0500
categories:
- today-i-learned
tags:
- Hanami
- Mailers
---

Recently, I've really dug into the Hanami framework. Coming from a Rails background
it is really fascinating to see what a Ruby framework looks like in a post-Rails world.
One of the first things I setup with new Rails projects is a process by which I can
test mailers in development. The letter_opener gem is perfect
for this purpose and luckily has a non-Rails setup guide.
<!--excerpt-->

## Installing the Letter Opener gem

First and foremost we need to add the **letter_opener** gem to our **Gemfile**. Version 1.7.0 is the current latest as of writing this. I use `~> 1.7` to limit the version number to be between `>= 1.7.0 and < 1.8.0`. This ensures any minor version changes aren't installed without this value being updated.

``` ruby
group :development do
  gem 'letter_opener', '~> 1.7'
end
```

Next you'll want to bundle the new gem by running `bundle install` on the command line.

Now that we've installed the gem in our project, we can begin setting up Hanami to start using it in the development environment.

## Configure Hanami to use Letter Opener

Hanami is a great lightweight Ruby framework. If you're unfamiliar with it, there is a great tutorial and documentation over at [https://hanamirb.org/](https://hanamirb.org/){:target="_blank"}. Assuming you're familiar or have read the guide let's move onto configuration.

Reading the letter_opener guide on [Non-Rails Setup](https://github.com/ryanb/letter_opener#non-rails-setup){:target="_blank"}, you'll see the following configuration:

``` ruby
  require "letter_opener"
  Mail.defaults do
    delivery_method LetterOpener::DeliveryMethod, :location => File.expand_path('../tmp/letter_opener', __FILE__)
  end
```

This is really close to how we'll configure this for Hanami.

The important things here are that we set the delivery method to use Letter Opener and we set the output location where the temporary email files live. The first argument to `delivery_method` contains the strategy and the keyword argument `location` sets the file location.

For Hanami mailer options are found within `project_name/config/environment.rb`. Open this file up.

First add we need to require letter opener at the top of the file.

``` ruby
require 'bundler/setup'
require 'hanami/setup'
require 'hanami/model'
require 'letter_opener' # add this line

...
```

Next, looking inside the `Hanami.configure` block, there are two ways you can configure but I only recommend the first.

## (Recommended) Method 1 - Development environment

If you look inside the `environment :production do` block you'll see that it contains a special `mailer do` block. Directly above the production environment setup is a `environment :development do` block. That's where we want to add our configuration code. This ensures that it only will send us emails when running in the development environment.

Here's the working setup code:

``` ruby
  environment :development do
    # See: https://guides.hanamirb.org/projects/logging
    logger level: :debug

    mailer do
      delivery LetterOpener::DeliveryMethod, location: File.expand_path('../tmp/letter_opener', __FILE__)
    end
  end
```

Now when an email is sent in development, it will instead be opened in the browser.

## (Alternative) Method 2 - All environments but production

The second method is not scoped to any one environment but rather configured in the main `mailer do` configuration block. This has the benefit of setting up Letter Opener for all environments but production (because production has its own special setup block already). Unfortunately, this benefit is also a downside as now mailers will be opened in your browser for other environments like test. This means running a spec that contains a mailer will open browser tabs. Not necessarily ideal if you have a lot of mailer specs.


You can find the main mailer setup by searching for the `mailer do` block which by default looks like:

``` ruby
  mailer do
    root 'lib/project_name/mailers'

    # See https://guides.hanamirb.org/mailers/delivery
    delivery :test
  end
```

Change the delivery line to use `LetterOpener::DeliveryMethod` and a set location and you're good to go.

``` ruby
  mailer do
    root 'lib/project_name/mailers'

    # See https://guides.hanamirb.org/mailers/delivery
    delivery LetterOpener::DeliveryMethod, location: File.expand_path('../tmp/letter_opener', __FILE__)
  end
```

What did you think? Was this a helpful tip for testing mailers in Hanami? I'd love to hear about other tips using Hanami in the comments below.

Thanks for reading.
