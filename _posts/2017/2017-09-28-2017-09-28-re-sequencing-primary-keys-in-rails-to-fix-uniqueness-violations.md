---
layout: post
title: Re-sequencing primary keys in Rails to fix uniqueness violations
category: 
- fixes
tags:
- ruby on rails
- sql
---

Recently, I ran into an issue with migrating data from one table to another incorrectly set the sequence for the new table's primary key. The issue stemmed from inserting all the data from one table into another.
<!--excerpt-->

Here's the offending migration that caused the issue:

{% highlight sql %}
# Copy all the data from first_tables into second_tables
execute <<-SQL
  INSERT INTO second_tables SELECT * FROM first_tables;
SQL
{% endhighlight %}

The above works great for copying data but unfortunately doesn't actually properly set the sequence for the primary key. Because of this trying to create new records in rails console was throwing uniqueness violation errors.

{% highlight sql %}
ActiveRecord::RecordNotUnique: PG::UniqueViolation: ERROR:  duplicate key value violates unique constraint "second_tables"
DETAIL:  Key (id)=(1) already exists.
{% endhighlight %}

Essentially, what the error translates to is that activerecord is attempting to insert a record with a primary key value of 1 but that value already exists. This violates the uniqueness constraint which every primary key has in usage.
 
Luckily, there was a fairly easy way to fix my bluster by calling <code>reset_pk_sequence!</code> on the table. This properly re-sequences the primary key so that the next time you insert a record it uses the appropriate value. 

{% highlight ruby %}
ActiveRecord::Base.connection.reset_pk_sequence!("second_tables") => "8"
{% endhighlight %}
