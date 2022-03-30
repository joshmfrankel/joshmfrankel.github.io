---
layout: post
title: Tips I learned working with ActiveStorage
categories:
- today-i-learned
tags:
  - rails
---

ActiveStorage is a great system for configuring your cloud file storage. It takes
care of much of the details and allows you to focus on the most important parts: file upload and file download. Recently, I had the opportunity to work through an uploader on Rails 7 within the Administrate admin interface. Fun journey that left me with a couple tricks up my sleeve the next time I reach for ActiveStorage.

<!--excerpt-->

## Table of contents
Only care about one tip? Here's a table of contents to get you moving along

* [Setting a custom filename](#setting-a-custom-filename)
* [Querying records with existing file attachments](#querying-records-with-existing-file-attachments)
* [Adding bi-direction polymorphic association between model and attachment](#adding-bi-direction-polymorphic-association-between-model-and-attachment)
* [Uploading files within the Administrate gem](#uploading-files-within-the-administrate-gem)

## Setting a custom filename

File asset can be easily downloaded using the `link_to` helper. For example:

```ruby
link_to "Link Text", url_to_asset, download: "your custom filename"
```

However, when working with ActiveStorage the same mechanism does not work. Instead the following syntax is necessary:

```ruby
link_to rails_blob_path(model_object.file, disposition: "attachment")
```

{% include blockquote.html quote="To create a download link, use the rails_blob_{path|url} helper. Using this helper allows you to set the disposition." source_link="https://edgeguides.rubyonrails.org/active_storage_overview.html#serving-files" source_text="Rails Guides Active Storage Overview" %}

While using `rails_blob_path` for ActiveStorage asssets be default you can't adjust filenames. One method which works well to allow more control over filename is to modify files on upload. Below is a simple `after_save` callback which makes this happen:

```ruby
class MyModel < ApplicationRecord
  has_one_attached :file

  def file_attached?
    file.attached?
  end

  after_save :set_filename, if: :file_attached?

  private

  def set_filename
    extension = file.filename.extension
    file.blob.update(filename: "#{name}.#{extension}")
  end
end
```

Assuming you have an attached virtual field called "file" and a column called "name", the above will update
the file blob's filename upon saving. Note the preservation of file extension as part of the update.

With the above in place you can utilize

## Querying records with existing file attachments

ActiveStorage provides a nice scope for avoiding n+1 queries for objects with files called `with_attached_file`. Now this works great for eager loading but what about querying by records that have a valid file attachment? I dug into the internals of ActiveStorage::Attachment to discover that the way in which it works is a polymorphic join onto record. Record in this case being whatever is set to `has_one_attached :file`. With this knowledge, we can derive a query using arel:

```ruby
has_one_attached :file

def self.with_attachments
  active_storage_arel = ActiveStorage::Attachment.arel_table
  with_attached_file
    .joins(
      arel_table.join(active_storage_arel)
      .on(
        arel_table[:id]
          .eq(active_storage_arel[:record_id])
          .and(active_storage_arel[:record_type].eq(name))
      )
      .join_sources
    )
end
```

The above uses the eager loading of `with_attached_file` along with polymorphically joining onto the ActiveStorage::Attachment from the current table. This is accomplished from the following:

* Utilize `arel_table[:id]` to represent the current model's id
* `name` refers to the current model's database table name
* Retrieving `ActiveStorage::Attachment` as an arel table

## Adding bi-direction polymorphic association between model and attachment

While I was digging into the querying example above, I also noted that `ActiveStorage::Attachment` contains the following association:

```ruby
belongs_to :record, polymorphic: true, touch: true
```

From this we can also create our own association from within our model. This gives
easy access to available file attachments.

```ruby
class MyModel
  has_one_attached :file
  has_one :file_attachment, as: :record, class_name: "ActiveStorage::Attachment"

  ...
end
```

I opted not to use the above and instead work through the `has_one_attached` interface but the above working was cool nonetheless.

## Uploading files within the Administrate gem

Administrate is in my opinion a much friendlier alternative to gems like ActiveAdmin. It is simple and mostly adheres to ruby standards without a DSL.

That being said it tries to not do everything so there are times when you'll
need to reach for a second gem. In this case, when you specify that you have a
field that needs to be a file upload things get complex fast. I spent several hours digging through the gem's source code and forums until [I discovered a gem dedicated
to solving this issue](https://github.com/Dreamersoul/administrate-field-active_storage). Called administrate-field-active_storage, all that is necessary is to utilize the new field type called `Field::ActiveStorage` and you can begin
uploading files.

```ruby
# Gemfile
gem "administrate-field-active_storage"

# Administrate Dashboard for model
class MyModelDashboard < Administrate::BaseDashboard

  # Hash to set the types for each column
  ATTRIBUTE_TYPES = {
    id: Field::String,
    file: Field::ActiveStorage
  }

  # Displayed on model form
  FORM_ATTRIBUTES = [
    :file
  ].freeze
end
```

That's it for now. Let me know what you think or if you've got any tasty tricks.

Thanks for reading.
