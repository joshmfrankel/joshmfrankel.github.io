---
layout: post
title: Automating Accessibility with Ruby and Axe Core
categories:
- tutorials
tags:
  - accessibility
  - testing
  - minitest
---

Building an accessible application can be challenging. Keeping an application accessible can be impossible. Without dedicated roles or full team support, accessibility regressions are easy to introduce.

In the past, I've used `eslint-plugin-jsx-a11y` for ensuring React components are accessible, and it helped tremendously. Recently, I've dived into Hotwire & Stimulus and needed a solution for ensuring an accessible application. Now as a disclaimer, all automated accessibility checkers have gaps between them and being fully accessible. Striving for a dedicated position or policy to ensure compliance is still something that automation can't replace. That being said, automation can help keep regresssion at bay.

<!--excerpt-->

## What makes for an accessible application?

First off, what does it mean for an application to be accessible. The Web Content Accessibility Guidelines 2.1 (or WCAG) outline four principles of accessible media.

* **Perceivable** - Information and user interface components must be presentable to users in ways they can perceive.
* **Operable** - User interface components and navigation must be operable.
* **Understandable** - Information and the operation of user interface must be understandable.
* **Robust** - Content must be robust enough that it can be interpreted reliably by a wide variety of user agents, including assistive technologies.

**Source**: [w3.org](https://www.w3.org/WAI/WCAG21/Understanding/intro#understanding-the-four-principles-of-accessibility)

{% include blockquote.html quote="If any of these are not true, users with disabilities will not be able to use the Web." source_link="hhttps://www.w3.org/WAI/WCAG21/Understanding/intro#understanding-the-four-principles-of-accessibility" source_text="Understanding the Four Principles of Accessibility" %}

Any website or application that doesn't adhere to the above, will create challenges
for those in need of accessible requirements.

So the above contain overarching principles of good accessible design, but what about
specifics? WCAG has that covered as well as part of a three tier compliance recommendation. The tiers are defined as:

* A - minimal compliance
* AA - general compliance
* AAA - high compliance

In order to meet one of the above tiers, all recommendations for that level must be met. Looking for a full list of all the recommendations? The following link provides in-depth explanations: https://www.w3.org/TR/WCAG21/#abstract

There is a wealth of information on Accessibility for the web out there, and I am by no means an expert. This post is merely the tip of the iceberg into the subject.

With the stage set, let's get into how to implement your own automated accessibility check.

## Initial Approach

The goal for this initiative was to enforce [WCAG](https://www.w3.org/WAI/standards-guidelines/wcag/) AA (at least) compliance programmatically. Either through a linter check or testing. While looking for a solution, I came up with two potential strategies:

1. Linting raw HTML elements within ERB templates
2. Testing application pages for accessibility compliance within feature test

Approach one felt more inline with how `eslint-plugin-jsx-a11y` works. That is linting the html that will be rendered. One downside with this is that it is further
away from the actual end-user. Additionally, I couldn't find anything that quite worked for checking ERB templates against accessibility requirements.

For approach two, I searched for a library that would help fail a testing suite as well as print what needed to be fixed. I discovered a company dedicated to this purpose called Deque. They have several solutions which are really handy for testing accessibility (one of which is a ruby gem). So with that found, I moved into implementation.

## Testing for accessibility

Deque's gem `axe-core` has a nice capybara plugin called `axe-core-capybara`. Now for those of you using RSpec there's additionally a great [axe-core-rspec](https://github.com/dequelabs/axe-core-gems/blob/develop/packages/axe-core-rspec/README.md) gem which takes care of configuration. For me, I was using minitest and needed a separate solution.

First off, let's add `axe-core-capybara` to our Gemfile. I'm also assuming you have capybara and webdrivers for running feature tests.

```ruby
# Gemfile
group :test do
  gem "capybara", ">= 3.26"
  gem "selenium-webdriver"
  gem "webdrivers"

  # Axe-core
  gem "axe-core-capybara"
end
```

Next, within your `ApplicationSystemTestCase` you'll need to require the axe matchers.

```ruby
require "axe/matchers/be_axe_clean"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  ...
end
```

From here what we require is to use the internal axe matcher within a similarly named dsl style method to what we see in Capybara. What I mean by that is, a naming convention that follows the `assert_` prefix but indicates we're testing for accessibility. The following gives us the new `assert_accessible` method to be used on the current page rendered.

```ruby
def assert_accessible(page, matcher = Axe::Matchers::BeAxeClean.new.according_to(:wcag21aa, "best-practice"))
  audit_result = matcher.audit(page)
  assert(audit_result.passed?, audit_result.failure_message)
end
```

Above, `matcher` can be overridden, but by default is the internal BeAxeClean matcher. Note that I have it set to AA compliance as well as best practices. For a description of all available rules, you can visit the [following documentation](https://github.com/dequelabs/axe-core/blob/master/doc/rule-descriptions.md#rule-descriptions).

`audit` refers to the internal `Axe::Api::Audit` class which can be [found here](https://github.com/dequelabs/axe-core-gems/blob/develop/packages/axe-core-api/lib/axe/api/audit.rb). This provides several nice helper methods like: `passed?` and `failure_message`. Using these we can generate a simple assertion by saying, if the matcher result doesn't pass then display the failure message from axe.

With that, we can utilize the new matcher for our feature tests like so:

```ruby
# Accessibility System test
require "application_system_test_case"

class AccessibilityTest < ApplicationSystemTestCase
  test "landing page is accessible" do
    visit root_path

    assert_accessible(page)
  end
end
```

## Bonus: Browser Extension for testing Accessibility

Now, axe doesn't just have libraries for auditing page accessibility, they also have a great Chrome extension. I highly recommend installing this if you plan to do automated accessibility testing, as it helps track down failing issues. Additionally, it can find accessibility recommendations that the automated testing above misses. [Download the extension here](https://chrome.google.com/webstore/detail/axe-devtools-web-accessib/lhdoppojpmngadmnindnejefpokejbdd?hl=en-US). The free version works great

![Axe DevTools example](/img/2022/axe-dev-tools-demo.png)

## Conclusion

We now have a test suite which can fail on Accessibility being outside of compliance. This will help reduce any accessibility regressions being introduced into the application. I don't think this is a full solution for ensuring accessibility compliance. As far as I've found, you need an actual person conducting accessibility auditing. So keep that in mind when working towards a more accessible application.

We've only scratched the surface of automated accessibility testing. What have you found to make accessibility testing easier? Is there a tool or process you love? I'd love to hear about them and discuss in the comments below.

Thanks for reading.
