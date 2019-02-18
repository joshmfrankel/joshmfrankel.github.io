---
layout: post
title: Formatting Enum columns into a human readable format with SQL
category: today-i-learned
tags:
- Ruby
- Ruby on Rails
- SQL
---

Enum columns can be really handy when working with Rails. There are a lot of built-in helper methods that allow for quickly writing code and they make type-checking simple. However, when working with raw SQL they can be difficult to work with as they're stored as integers and as such lose their meaning. Keep reading to dig into a shortcut for staying SQL-land while returning an Enum in a human readable format.
<!--excerpt-->

First, let's say we have the following schema and model:

{% highlight sql %}
CREATE TABLE races (
  id integer,
  name string,
  race_type integer
)
{% endhighlight %}

{% highlight ruby %}
class Race < ApplicationRecord
  enum race_type: ["Road Race", "Fun Run", "Mud Run"]
end
{% endhighlight %}

Now if we use SQL to select data from the above table we'd end up with integers in the `race_type` field. These literally mean nothing to us and can't really be used for display.

{% highlight sql %}
SELECT *
FROM races;

id | name              | race_type
1  | "Warrior Dash"    | 2
2  | "Boston Marathon" | 0
3  | "Inflatable 5k"   | 1
{% endhighlight %}

We could iterate over the results and convert them to the corresponding value using Rail's built-in magic.

{% highlight ruby %}
Race.all.find_each do |race|
  puts race.race_type
end

# output:
Mud Run
Road Race
Fun Run
{% endhighlight %}

This is sufficient for most situations but let's assume for a one second that the above needs to be done in raw sql. Like a more complicated query with associations and aggregates. The following is a simple example but gets the point across.

{% highlight sql %}
# Additional schema
CREATE TABLE attendees (
  id integer,
  race_id integer,
  runner_id integer
)
CREATE TABLE runners (
  id integer,
  name string,
  age integer
)
{% endhighlight %}

{% highlight ruby %}
class Race < ApplicationRecord
  enum race_type: ["Road Race", "Fun Run", "Mud Run"]

  def race_results
    Race
      .left_joins(attendees: :runners)
      .select(
        :id,
        :name,
        :race_type,
        "
          COUNT(attendees.id) as total_attendees,
          AVG(runners.age) as average_runner_age
        "
      )
      .group(
        :id,
        :name,
        :race_type
      )
  end
end

# Here's the output sql from the above method
Race.race_results.to_sql #=> 
  SELECT races.id,
    races.name,
    races.race_type,
    COUNT(attendees.id) as total_attendees,
    AVG(runners.age) as average_runner_age
  FROM races
  LEFT OUTER JOIN attendees ON attendees.race_id = races.id
  LEFT OUTER JOIN runners ON runners.id = attendees.runner_id
  GROUP BY races.id, 
    races.name, 
    races.race_type
{% endhighlight %}

If we needed to return the `race_type` value from the query, how would we know which integer belongs to which value without having to check every record? Using a SQL case statement is a good solution for known key to value mappings.

{% highlight ruby %}
class Race < ApplicationRecord
  enum race_type: ["Road Race", "Fun Run", "Mud Run"]

  def race_results
    Race
      .left_joins(attendees: :runners)
      .select(
        :id,
        :name,
        "
          CASE races.race_type
            WHEN 0 THEN 'Road Race'
            WHEN 1 THEN 'Fun Run'
            WHEN 2 THEN 'Mud Run'
          END as race_type,
          COUNT(attendees.id) as total_attendees,
          AVG(runners.age) as average_runner_age
        "
      )
      .group(
        :id,
        :name,
        :race_type
      )
  end
end
{% endhighlight %}

Alright, we're done right? Not quite.

The above works correctly but doesn't plan for a future where new `race_type` Enums
could be added to `Race.race_types`. We want something that ensures the above
SQL continues to work even with changes.

We can use a joint ruby-sql solution in order to achieve a flexible query that is
considerate of future changes.

{% highlight ruby %}
class Race
  enum race_type: ["Road Race", "Fun Run", "Mud Run"]
  
  # ... more methods

  def race_type_case_statement_sql
    sql = "CASE races.race_type"
    Race.race_types.each do |key, value|
      sql = sql + " WHEN #{value} THEN '#{key.titleize}'"
    end
    sql = " END as race_type"
  end
end
{% endhighlight %}

The above uses the plural Enum helper for the column of race_type by calling 
`Race.race_types`. This gives us access to both the integer key along with the 
corresponding value. From that we can use a bit of meta-programming to construct
the SQL case statement.

Now the above query should look like:

{% highlight ruby %}
class Race < ApplicationRecord
  enum race_type: ["Road Race", "Fun Run", "Mud Run"]

  def race_results
    Race
      .left_joins(attendees: :runners)
      .select(
        :id,
        :name,
        "
          #{race_type_case_statement_sql},
          COUNT(attendees.id) as total_attendees,
          AVG(runners.age) as average_runner_age
        "
      )
      .group(
        :id,
        :name,
        :race_type
      )
  end

  def race_type_case_statement_sql
    # ...
end
{% endhighlight %}

We can further improve the method's readability using something like `#tap` or 
`#inject`.

{% highlight ruby %}
class Race
  enum race_type: ["Road Race", "Fun Run", "Mud Run"]

  def race_type_case_statement_sql
    "CASE races.race_type".tap do |sql|
      Race.race_types.each do |key, value|
        sql << " WHEN #{value} THEN '#{key.titleize}'"
      end
      sql << " END as race_type"
    end
  end
end
{% endhighlight %}

Now we're ready for the future!

![Back to the future](/img/2019/BTTF-See-you-in-the-future.gif)

Got another Enum trick? How about a single table inheritance one? I'd love to talk about it in the comments below.
