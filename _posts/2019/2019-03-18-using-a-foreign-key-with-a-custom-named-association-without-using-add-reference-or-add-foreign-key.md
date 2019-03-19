---
layout: post
title: Adding a foreign key with a custom named association without using add_reference or add_foreign_key
categories:
- series
tags:
- ruby on rails
---

<p itemprop="description">Using custom association names in Rails
  can be a great way of increasing understanding throughout an application. For example, it can allow for using a column called owner_id that references a User record. Let's dig into getting this setup for a migration.</p>
<!--excerpt-->

<blockquote class="Series">
  <h2>This is the <em>second</em> post in a series</h2>
  <ul>
    <li><a href="{{ site.baseurl }}/blog/custom-named-belongs_to-associations-in-rails-with-foreign-key-constraints/">1. Custom named belongs_to associations in Rails with foreign key constraints</a></li>
    <li>2. Using custom named association columns in a Rails migration</li>
  </ul>
</blockquote>

Previously, I've written about using `add_reference` and `add_foreign_key` in the above post. These are great and definitely accomplish the end goal, but recently I discovered an easier approach.

As a refresher, here's the previous post's example dataset:

<ul>
  <li>A User has_many Journals</li>
  <li>A Journal belongs_to a User</li>
</ul>

Along with the corresponding migration:

{% highlight ruby %}
# What the migration might look like
class AddCreatorToJournal < ActiveRecord::Migration
  def change

    # Notice how the index is for :creator but references users
    add_reference :journals, :creator, references: :users, index: true

    # Just like the belongs_to contained class_name: :User, the foreign key
    # also needs a specific custom column name as :creator_id
    add_foreign_key :journals, :users, column: :creator_id
  end
end
{% endhighlight %}

So to get started, let's run the following Rails generator command:

{% highlight ruby %}
# You can still use the :references syntax here
rails g model Journal creator:references

# The resulting database migration from the above command
class CreateJournals < ActiveRecord::Migration[5.0]
  def change
    create_table :journals do |t|
      t.references :creator, foreign_key: true

      t.timestamps
    end
  end
end
{% endhighlight %}

## Making a column reference a different table

So how do we hook up the above **:creator** column to the **User** model without using `add_reference`? It's actually pretty straight forward. First we need to let the reference know that it actually points at a table with a different name than the column. This is done via the `references: :table_name` syntax.

{% highlight ruby %}
class CreateJournals < ActiveRecord::Migration[5.0]
  def change
    create_table :journals do |t|
      t.references :creator, references: :users, foreign_key: true

      t.timestamps
    end
  end
end
{% endhighlight %}

The above should look really familiar as in the last post we used: `add_reference :journals, :creator, references: :users`. Above we just added `references: :users` to the existing t.references line.

## Hooking up a foreign key to a different table

Now that the table references the other one properly, we need to hook up its foreign key constraint. There's a lesser known syntax for specifying the table that a foreign key points to called `foreign_key: { to_table: :association }`. You can find more documentation on in within the [add_reference section](https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_reference) of the rails api. This is exactly what we need here in order to make the migration work.

So let's update the previous migration with the new `to_table` syntax.

{% highlight ruby %}
class CreateJournals < ActiveRecord::Migration[5.0]
  def change
    create_table :journals do |t|
      t.references :creator, references: :users, foreign_key: { to_table: :users}

      t.timestamps
    end
  end
end
{% endhighlight %}

The result from running the above migration is a great looking schema that properly 
creates a new creator_id column, references the users table, and constrains it with
a foreign key. See ya later `add_reference` and `add_foreign_key`

{% highlight sql %}
   Column   |            Type             | Collation | Nullable |               Default                
------------+-----------------------------+-----------+----------+--------------------------------------
 id         | integer                     |           | not null | nextval('journals_id_seq'::regclass)
 creator_id | integer                     |           |          | 
 created_at | timestamp without time zone |           | not null | 
 updated_at | timestamp without time zone |           | not null | 
Indexes:
    "journals_pkey" PRIMARY KEY, btree (id)
    "index_journals_on_creator_id" btree (creator_id)
Foreign-key constraints:
    "fk_rails_40d4de14e6" FOREIGN KEY (creator_id) REFERENCES users(id)
{% endhighlight %}

What's the most helpful SQL migration trick that you know? Tell me about it by leaving a comment below. Thanks for reading.

