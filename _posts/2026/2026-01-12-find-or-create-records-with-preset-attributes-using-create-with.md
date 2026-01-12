---
layout: post
title: Find or Create Records with Preset Attributes using create_with
categories:
  - today-i-learned
tags:
  - ruby on rails
  - RabbitMQ
---

I've been working heavily with [RabbitMQ message broker](https://www.rabbitmq.com) infrastructure recently to coordinate events between two Rails applications. The event work involves maintaining synchronized data between specific shared data. In the past, I've implemented a `find_or_create_by` style of idempotent backfilling required associations. Today I learned about a separate syntax via `create_with` for presetting record creation attributes.

<!--excerpt-->

Below we have a simplified variation of the problem I encountered. This uses the [kicks gem](https://github.com/ruby-amqp/kicks) to process published RabbitMQ messages.

```ruby
class GroupCreateEventProcessor
  include Sneakers::Worker

  from_queue "group.create"

  def work(message)
    parsed_message = JSON.parse(message)

    creator = retrieve_creator(
      external_id: parsed_message[:external_creator_id]
    )
    group = Group.create(
      external_id: parsed_message[:external_id],
      name: parsed_message[:name]
      creator_id: creator.id
    )

    ack!
  end

  private

  def retrieve_creator(external_id:)
    User.find_by(external_id:)
  end
end

# app/models/user.rb
class User < ApplicationRecord
  has_many :created_groups

  # Additionally, email and external_id have a database level constraints of NOT NULL
  validates :email, presence: true
  validates :external_id, presence: true
end

class Group < ApplicationRecord
  belongs_to :creator, class_name: :User, foreign_key: :creator_id
end
```

What happens when we go to create a new Group, but the Creator (User) doesn't yet exist?

You'll end up with a validation error here since each Group above must reference a Creator. Our
**#retrieve_creator** method can reasonably return nil in the above case. To fix this we can use
**find_or_create_by** to adjust the method along with the required **email** field for the User
model validations.

```ruby
class GroupCreateEventProcessor
  include Sneakers::Worker

  from_queue "group.create"

  def work(message)
    parsed_message = JSON.parse(message)

    creator = retrieve_creator(
      external_id: parsed_message[:external_creator_id]
      email: parsed_message[:user_email] # Add additional message data from queue
    )
    group = Group.create(
      external_id: parsed_message[:external_id],
      name: parsed_message[:name]
      creator_id: creator.id
    )

    ack!
  end

  private

  # Ensure we create missing User records with `email` to satisfy the Model validations
  def retrieve_creator(external_id:, user_email:)
    User.find_or_create_by(external_id:) do |user|
      user.email = user_email
    end
  end
end
```

The above now functions as a self-healing flow for missing records that are required for newly
created Groups. Now I knew about the block syntax for **find_or_create_by** which allows you to
specify the creation attributes but what I learned about today was the **create_with** syntax.

## Create_with

The **create_with** method allows for preemptively setting attributes for future created records.
With this knowledge we can update our **#retrieve_creator** method signature to read a bit cleaner:

```ruby
  def retrieve_creator(external_id:, user_email:)
    User
      .create_with(email: user_email)
      .find_or_create_by(external_id:)
  end
```

This preserves the record finding specificity to only query for **external_id** while instructing
record creation to use the **user_email** coming from the RabbitMQ message. Now there are certainly
arguments stylistically for the block syntax vs create_with syntax but one thing the block style syntax
allows for is custom logic for generating values. Create_with does not provide as clean as a location
for custom logic within a block.

If you want to learn more here is the documentation entry for [ActiveRecord::QueryMethods#create_with](https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html#method-i-create_with)
