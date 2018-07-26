---
layout: post
title: Constructing a select * from subquery using ActiveRecord
category: articles
tags:
- ruby on rails
- postgres
- ActiveRecord
- SQL
---

ActiveRecord is a great tool for making SQL a bit more readable. However, there
are some things that are really difficult to do in pure ActiveRecord. One of those is 
a <code>SELECT * FROM (subquery)</code> style of query. Before writing it in a raw
SQL query, check out the following tip for preserving some ActiveRecord goodness.
<!--excerpt-->

## Our database structure

![User to Artist attendances](/img/2018/select_all_from_subquery.png)

The way the above works is that a User is able to attend the Concert of a particular
Artist. An Artist <code>has_many</code> Concerts (since they perform in multiple locations) and a 
User can Attend as many Concerts as they want. A User is assumed as having attended a particular Artist's Concert if they have a valid Attendance record. The resulting models might look 
like the following code:

{% highlight ruby %}
class User < ApplicationRecord
  has_many :attendances
  has_many :attended_concerts, through: :attendances, source: :concert
  has_many :attended_artists, through: :attended_concerts, source: :artist
end

class Attendance < ApplicationRecord
  belongs_to :user
  belongs_to :concert
end

class Concert < ApplicationRecord
  has_many :attendances
  belongs_to :artist
end

class Artist < ApplicationRecord
  has_many :concerts
  has_many :attendances, through: :concerts
end
{% endhighlight %}

Now that we've defined a simple database structure and model code, let's say our task is to return a table that contains the following:

<blockquote class="Info Info-right"><strong>Association source option</strong><br />
  The :source option specifies the source association name for a has_many :through association. You only need to use this option if the name of the source association cannot be automatically inferred from the association name.
  <br>
  <cite><a href="https://guides.rubyonrails.org/association_basics.html#options-for-has-one-source">- guides.rubyonrails.org</a></cite>
</blockquote>

