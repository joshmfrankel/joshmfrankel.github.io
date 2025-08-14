---
layout: post
title: Using Database Functions in Ruby on Rails Migrations
categories:
  - today-i-learned
tags:
  - postgresql
  - ruby on rails
---

Today I learned you can specify PostgreSQL functions within a Ruby on Rails migrations. I found this particularly useful for setting a random default value for a database column. Let's dig into a simple
example of this concept.

<!--excerpt-->

```ruby
add_column :table, :column, :text, default: -> { "('prefix-' || md5(random()::text) || 'suffix')" }, null: false, if_not_exists: true
```

The default uses block syntax and string interpolation in order to add a prefix and suffix to a random string. The **md5** and **random** functions come from PostgreSQL.

Know of any other tricks using functions within migrations? I'd love to learn about them in the comments below.

Thanks for reading!
