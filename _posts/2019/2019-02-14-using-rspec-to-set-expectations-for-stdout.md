---
layout: post
title: Using RSpec to set expectations for stdout
category: today-i-learned
tags:
- RSpec
- Ruby
---

I've been working on a gem in my spare time for code coverage and one of its goals is to output coverage reports to stdout. The hope is that while running something like guard for RSpec, also having test coverage output during the red-green-refactor cycling will be useful. What I didn't realize beforehand is that output is fully testable within RSpec. Here's an example of ensuring a proper message is sent via `puts`.
<!--excerpt-->

{% highlight ruby %}
# lib/formatter.rb
class Formatter
  ...

  def print_coverage(covered_percent, lines_of_code)
    puts "#{covered_percent}% coverage, #{lines_of_code} total lines"
  end

  ...
end

# spec/lib/formatter_spec.rb
it "displays the coverage" do
  expect {
    formatter.format(simple_cov_result_stub)
  }.to output(/90% coverage, 127 total lines/).to_stdout
end
{% endhighlight %}

<blockquote class="Info">
  "The output matcher provides a way to assert that the block has emitted content to either
$stdout or $stderr."
<br>
<cite><a href="https://relishapp.com/rspec/rspec-expectations/v/3-8/docs/built-in-matchers/output-matcher">- relishapp.com</a></cite>
</blockquote>

This is accomplishing using [RSpec's output matcher](https://relishapp.com/rspec/rspec-expectations/v/3-8/docs/built-in-matchers/output-matcher). I could see this being useful for cli based gems the display feedback for developers.


