---
layout: post
title: Use ActiveModel::Api for a Bare Bones Action Model interface
categories:
  - today-i-learned
tags:
  - ruby on rails
---

Today, I learned that **ActiveModel::Api** is the minimal implementation for an object to act like a model. **ActiveModel::Model** was the standard prior to Rails 7. You can still use it but it implies additional model-esque functionality where as API is the bare bones interface.

<!--excerpt-->

> Includes the required interface for an object to interact with Action Pack and Action View, using different Active Model modules. It includes model name introspections, conversions, translations, and validations. Besides that, it allows you to initialize the object with a hash of attributes, pretty much like Active Record does.
>
> - [api.rubyonrails.org](https://api.rubyonrails.org/classes/ActiveModel/API.html)

I dug in a bit further and looked at the [current implementation](https://github.com/rails/rails/blob/9f466dd9d4672d7dc6c49a7861d9e30eff69c163/activemodel/lib/active_model/model.rb#L45) for **ActiveModel::Model** and it looks like the only current difference is the addition of the **ActiveModel::Access** module. For more information see the [full documentation](https://api.rubyonrails.org/classes/ActiveModel/API.html) or the [implementing pull request](https://github.com/rails/rails/pull/43223).