1. The first column should display the Artist of the Concert
2. Then the user whom attended
3. (Here's the tricky part) The user's latest attendance if they have seen an artist more than once.
4. We want Attendances sorted by newest first in the results
<div class="clearfix"></div>
Here's an example output:

| Artist        | Attendee         | Latest Attendance |
|---------------|------------------|-------------------|
| Pretty Lights | Sean Bean        | Mon, 16 Jul 2018 15:39:36 EDT -04:00 |
| Bassnectar    | Emilia Clarke    | Wed, 06 Jun 2018 15:42:02 EDT -04:00 |
| Kodomo        | Kit Harington    | Fri, 27 Apr 2018 15:42:17 EDT -04:00 |

For our above example, let's also specify that Sean Bean has attended a Pretty Lights concert three times in the last year. We only want to see his most latest attendance which happens to be July 16th, 2018 above. 

## Getting into the SQL

With the above specifications we could write some SQL code to query for the results like so:

{% highlight sql %}
SELECT art.name as artist,
  u.name as attendee,
  MAX(c.start_time) as latest_attendance
FROM users u
INNER JOIN attendances a ON a.user_id = u.id
INNER JOIN concerts c ON c.id = a.concert_id
INNER JOIN artists art ON art.id = c.artist_id
GROUP BY art.name, u.name
ORDER BY latest_attendance DESC
{% endhighlight %}

Here's the above SQL translated to ActiveRecord:

{% highlight ruby %}
User
  .all
  .joins(attendances: { concert: :artist })
  .select("
    users.name as attendee,
    artists.name as artist,
    MAX(concerts.start_time) as latest_attendance
  ")
  .group(:id, "artist")
  .order("latest_time DESC")
{% endhighlight %}

We're done! Right? Well not quite. As with any real world project, there's a new client request.

The client now wants to display the Concert's location along with latest attendance for each User / Artist combination. That seems like it would be as easy as adding location into the select clause and grouping by it with <code>.group(:id, "artist, latest_attendance")</code> but unfortunately that would only give you <code>latest_attendance</code> specified by unique <code>locations</code> as you can see below.

| Artist        | Attendee         | Latest Attendance | Location |
|---------------|------------------|-------------------|----------|
| Pretty Lights | Sean Bean        | Mon, 16 Jul 2018 15:39:36 EDT -04:00 | Los Angeles, CA |
| Pretty Lights | Sean Bean        | Sat, 14 Jul 2018 19:00:18 EDT -04:00 | Atlanta, GA |
| Pretty Lights | Sean Bean        | Tue, 10 Jul 2018 19:00:46 EDT -04:00 | Indianapolis, IN |
| Bassnectar    | Emilia Clarke    | Wed, 06 Jun 2018 15:42:02 EDT -04:00 | Houston, TX |
| Kodomo        | Kit Harington    | Fri, 27 Apr 2018 15:42:17 EDT -04:00 | Miami, FL |

This is great information but isn't quite accurate to the client's requirements. (Also, Sean Bean apparently has been flying all over the country)

<blockquote class="Info Info-right"><strong>PostgreSQL DISTINCT ON</strong><br />
  SELECT DISTINCT ON ( expression [, ...] ) keeps only the first row of each set of rows where the given expressions evaluate to equal. 
  <br>
  <cite><a href="https://www.postgresql.org/docs/9.5/static/sql-select.html">- postgresql.org</a></cite>
</blockquote>

In order to properly select the <code>latest_attendance</code> while keeping it in sync with the location we'll need to use a subquery to keep them associated with each other while filtering it to only return a single row. We can accomplish this with PostgreSQL's <code>DISTINCT ON</code> clause.

Using <code>DISTINCT ON</code> we can eliminate returned rows by making the results unique to the selected content. Let's see if we can write the proper SQL for this.

{% highlight sql %}
SELECT * FROM (
  SELECT DISTINCT ON(art.id)
    art.name as artist,
    a.name as attendee,
    c.start_time as latest_attendance
  FROM users u
  INNER JOIN attendances a ON a.user_id = u.id
  INNER JOIN concerts c ON c.id = a.concert_id
  INNER JOIN artists art ON art.id = c.artist_id
  ORDER BY art.id, latest_attendance DESC
) inner_query
ORDER BY inner_query.latest_attendance DESC
{% endhighlight %}

This could also be written as a [Common Table Expression](https://www.postgresql.org/docs/9.1/static/queries-with.html) using the <code>WITH</code> keyword in PostgreSQL. It could be considered a more readable query.

{% highlight sql %}
WITH inner_query AS (
  SELECT DISTINCT ON(art.id)
    art.name as artist,
    a.name as attendee,
    c.start_time as latest_attendance
  FROM users u
  INNER JOIN attendances a ON a.user_id = u.id
  INNER JOIN concerts c ON c.id = a.concert_id
  INNER JOIN artists art ON art.id = c.artist_id
  ORDER BY art.id, latest_attendance DESC
)

SELECT *
FROM inner_query
ORDER BY inner_query.latest_attendance DESC
{% endhighlight %}

At this point you could just run the above raw SQL directly using <code>ActiveRecord::Base.connection.execute(raw_sql)</code> and that would be perfectly reasonable way of accomplishing our goal. However, by doing this we lose <code>ActiveRecord's</code> built in helpers and chainability as we are instead return a <code>PG::Result object</code>.

{% highlight ruby %}
raw_sql = <<~SQL
  SELECT * FROM (
    SELECT DISTINCT ON(art.id)
      art.name as artist,
      u.name as attendee,
      c.start_time as latest_attendance
    FROM users u
    INNER JOIN attendances a ON a.user_id = u.id
    INNER JOIN concerts c ON c.id = a.concert_id
    INNER JOIN artists art ON art.id = c.artist_id
    ORDER BY art.id, latest_attendance DESC
  ) inner_query
  ORDER BY inner_query.latest_attendance DESC
SQL
result = ActiveRecord::Base.connection.execute(raw_sql)
result #=> #<PG::Result:0x000000000db346a8 status=PGRES_TUPLES_OK ntuples=5 nfields=29 cmd_tuples=5>
{% endhighlight %}

PG::Result object's are still pretty useful giving you access to a implementation of <code>#each</code>, <code>#values</code>, and <code>#to_a</code>. However, we lost scopes and a whole slew of ActiveRecord collection methods in the process. Let's fix that!

## Converting this back into ActiveRecord

Making this work in ActiveRecord is hard. There isn't a lot of clear guidance of how
to accomplish this. I was able to dig through enough forums and documentation until I happened upon the following entry: https://apidock.com/rails/v4.0.2/ActiveRecord/QueryMethods/from. 

The <code>ActiveRecord</code> from method generally just specifies the table you
are querying against. However, it also allows you to pass in built ActiveRecord collections and query from those. This is accomplishes by building the subquery and passing it into the <code>from</code> method on a valid ActiveRecord scope. 

{% highlight ruby %}
# This example builds equivalent SQL to the previous raw_sql example 
# with the SELECT * FROM (DISTINCT ON ...)
inner_query = User
  .all
  .joins(attendances: { concert: :artist })
  .select("DISTINCT ON(artists.id)
    artists.name as artist,
    users.name as attendee,
    concerts.start_time as latest_time
  ")
  .order("
    artists.id, 
    latest_time DESC
  ")

User
  .unscoped
  .select("*")
  .from(inner_query, :inner_query)
  .order("inner_query.latest_time DESC")
{% endhighlight %}

Note that in the above query we use the second argument in the from clause to specify
the alias for the table <code>.from(inner_query, :inner_query)</code>. This allows us to more easily order by the latest_time in the order by clause. 

Additionally, the <code>unscoped</code> just gives us a basic
ActiveRecord Association to use for querying. I used <code>User</code> as the starting
point above but really you can use any model to achieve the same results. This is
because we're only querying from the data within the subquery and not the actual <code>User</code> model directly.

## Conclusion

And there you have it. We've fully moved our subquery into ActiveRecord and can now enjoy all the conveniences that it provides. It took a little tweaking to the built
in ActiveRecord from clause by passing in a subquery but it works like a charm.

What did you think about this technique? Let me know by leaving a comment below and thanks for reading.
