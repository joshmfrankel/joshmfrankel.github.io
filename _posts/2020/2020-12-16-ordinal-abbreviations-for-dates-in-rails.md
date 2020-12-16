---
layout: post
title: Ordinal abbreviations for dates in Rails
date: 2020-12-16 10:16 -0400
category:
- today-i-learned
tags:
- rails
- ruby
---

Working with Dates and Times as a developer can be a tricky business. Recently I ran into an interesting time format which consisted of day of week, abbreviation of month, and the ordinalized day of the month. If you're unfamiliar with ordinalized dates what it refers to are day of the month followed by a suffix. So 1st, 2nd, 3rd... I found a super cool trick to making this happen built into ActiveSupport::Inflector.

<!--excerpt-->

First off the format we're looking to replicate here is: `Monday, Dec 14th` based on a timestamp. Let's use the timestamp for this post "2020-12-16 12:49 -0400".

Assuming we have access to the string version of our date we can convert it to a DateTime object using the `parse` method. Once converted we can save it to a temporary variable which allows us to both format the first half of the output (day, month) and the ordinalized day of the month (16th).

Formatting the first part of the DateTime utilizes the `strftime` method along with the proper directives. In our case we need the following:

* `%A - The full weekday name ('Sunday')`
* `%b - The abbreviated month name ('Jan')`

For a full listing of directives [check out the docs here](https://ruby-doc.org/stdlib-2.6.1/libdoc/date/rdoc/DateTime.html#method-i-strftime).

Now for the ordinalization. [ActiveSupport::Inflector has a method, ordinalize](https://apidock.com/rails/Integer/ordinalize), which does exactly what we need here. We
can further improve upon this by using our previously parsed DateTime object's day method to retrieve the
numerical value for the day of the month. With these two together we ensure that we generate the correct date format
as well as only parsing the original date once.

Check out the below for how this is accomplished.

```ruby
def format_date(date:)
  parsed_date = DateTime.parse(date)

  parsed_date.strftime("%A, %b ") + parsed_date.day.ordinalize
end

format_date(date: "2020-12-16 12:49 -0400") #=> Monday, Dec 14th
```

Got any DateTime tricks up your sleeve? I'd love to hear about them in the comments.

41 45 62
