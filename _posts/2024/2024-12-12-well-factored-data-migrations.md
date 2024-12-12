---
layout: post
title: Well-factored Data Migrations
categories:
- articles
tags:
- ruby
- sql
---

Large dataset migrations are always an adventure. A well factored data migration considers: Performance, Accuracy, Reversibility, and Repeatability. Each of these supports a successful data migration. Previously, I have determined these through running the SQL multiple times against a before and after state. While effective, it can get tedious, which is where testing comes into play.
<!--excerpt-->

## Performance

![Close up Photo of Mining Rig by https://www.pexels.com/@cookiecutter/](/img/2024/well-factored-data-migration/performance-pexels-cookiecutter-1148820.jpg "Close up Photo of Mining Rig by https://www.pexels.com/@cookiecutter/")

You might be thinking performance == speed from my heading. The speed in which a data migration runs at can be important, but is not the most important factor. Generally, a data migration can be run once and at off-peak usage hours for an application. This is not exactly what I mean by performance.

Performance refers to overall application performance and health. For example, if running this data migration will lock database tables or cause a high frequency of reads / writes, it could as a side effect impact current users. This makes it essential for safe data migrations that reduce locking, reads, and writes.

Several techniques for safer migrations include:

* Batch processing of multiple results
* Bulk operations within batching
* Working only with pertinent data by using effectively using SQL where clauses and db constraints
* Delayed processing for sensitive performance concerns (e.g., slow batching)
* Identifying efficient solutions by analyzing query planner cost and timing (e.g., `EXPLAIN ANALYZE`)

A data migration is no good however if it doesn't solve the underlying need accurately.

## Accuracy

![Woman Measuring with Yellow Pencil on Board by https://www.pexels.com/@cristian-rojas/](/img/2024/well-factored-data-migration/accuracy-pexels-cristian-rojas-8447773.jpg "Woman Measuring with Yellow Pencil on Board by https://www.pexels.com/@cristian-rojas/")

What is the before and after state of the database? What records do you need to work with to complete the migration? How will you know this was successful?

These are all questions to ask during a data migration process. A well factored data migration has the Engineer understanding the “Why” behind the use case. Did we add a new type of object users can interact with? Did the UI update a form field name, and we need to store existing and new data in it? Understanding why data moves between the before and after states, helps inform how you build the migration.

Part of maintaining accuracy, is ensuring you have a success metric. I always have some way of checking the dataset before and after the data migration runs to ensure all related records that require changing are changed correctly. This keeps the result accurate along with catching any edge cases that were missed. To go further, checking for accuracy several days / weeks after the data migration runs can be useful. Metabase is a great tool for creating reports that can indicate when data becomes out-of-sync.

## Reversibility

![Gear Lever by https://www.pexels.com/@rileyfranzke/](/img/2024/well-factored-data-migration/reversability-pexels-rileyfranzke-1822838.jpg "Gear Lever by https://www.pexels.com/@rileyfranzke/")

If the worst happens and running your data migration causes a severe issue, preparing a backup plan can save you a lot of stress. Asking questions such as:

* What happens if this changes the wrong records?
* What if causes a database lock or worse a deadlock?

Knowing the worst-case scenario during a data migration can help inform your strategy to implement them. Spending the same amount of brainstorming on having a rollback strategy as the actual migration is a good rule that I like to follow.

Now, not all data migrations are reversible. Whenever possible, I'd recommend crafting migrations to be reversible. Even migrations that seem to be irreversible can be by creating artifacts during the `up` process of the migration. For example, storing IDs for the effected records or taking a database snapshot before running the backfill.

Much like rolling back an application deployment, having a backup plan can be the difference between a blip of downtime versus a multi-hour outage.

## Repeatability

![Yellow Colonnade with Doors and Lamps by https://www.pexels.com/@enrique/](/img/2024/well-factored-data-migration/repeatability-pexels-enrique-24514435.jpg "Yellow Colonnade with Doors and Lamps by https://www.pexels.com/@enrique/")

Being able to re-run the data migration multiple times, and it not have a multiplicative effect on the data, is incredibly useful for making the migration easy to use. This is also important in the case of missed edge cases where you need to re-run the data migration BUT don't necessarily want to impact any of the properly migrated records.

Keeping your data migration idempotent is another way of encapsulating the idea of repeatability. I should be able to run the migration many times and have a deterministic result. Deterministic, meaning predictable output.

## So what?

Now that I've gone through my philosophy of well-factored data migrations, how can we make support this? Having to set up a before state, running the data migration, and then checking for successful after state works but can become tedious if each time you have to reset the data. This is where testing comes in, as it directly solves this problem. Generally, we don't think of migrations as testable, and most really aren't. Data migrations on the other hand are the exception and I believe SHOULD be tested.

Using RSpec we can generate a simple data migration spec which works by including the data migration directly in our `rails_helper`. The example below comes from me using the [excellent gem data-migrate](https://github.com/ilyakatz/data-migrate).

``` ruby
# db/data/123456789_backfill_new_column_on_users.rb
class BackfillNewColumnOnUsers < ActiveRecord::Migration
  def up
    # Backfills new column
  end

  def down
    # Reverts the backfill
  end
end

# spec/data/backfill_new_column_on_users_spec.rb
require "rails_helper"

RSpec.describe BackfillNewColumnOnUsers do
 let(:old_value) { "Old Value" }
 let(:updated_value) { "New Value" }

 it "syncs new column 'a' to 'b' (accuracy)" do
   user = create(:user, a: old_value)

   # Run the migration directly from the loaded migration files
   BackfillNewColumnOnTable.new.up

   # Ensure record is up-to-date post-migration
   user.reload

   expect(user.b).to eq updated_value
 end

 it "maintains idempotence (repeatability)" do
   user = create(:user, a: old_value)

   # Run twice to watch for errors, duplicate data, and invalid post-migration state
   BackfillNewColumnOnTable.new.up
   BackfillNewColumnOnTable.new.up

   # Ensure record is up-to-date post-migration
   user.reload

   expect(user.b).to eq updated_value
 end

 it "can be rolled back (reversibility)" do
   user = create(:user, a: old_value)

   BackfillNewColumnOnTable.new.up

   # Ensure record is up-to-date post-migration
   user.reload

   expect(user.b).to eq updated_value

   # Run the "down" portion of the migration
   BackfillNewColumnOnTable.new.down

   # Ensure record is up-to-date post-migration
   user.reload

   # Strict check
   expect(user.b).to be_nil

   # OR less strict
   expect(user.b).not_to eq updated_value
 end
end

# spec/rails_helper.rb
# Load all db/data classes
Dir[Rails.root.join('db/data/*.rb')].each { |f| require f }
```

Now assuming you have a setup / teardown phase configured in RSpec which resets your database, you now have functional spec coverage to ensure both Accuracy, Reversibility, and Repeatability. The idempotence block above is somewhat contrived but illustrates the idea of avoiding a migration causing side effects when repeated.

The two parts to remember from above are loading the migration and calling the migration's class.

``` ruby
# Loading migrations
Dir[Rails.root.join('db/data/*.rb')].each { |f| require f }

# Calling migrations
BackfillNewColumnOnTable.new.up
BackfillNewColumnOnTable.new.down
```

Performance is a bit trickier, as you'd need a dataset that is representative of production (ideally at 10x scale). For something like this, I'd recommend spec coverage which can be run on-demand instead of automatically. Fixtures and/or seeds can be beneficial in this case for generating a large volume of test data efficiently. You could then measure things like: query cost and query planner steps to determine success / failure. An easy step you can take here is to generate the raw SQL you plan on running and on a follow database running it with `EXPLAIN ANALYZE` to see what the resulting cost and query plan look like.

If you are using the `data-migrate` gem, I took an initial pass at implementing first-class support for [data migration testing in this pull request](https://github.com/ilyakatz/data-migrate/pull/355). Hopefully, the proposed baked in setup will help others efficiently craft their own test coverage.

Got any tips for working with large data migrations? Maybe a slick AI tool you know of? Let me know in the comments below to continue the conversation.
