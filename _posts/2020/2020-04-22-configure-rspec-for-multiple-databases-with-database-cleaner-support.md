---
layout: post
title: Configure RSpec for multiple databases with Database Cleaner support
date: 2020-04-22 12:52 -0400
categories:
- tutorials
tags:
- RSpec
- ActiveRecord
- Rails
- Multiple Databases
---

When working with multiple databases, code can
become intertangled with records spanning across the systems. Seperating
database domain reponsibilities away from each other is challenging but a good
end goal. Sometimes this isn't possible which in the case of testing clean-up
can make for some tricky configuration details.
<!--excerpt-->

## Setting up multiple databases

First you'll want to create multiple `database.yml` files within your `config/`
directory. These should be named appropriately to your external databases. For example:

* statistics_database.yml
* events_database.yml
* database.yml (for core application)

[Unless you're running Rails 6.0](https://guides.rubyonrails.org/active_record_multiple_databases.html), multiple database support will be a manual process for schema and migration loading. Running SQL commands in the console isn't sustainable in the long run
so I recommend checking out [Robert Ostinelli's post](http://www.ostinelli.net/setting-multiple-databases-rails-definitive-guide/) which has a section on setting up
Rake tasks for each database.

Back to your `database.yml` files, they should each follow the best practices set forth from [Rails Guides](https://edgeguides.rubyonrails.org/configuring.html#configuring-a-database).

Once created, we can use namespaces to segment models based on database. This allows
for a discrete namespace to database mapping. I've written a brief example below
of what this would look like for the `statistics_database.yml` configuration within
the `Statistics` namespace.

``` ruby
module Statistics
  # Directly loads YAML configuration from filesystem by current Rails environment
  DATABASE = YAML.load_file(File.join(Rails.root, "config", "statistics_database.yml"))[Rails.env]

  class ApplicationRecord < ActiveRecord::Base

    # Ensure that this class:
    # * cannot be instantiated
    # * does not represent an available database table
    self.abstract_class = true

    # Connect to the secondary database
    # NOTE: We use this in the base class and have new models inherit from it
    #       to limit how many database connections are active at one time.
    establish_connection DATABASE
  end
end
```

`ActiveRecord::Base.establish_connection` or just `establish_connection` since we're
inside the ActiveRecord::Base scope, allows for direct connection to a new database
based on the supplied configuration.

`YAML.load_file(File.join(Rails.root, "config", "statistics_database.yml"))[Rails.env]` really just says: Build a filepath, supply it to YAML.load_file, and extract out environment
specific configuration values. Here's an example of it in the console:

```
YAML.load_file(File.join(Rails.root, "config", "statistics_database.yml"))[Rails.env]
=> {"adapter"=>"postgresql", "host"=>"localhost", "username"=>"postgres", "encoding"=>"unicode", "database"=>"statistics_development", "pool"=>5}
```

Finally `self.abstract_class` ensures we never directly use this class but instead have
other classes inherit from it. Additionally, it ensures that it doesn't map to a database
table.

Now that we've setup a second database connection what about testing multiple
databases?

## Database Cleaner Configuration

Database Cleaner is a fantastic gem for making test cleanup and teardown simple.
We're going to work with a simple configuration which looks like the following:

``` ruby
RSpec.configure do |config|
  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end

  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.start
  end

  config.append_after(:each) do
    DatabaseCleaner.clean
  end
end
```

Now presently this will only connect to the default database for the application.
Not a great situation if we want to ensure data remains cleaned up before/after
tests. Similar to the ApplicationRecord abstract model above, we'll need to
connect to the database using `establish_connection`. I've made the assumption that
you've followed the first part of the guide settings up constants in the respective
namespaces: `Statistics::DATABASE`, `Events::DATABASE`, `Core::DATABASE`.

``` ruby
RSpec.configure do |config|
  config.before(:suite) do
    ActiveRecord::Base.establish_connection Statistics::DATABASE
    DatabaseCleaner.clean_with(:truncation)
    ActiveRecord::Base.establish_connection Events::DATABASE
    DatabaseCleaner.clean_with(:truncation)
    ActiveRecord::Base.establish_connection Core::DATABASE
    DatabaseCleaner.clean_with(:truncation)
  end

  config.before(:each) do
    ActiveRecord::Base.establish_connection Statistics::DATABASE
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.start
    ActiveRecord::Base.establish_connection Events::DATABASE
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.start
    ActiveRecord::Base.establish_connection Core::DATABASE
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.start
  end

  config.append_after(:each) do
    ActiveRecord::Base.establish_connection Statistics::DATABASE
    DatabaseCleaner.clean
    ActiveRecord::Base.establish_connection Events::DATABASE
    DatabaseCleaner.clean
    ActiveRecord::Base.establish_connection Core::DATABASE
    DatabaseCleaner.clean
  end
end
```

Verbose no? We have to specify the Database Cleaner configuration for each database connection. We'll make this better in a second.

One thing that isn't clear from above is that the database connections above
are order dependent. Meaning **the last `establish_connection` call should be to
your core application database**. I'm not completely certain why this is but it seems
like without it the database won't be setup properly.

## Improving Database Cleaner

So the above is really wordy. Let's imagine that you have 10 external databases to
connect to. That is a lot of repetition. Luckily we can lean into the `yield` keyword
here to make it simple. Additionally, we can enforce the order of database connections.

``` ruby
EXTERNAL_DATABASE_CONNECTIONS = [
  Statistics::DATABASE,
  Events::DATABASE
]

RSpec.configure do |config|
  config.before(:suite) do
    connect_to_available_databases do
      DatabaseCleaner.clean_with(:truncation)
    end
  end

  config.before(:each) do
    connect_to_available_databases do
      DatabaseCleaner.strategy = :transaction
      DatabaseCleaner.start
    end
  end

  config.append_after(:each) do
    connect_to_available_databases do
      DatabaseCleaner.clean
    end
  end

  def connect_to_available_databases
    EXTERNAL_DATABASE_CONNECTIONS.push(Core::Database).each do |db_connection|
      ActiveRecord::Base.establish_connection db_connection
      yield
    end
  end
end
```

From the above we've defined a new array to contain all available external
database connections (`EXTERNAL_DATABASE_CONNECTIONS`).

Next, we have a new method `#connect_to_available_databases` which takes the
array and pushes the core (or default application) database onto the end of the
array to ensure it it loaded last.

We then establish_connection directly to ActiveRecord::Base like we've been doing.

Finally, the `yield` keyword allows us to use a block syntax and pass in the Database Cleaner
configuration to be executed against the currently connected database. Looking at the current code again, the following:

``` ruby
connect_to_available_databases do
  DatabaseCleaner.clean
end
```

will translate to

``` ruby
ActiveRecord::Base.establish_connection Statistics::DATABASE
DatabaseCleaner.clean
ActiveRecord::Base.establish_connection Events::DATABASE
DatabaseCleaner.clean
ActiveRecord::Base.establish_connection Core::DATABASE
DatabaseCleaner.clean
```

With this final change Database Cleaner is configured to run for multiple databases. Now
go write some tests!
