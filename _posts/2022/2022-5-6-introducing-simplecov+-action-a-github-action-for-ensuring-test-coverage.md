---
layout: post
title: "Introducing SimpleCov+ Action: A Github action for ensuring test coverage"
categories:
- articles
tags:
  - testing
  - github
---

Testing your code thoroughly is an important part of a well functioning
and easy to change application. Lack of adequate test coverage can be frustrating
when refactoring, upgrading, or tracking down a bug. I've always wanted a way
to ensure that each file maintains
a minimum test coverage and if not fail continuous integration checks. After much
searching, I decided to go ahead and build my own. Introducing SimpleCov+ Action
for use within your Github actions.
<!--excerpt-->

## Why use it

So why would you want to use this in your build workflow?

SimpleCov+ Action provides a mechanism for reading SimpleCov results and using them
to either pass or fail your Github workflow. Many of the solutions I came across concerned me, as failing a build based on test coverage could become a blocker for new work. I wanted a solution that was configurable based on coverage threshold and was accurate for my test suite. In general, averages (as in average test coverage), are easily influenced by outlier results. This was another concern I had with using such a system.

![SimpleCov Basic line view](/img/2022/simple-cov-check-basic.png)

So here are the basic functions that I've built-in to this Github action.

### Minimum coverage threshold

By default, it will fail builds, if the overall test suite coverage is lower than 90%.
This can be configured using the `minimum_coverage` key and an integer value. Since
you can configure the value, you can slowly raise it as you cover more and more code.

![SimpleCov Basic line view](/img/2022/simple-cov-check-basic-detailed.png)

### Line vs Branch

As in the name, it relies on the SimpleCov gem, so that needs to be both
installed and configured correctly. With the newer versions of SimpleCov, you can
specify line vs branch coverage. This allows you to specify how well tested things
like conditional logic branches are vs simple line-by-line coverage.

The configuration value to change coverage type is called `minimum_coverage_type`

### Per file test coverage

If you also utilize the simplecov-json gem, you can activate the advanced mode, which will
fail builds based on file coverage percentage. This is really useful for
avoiding some well tested files throwing the results of our overall test suite.

## Why it's awesome

Test coverage is an important, albeit overlooked, part of a great test suite and healthy application.
SimpleCov+ Action helps in several key ways to improve overall confidence in your testing suite.

### It is incremental

Let's say you're adding test coverage to your application for the first time.
You're likely going to find many files that are lacking adequate test coverage (or if you're lucky just a few). By using `minimum_coverage` configuration, you can incrementally increase
test coverage a little, a bit at a time. I've personally used this to identify what my lowest covered file is and then set the `minimum_coverage` to 10% greater. This way you can chunk out the work necessary to increase overall test suite coverage a little at a time.

### It gives accurate results

Using the advanced version via simplecov-json, gives the ability to avoid situations where
one file is 100% covered and another is 20% covered, skewing your overall test suite coverage. Let's say your minimum coverage threshold is 50%. In the case above, the coverage would calculate to 60%, but you've got a file at 20%. That isn't a great metric for ensuring good test coverage when outliers throw results. That's why I recommend using the advanced file-by-file coverage version. Now for the above situation we'll keep the same threshold of 50%. Applied to the example, the build would now fail and display the file that only has 20% coverage as the failure point. This is much more useful for ensuring quality code coverage.

![SimpleCov Advanced detail view](/img/2022/simple-cov-check-advanced-detailed.png)

The screenshot above shows the result panel of the action, which will display each file
that failed to meet the minimum test coverage.

### It ensures new features are well tested

Having a minimum coverage necessary to deploy code, locks your application into a
base level of coverage. This means that if new features want to be released to production
they must contain proper test coverage. With this action in place, your code coverage
never drops below your minimum coverage threshold.

### It increases test suite confidence

Good test coverage ensures that refactoring code goes smoothly. There's nothing more
frustrating than realizing a large section of the codebase needs reworking only to find
out there is no test coverage of it. Now you have two jobs: refactoring the feature and adding the missing test coverage.
With test coverage becoming a priority, you reduce engineer frustration as well as make the system easier to change.

## So what are you waiting for?

Go ahead and try it out. You can start with a low `minimum_coverage` and work your way
up to a larger percentage. Every bit of coverage helps in the longevity of
an application.

The current version can be found in the [Github marketplace](https://github.com/marketplace/actions/simplecov-action). SimpleCov+ Action also utilizes itself to ensure the action's code maintains adequate test coverage. Because of this, there is a great example for how to set this up in your workflow. [Check out example configurations here](https://github.com/marketplace/actions/simplecov-action#example-configuration).

If you end up using this in your project, let me know how it goes in the comments below! Thanks for reading.
