---
layout: post
title: Ensure dropping a database table is reversible
category: 
- today-i-learned
---

When dropping a database table in a migration all data contained within the dropped table will be lost as well. This is to be expected since the table schema no longer exists. However, it is possible to make such a migration reversible so that at least the database structure is preserved.
<!--excerpt-->

The expectation here is that we can run <code>rails db:migrate</code> followed by <code>rails db:rollback</code> and recover the previous table's structure.

<blockquote class="Info Info-right"><strong>Note</strong><br>
I'm using Rails 5 in the above example which makes <code>rake</code> commands consistent with <code>rails</code> commands. This means that any <code>rake</code> command such as <code>rake db:migrate</code> has a counterpart of <code>rails db:migrate</code>.
</blockquote>

A naive first approach might look something like the following:

{% highlight ruby %}
class DropTableWidgets < ActiveRecord::Migration[5.0]
  drop_table :widgets
end
{% endhighlight %}

Unfortunately, the above results in the following errors after attempting to rollback the drop migration.

{% highlight ruby %}
DropTableWidgets: reverting ==========
rake aborted!
StandardError: An error has occurred, this and all later migrations canceled:

To avoid mistakes, drop_table is only reversible if given options or a block (can be empty).

Caused by:
ActiveRecord::IrreversibleMigration: 
{% endhighlight %}

As in the error message says above, drop_table can be reversible if you specify options.

Let's specify the specific columns that were on the original table as part of the drop table block.

{% highlight ruby %}
class DropTableWidgets < ActiveRecord::Migration[5.0]
  drop_table :widgets do |t|
    t.string name
    t.text description

    t.timestamps null: false
    t.index :name, unique: true
  end
end
{% endhighlight %}

With the addition of the table columns we can now rollback the database structure and retain the its original schema. Now we can migrate and rollback at will without worrying about losing the widgets table.
