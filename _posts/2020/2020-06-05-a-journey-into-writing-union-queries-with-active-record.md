---
layout: post
title: A Journey into Writing Union queries with Active Record
categories:
- articles
tags:
- SQL
- Active Record
---

Active record has a multitude of features and niceties. From merging scopes to performing complex joins. However, sometimes it falls short. One shortcoming is a Union query.

When I've got a roadblock with active record, I reach for either raw SQL or Arel Tables. Both of these work well but tend to produce verbose code. That's why it's nice to stay in Active record for readability. Ok, so let's actually look at how to accomplish this.
<!--excerpt-->

## Potential solutions

Throughout this adventure I looked into several solutions to this conundrum. I considered using raw SQL. I dug into Arel Tables which have several promising approaches. I searched for a gem that handled union queries. In the
end I ruled each of these out in favor of rolling my own solution.

[If you're just looking for the solution click here.](http://joshfrankel.me/blog/a-journey-into-writing-union-queries-with-active-record#full-solution)

### Raw SQL, shouldn't this work?

Yes!

This approach would work for union queries. Something like the following:

``` ruby
result = ActiveRecord::Base.connection.execute("SELECT users.* FROM users WHERE users.active = 't' UNION SELECT users.* FROM users WHERE users.manager = 't'")
result.to_a #=> Collection of users from the union query
```

There are some downsides to this approach. Most importably you have to call `.to_a` on the result as otherwise you can't work with it. This means no ActiveRecord chaining or helper methods work on the result. [For more information see my older post on the topic.](http://joshfrankel.me/blog/constructing-a-sql-select-from-subquery-in-activerecord/)

Moving onto Arel tables next...

### Can we use Arel tables?

Recently, I did run into a roadblock with Active Record. In this case it was an upgrade to a newer version of Rails. The application had relied on
a custom method that dug deeply into Arel Tables (bind_keys I believe). Unfortunately, the features it was using became a private method rendering
the current solution unworkable for union queries. A rule I follow while adding code on top of a gem or third-party library, is
to only use the public api for exactly this reason. You can't depend on library features where you don't control that library unless
it is going to stick around for some time.

There are a lot of other Arel table solutions out there (many of which work) but I wanted to keep my solution high-level and for lack of better
phrasing a bit dumb. Some of the best solutions are clear, explicit code that is straightforward. Arel Tables add a layer
of complexity that is unnecessary due to its potentially shifting api.

So Arel Table solutions are out. Before we write our own solution, what about any gems?

## What about a gem?

I searched for a replacement gem for union in ActiveRecord and suprisingly didn't find anything recent that fit the bill except one. ActiveRecordExtended. [ActiveRecordExtended](https://github.com/GeorgeKaraszi/ActiveRecordExtended/) it does a nice job of extending many of the underlying
Postgres features into ActiveRecord-land. Since Rails is database agnostic it includes many previously unsupported features. It even supports union queries!

So why did I not go with this option? Unfortunately, while a really stellar gem, it has 1 pretty nasty gotcha associated with
union queries. That is merging of scopes. I'll show you what I mean.

``` ruby
User.where.not(updated_at: nil).merge(
  User.union(
    User.where(name: "Jon")
  ).union(
    User.where(active: true)
  )
).order(start_date: :desc)
```

From above, you might expect the resulting SQL to look like:

``` sql
SELECT users.*
FROM (
  SELECT users.*
  FROM users
  WHERE users.name = 'Jon'
  UNION
  SELECT users.*
  FROM users
  WHERE users.active = 't'
) subquery
WHERE subquery.updated_at IS NOT NULL
ORDER BY subquery.start_date DESC
```

Unfortunately, ActiveRecord has a hell of a time deciding what to do with a passed in union to a merged scope. It ends
up generating:

``` sql
SELECT *
FROM users
WHERE users.updated_at IS NOT NULL
ORDER BY users.start_date DESC
```

Effectively swallowing the union query and producing a defective query. And this doesn't raise an error when it happens
meaning it is silently excluding results. Ouch!

[For more details see the open issue here.](https://github.com/GeorgeKaraszi/ActiveRecordExtended/issues/35)

Alright, gems are out. Guess we'll have to write our own.

## Writing a simple ActiveRecord Union helper

Let's line up our goals for this helper.

1. Method exists within ActiveRecord for ease-of-use
2. Method that can be called to union two relations together
3. Method supports merging of ActiveRecord scopes

Now before we dig into writing an ActiveRecord helper, let's briefly describe our
goal we're looking to achieve by defining a Union.

### Anatomy of a Union

When working with SQL unions, the goal is to combine multiple datasets into one collection as a single query. Generally, they follow the format of:

``` sql
SELECT table_a.column_1, table_a.column_2
FROM table_a
UNION
SELECT table_b.column_1, table_b.column_2
FROM table_b
```

<blockquote class="Info Info-right">
"UNION removes duplicate rows but UNION ALL does not remove duplicate rows. UNION ALL just merges all the rows that satisfy all the conditions."
<cite><a href="https://kb.objectrocket.com/postgresql/postgresql-union-vs-union-all-534">- Object Rocket</a></cite>
</blockquote>

The requirements of a **UNION** say that each side of the union selects the same number of columns with the same names. This also means that any aliased columns must also match. Along with **UNION** there is also **UNION ALL**. The difference between these two is that **UNION** returns distinct results while **UNION ALL** returns duplicates.

``` sql
SELECT table_a.column_1 as cheezeburger
...
UNION
SELECT table_b.column_1 as cheezeburger
...
```

Now that we've got basic UNION knowledge we can jump back into writing our own
implementation

### Extending ActiveRecord

Our first stop is to create a initializer so we can utilize the new method anywhere within ActiveRecord.

``` ruby
# config/initializers/active_record_union.rb
module ActiveRecordUnion
  extend ActiveSupport::Concern

  class_methods do
    def union
    end
  end
end

# Inject our module into the top level ActiveRecord::Base class
# This allows us to to write functionality for ActiveRecord objects
ActiveRecord::Base.send(:include, ActiveRecordUnion)
```

The above satisfies our first requirement, _Method exists within ActiveRecord for ease-of-use_. We can now call something like `Model.union` and have it return
`nil` instead of method missing.

``` ruby
User.union #=> nil
User.all.union #=> nil
```

We've included ActiveSupport concerns, in order to gain the nice **class_methods** syntax. All this does is include the new union method as a class level method that
can be called directly on a model or relation.

Additionally, at the end of the file we call the include method on **ActiveRecord::Base** which effectively allows us to inject the new module into
ActiveRecord. This is what makes the union method available.

Here comes the fun part. Now we have to construct the union query.

### Unioning multivariate relations

The basic premise here is that we want to craft a union query by combining the
output SQL from multiple relations of the same data type. Or rather, we can
create a stupid builder method that calls `.to_sql` and joins it together
into valid SQL. Simple right?

Here's the naive first attempt to make it work (we'll go through this line-by-line)

``` ruby
module ActiveRecordUnion
  extend ActiveSupport::Concern

  class_methods do
    def union(*relations)
      mapped_sql = relations
        .map(&:to_sql)
        .join(") UNION (")

      unionized_sql = "((#{mapped_sql})) #{table_name}"

      from(unionized_sql)
    end
  end
end

ActiveRecord::Base.send(:include, ActiveRecordUnion)
```

<blockquote class="Info Info-right"><strong>Watch the database calls</strong><br>
  ActiveRecord doesn't execute SQL until right before it it used. We leverage this
  optimization by calling `.to_sql` on relations which DOES NOT make a database call.
  Rather it deciphers the underlying Arel representation of final SQL.
</blockquote>

First off we specify that the method definition accepts a splatted parameter `union(*relations)`. This allows us to pass in an infinite number of relations with
each separated by a a comma: `User.union(n1, n2, n3, ...)`. Useful for variable number of relations to add to the union.

``` ruby
mapped_sql = relations
  .map(&:to_sql)
  .join(") UNION (")
```

Above we iterate through available relations and call `to_sql` on them to convert
them into an array full of stringified SQL results. ActiveRecord responds to `to_sql` by producing the
resulting SQL. Here's an example call:

``` ruby
> User.union(User.where(name: "Nic Cage"), User.where(active: false))
mapped_sql = relations.map(&:to_sql)
#=> ["SELECT * FROM users WHERE name = 'Nic Cage'", "SELECT * FROM users WHERE active = 'f'"]
```

We can use this array as a temporary storage solution for generating our new SQL statement.

The next part `.join(") UNION (")` looks bananas. Really we just join
our array back into a single string where each element has `) UNION (` between
them. What we want is a sql fragment. Here's an example output:

``` ruby
"SELECT * FROM users WHERE name = 'Nic Cage') UNION (SELECT * FROM users WHERE active = 'f'"
```

This ensures that we can programatically build the union for as many elements
are in the relations array. Next comes the outer parenthesis setup.

``` ruby
unionized_sql = "((#{mapped_sql})) #{table_name}"
```

First we surround the UNION sql with a `( ... )` as we want the end result part of the same statement such as: `SELECT * FROM ( ...union... )`.

Next there's a another bit of parenthesis interpolation.
```
(#{mapped_sql})
```
Since we're inside a string we had to interpolate
`mapped_sql`. We've also specified an opening and closing parenthesis. These correspond
directly to the partially constructed sql above. The previous `sql...) UNION (...sql` string
from the joined array now fits into the opening/closing parenthesis and becomes valid
SQL.

``` ruby
"... #{table_name}"
```

We use an ActiveRecord method called `table_name` which returns the database
representation of the table. This is based on the calling relation. So for `User.union` the plural version would be (users) of the model (User). This works
because `self` in the above intializer context is the model. The importance of this is that the inner SQL string will be identifying its base
table to query from. By also specifying that the union is aliased
as `table_name` we can ensure we select properly from the union without naming collisions.

``` ruby
unionized_sql = "((#{mapped_sql})) #{table_name}"

#=> "((SELECT * FROM users WHERE name = 'Nic Cage') UNION (SELECT * FROM users WHERE active = 'f')) users"
```

The final line `from(unionized_sql)` also relies on the calling relation to determine
where it will be selecting from. In other words we can substitute the table to query from to instead be the result of our union sql. Running with our `User.union` example this would
translate to:

``` ruby
> User.union(User.where(name: "Nic Cage"), User.where(active: false)).to_sql
#=> SELECT * FROM ((SELECT * FROM users WHERE name = 'Nic Cage') UNION (SELECT * FROM users WHERE active = 'f')) users
```

Notice that both the inner queries along with the alias for the `FROM (...union...) alias` both refer to the same table name. This is a little trick to allow the outer query to request records from the inner unioned query. Without this the syntax
will be invalid and you'll end up with errors such as:

```
# PG::SyntaxError:
#   ERROR:  subquery in FROM must have an alias
```

Now with the above we can run code like the following and gather results from
a union. This preserves the `where.not(updated_at: nil)` line effectively filtering
out the results of the inner union query by the outer condition.

``` ruby
User.where.not(updated_at: nil).merge(
  User.union(
    User.where(name: "Jon")
  ).union(
    User.where(active: true)
  )
).order(start_date: :desc)

#=> SELECT users.* FROM ((SELECT users.* FROM users WHERE name = 'Jon') UNION (SELECT users.* FROM users WHERE active = 't')) users WHERE users.updated_at IS NOT NULL
```

This fulfills our last condition _Method supports merging of ActiveRecord scopes_.

So we're done right? Well not quite.

## Gotcha!

Anytime you bake new functionality into a library you don't control you run
the risk of having to support everything the library supports. Above we have
a couple issues that aren't super evident but I'll run through them with examples below.

### No arguments

First up, anytime you use the splat (*) operator as a parameter you are essentially
saying that the parameter is optional. So `User.union(nil)` won't raise an error. Unlike traditional parameters, we'll want
to add the following code in order to prevent method calls without parameters:

``` ruby
def union(*relations)
  raise ArgumentError, "wrong number of arguments (given 0, expected 1+)" if relations.empty?

  # mapped_sql = relations
  #  .map(&:to_sql)
  #  .join(") UNION (")

  # unionized_sql = "((#{mapped_sql})) #{table_name}"

  # from(unionized_sql)
end
```

### ActiveRecord.none method

What happens if you run the valid ActiveRecord method `.none` within our
union method? `User.union(User.active, User.none)`. Well bad things. User.none returns an empty relation. What's worse is that User.none.to_sql is simply just `""`. Having one side of a union be empty will result in:

``` ruby
User.union(User.none).first
# ActiveRecord::StatementInvalid: PG::SyntaxError: ERROR:  syntax error at or near ")"
# LINE 1: SELECT  "users".* FROM (()) users
```

We'll solve this by only using relations that produce SQL strings and ignoring the rest. The **select** block below does this by checking to see if to_sql produces
something other than an empty string.

``` ruby
def union(*relations)
  # raise ArgumentError, "wrong number of arguments (given 0, expected 1+)" if relations.empty?

  valid_relations = relations
    .select do |relation|
      relation.to_sql.present?
    end

  # mapped_sql = valid_relations
  #   .map(&:to_sql)
  #   .join(") UNION (")

  # unionized_sql = "((#{mapped_sql})) #{table_name}"

  # from(unionized_sql)
end
```

You might be thinking why not use something like `relation.respond_to?(:to_sql)` here. We don't use this because `User.none` responds to **to_sql**. We
want to make sure that when it does respond that it is something other than an empty string.

### Data type mismatch

How about the case where someone uses a different model as either the base model
or the ones inside the union?

``` ruby
User.union(OtherModel.all, OtherModel.where(some: true))
```

In this case our previous trick for **table_name** will cease to work as the database tables for each of the models above are different. We'll want to handle this situation with a helpful error message.

``` ruby
def union(*relations)
  # raise ArgumentError, "wrong number of arguments (given 0, expected 1+)" if relations.empty?

  valid_relations = relations
    .select do |relation|
      raise ArgumentError, "type mismatch. Base model table #{table_name} does not match table #{relation.table_name} of at least one relation" if table_name != relation.table_name

      relation.to_sql.present?
    end

  # mapped_sql = valid_relations
  #   .map(&:to_sql)
  #   .join(") UNION (")

  # unionized_sql = "((#{mapped_sql})) #{table_name}"

  # from(unionized_sql)
end
```

This will help guide future engineers on proper usage of the method.

### Merging associations

This is a nasty one. The situation is that we have a model which we're working
from an association to merge in a union query. Hopefully doesn't happen
often but I did run into it. Here's what this might look like:

``` ruby
User
  .joins(:posts)
  .merge(
    Post.union(
      Post.where(published: true),
      Post.where(has_comments: true)
    )
  )
```

The above is trying to constructs some conditions on the `inner join` from User to
Posts. The goal with a query like this is to return Users which have at least one Post associated with them and are either published or have comments. The published and have comments conditions are merged into the result. Translated to SQL this is what the above is attempting to accomplish:

``` sql
SELECT *
FROM users
INNER JOIN posts p ON p.user_id = u.id
WHERE posts.id IN(
  SELECT posts.id
  FROM (
    (
      SELECT posts.*
      FROM posts
      WHERE posts.published = 't'
    )
    UNION
    (
      SELECT posts.*
      FROM posts
      WHERE posts.has_comments = 't'
    )
  ) posts
)
```

Unfortunately, our code current produces the following SQL exception.

```
PG::DuplicateAlias: ERROR:  table name "posts" specified more than once
 : SELECT "users".* FROM ((SELECT "posts".* FROM "posts" WHERE "posts"."name" = 'keyword') UNION (SELECT "posts".* FROM "posts" WHERE "posts"."name" = 'matched')) posts INNER JOIN "posts" ON "posts"."user_id" = "users"."id"
```

Notably the resultting SQL is a bit non-sensical. Look at the following: `SELECT "users".* FROM ((SELECT "posts".* FROM ...)) posts`. This is problematic because
we being selecting from the `users.*` alias on a FROM clause that is aliased as
`posts`. Additionally, `posts` is seen as specified more than once in the query
because it is the alias used for the from clause as well as an inner join onto
the table. Due to this SQL can't figure out which table is the correct one and
throws an exception.

So how do we fix this?

A key piece of information is that merged scopes work well with where clauses. Often
you'll see scopes defined in models used directly for `.merge`. An example can
be seen above:

``` ruby
  .merge(
    Post.union(
      Post.where(published: true),
      ...
    )
  )
```

We can leverage this fact with our implementation's output. Above we noticed that
the from clause was non-sensical. Combining both of these facts we can still
gather unioned relations albeit in a slightly different format.

Previously I stated, "we can substitute the table to query from to instead be the result of our union sql". Taking this to the next logical step from what we've learned, we can transition to using the better supported `.where(id: ...)` syntax
here.

``` ruby
from(unionized_sql)
#=> SELECT "users".*
#=> FROM ((...union 1...) union (...union 2...)) union_table_name
#=> INNER JOIN union_table_name ...

# Updated
where(id: unionized_sql)
#=> SELECT "users".*
#=> FROM users
#=> INNER JOIN posts ON posts.user_id = users.id
#=> WHERE posts.id IN (... unionized_sql ...)
```

Now instead of the from clause specifying where we're querying data from, we instead use a subquery for selecting out the available ids of the table. The resulting
SQL works and better supports the `.merge(scope)` syntax due to its reliance on
the where id subquery approach.

### Full Solution

``` ruby
module ActiveRecordUnion
  extend ActiveSupport::Concern

  class_methods do
    def union(*relations)
      raise ArgumentError, "wrong number of arguments (given 0, expected 1+)" if relations.empty?

      valid_relations = relations
        .select do |relation|
          raise ArgumentError, "type mismatch. Base model table #{table_name} does not match table #{relation.table_name} of at least one relation" if table_name != relation.table_name

          relation.to_sql.present?
        end

      mapped_sql = valid_relations
        .map(&:to_sql)
        .join(") UNION (")

      unionized_sql = "((#{mapped_sql})) #{table_name}"

      where(id: from(unionized_sql))
    end
  end
end

ActiveRecord::Base.send(:include, ActiveRecordUnion)
```
Here's our task list that we set out to do at the beginning.

1. (Completed) Method exists within ActiveRecord for ease-of-use
2. (Completed) Method that can be called to union two relations together
3. (Completed) Method supports merging of ActiveRecord scopes

And that's it!

## Conclusion

That's how I solved a ActiveRecord union problem with some simple ruby code. Nothing
fancy, just straight to the point. This was focused on using public api methods that
are unlikely to change making it more resilient to future changes.

I did leave out one important piece which is I used Test Driven Development throughout
this entire process to ensure that no regressions occured. That's one of the ways
I tracked down several of those gotcha cases. Testing this was an interesting case
as well because a requirement of it was to not rely on any existing models so that
the tests would be independent of the active record layer. I'd be happy to write
a follow-up or discuss what I learned from testing the above.

What'd you think about this approach? I'd love to discuss any questions you have
on it.

Thanks for reading!

