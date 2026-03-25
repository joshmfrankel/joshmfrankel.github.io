---
layout: post
title: Ruby Enumerable Gonna Show You How It's Done, Done, Done
categories:
  - article
tags:
  - ruby
---

I've been using Ruby for over 11 years now and along the way I've discovered many favorite methods that I keep coming back to. Many of my favorites come from the Enumerable module. I love the simplicity of finding a standard method that does the exact data manipulation. Being able to share these methods with other engineers is fun in of itself. So without further ado, here's some excellent Ruby Enumerable methods to show you how it's done, done, done.

<!--excerpt-->

![How it's done, done, done](/img/2026/how-its-done-done-done.png "Property of KPop Demon hunters")

## Table of Contents

- [partition](#partition)
- [group_by](#group_by)
- [detect / find](#detect--find)
- [tally](#tally)
- [max_by](#max_by)
- [zip](#zip)
- [all-any-one-none](#all-any-one-none)
- [filter_map](#filter_map)
- [to_h](#to_h)

## partition

[Documentation](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-partition)

**What is it**
Splits a collection in two parts

**When to use it**
You need to perform different operations on a collection's subsets.

```ruby
User = Struct.new(:name, :signed_up_at)
collection = [
  User.new(name: "Rumi", signed_up_at: Time.now),
  User.new(name: "Mira"),
  User.new(name: "Zoey", signed_up_at: 1.day.ago),
  User.new(name: "Jinu", signed_up_at: Time.now),
  User.new(name: "Mystery Saja")
]
signed_up_users, not_signed_up_users = collection.partition do |user|
  user.signed_up_at.present?
end

signed_up_users
#=> [#<struct User name="Rumi", signed_up_at=2025-10-09 11:55:38.009997 -0400>, #<struct User name="Zoey", signed_up_at=2025-10-08 15:55:38.010126000 UTC +00:00>, #<struct User name="Jinu", signed_up_at=2025-10-09 11:55:38.010603 -0400>]

not_signed_up_users
#=> [#<struct User name="Mira", signed_up_at=nil>, #<struct User name="Mystery Saja", signed_up_at=nil>]
```

## group_by

[Documentation](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-group_by)

**What is it**
Reorganizes a collection based on result.

**When to use it**
You need to separate many different records into distinct groups

```ruby
collection = [
  { name: "Rumi", demon: :maybe },
  { name: "Mira", demon: :no },
  { name: "Zoey", demon: :no },
  { name: "Jinu", demon: :yes },
  { name: "Gwi-ma", demon: :yes },
  { name: "Mystery Saja", demon: :yes },
  { name: "Bobby", demon: :no },
]

collection.group_by { |item| item[:demon] }
#=> {
#     maybe: [{ name: "Rumi", demon: :maybe }],
#     no: [{ name: "Mira", demon: :no }, { name: "Zoey", demon: :no }, { name: "Bobby", demon: :no }],
#     yes: [{ name: "Jinu", demon: :yes }, { name: "Gwi-ma", demon: :yes }, { name: "Mystery Saja", demon: :yes }],
#    }
```

> Not a core Ruby Enumerable and there is also the Rails **Enumerable#index_by**. Unlike **group_by** it will only associate a single value to each key, so if you need an array associated to each key use group_by. Otherwise a 1-to-1 key value association can work well for **index_by**.
>
> [Enumerable#index_by](https://api.rubyonrails.org/classes/Enumerable.html#method-i-index_by)

## detect / find

[Documentation](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-detect)

**What is it**
Finds the first match in a collection

**When to use it**
You need to return a record from a collection as soon as it is found

```ruby
collection = [
  { name: "Rumi", demon: :maybe },
  { name: "Mira", demon: :no },
  { name: "Zoey", demon: :no },
  { name: "Jinu", demon: :yes },
  { name: "Gwi-ma", demon: :yes },
  { name: "Mystery Saja", demon: :yes },
  { name: "Bobby", demon: :no },
]

collection.detect { |item| item[:demon] == :maybe }
# => {name: "Rumi", demon: :maybe}
```

## tally

[Documentation](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-tally)

**What is it**
Counts the total occurrences of each value. Akin to a frequency distribution of the available results.

**When to use it**
You need to count the unique occurrences for each value in a collection.

```ruby
character_ages = [40, 12, 32, 63, 32, 32, 11, 20, 12]
#=> [40, 12, 32, 63, 32, 32, 11, 20, 12]
character_ages.tally
#=> {40 => 1, 12 => 2, 32 => 3, 63 => 1, 11 => 1, 20 => 1}
```

## max_by

[Documentation](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-max_by)

**What is it**
Returns the object which is the greatest value in the collection

**When to use it**
You have a numerical value associated to a record where you want the top value

```ruby
collection = [
  { name: "Rumi", demon: :maybe, height: 62 },
  { name: "Mira", demon: :no, height: 70 },
  { name: "Zoey", demon: :no, height: 63 },
  { name: "Jinu", demon: :yes, height: 73 },
  { name: "Gwi-ma", demon: :yes, height: 240 },
  { name: "Mystery Saja", demon: :yes, height: 72 },
  { name: "Bobby", demon: :no, height: 65 },
]

collection.max_by { |item| item[:height] }
# => {name: "Gwi-ma", demon: :yes, height: 240}

# With parameter `2` for 2 results
collection.max_by(2) { |item| item[:height] }
#=> [{name: "Gwi-ma", demon: :yes, height: 240}, {name: "Jinu", demon: :yes, height: 73}]
```

## zip

[Documentation](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-zip)

**What is it**
Merges two or more collections into a single collection which is zippered together

**When to use it**
You have multiple datasets which need to be combined in a staggered style. Very useful for database imports.

```ruby
names = ["Rumi", "Mira", "Zoey"]
demon_status = [:maybe, :no, :no]
heights = [62, 70, 63]

names.zip(demon_status, heights)
#=> [["Rumi", :maybe, 62], ["Mira", :no, 70], ["Zoey", :no, 63]]
```

## all?, any?, one?, none?

[all?](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-all-3F)  
[any?](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-any-3F)  
[none?](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-none-3F)  
[one?](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-one-3F)  

**What is it**
Detects when the block condition returns true for the element and returns boolean

**When to use it**

- **all?** Should be used when you MUST check every single element in the collection (will iterate all elements)
- **any?** Should be used when you want to return as early as the FIRST element that matches (will early return)
- **one?** Should be used when you MUST ensure that 1 and only 1 element matches the block (will iterate all elements)
- **none?** Should be used when you MUST check every single element in the collection to ensure it never matches (will iterate all elements)

```ruby
collection = [
  { name: "Rumi", demon: :maybe, height: 62 },
  { name: "Mira", demon: :no, height: 70 },
  { name: "Zoey", demon: :no, height: 63 },
  { name: "Jinu", demon: :yes, height: 73 },
  { name: "Gwi-ma", demon: :yes, height: 240 },
  { name: "Mystery Saja", demon: :yes, height: 72 },
  { name: "Bobby", demon: :no, height: 65 },
]

collection.all? { |item| item[:demon] == :maybe }
# => false
# Conclusion:
# False! All results must have a key :demon that is equivalent to :maybe
# Returns early on FAILURE, otherwise must check all items

collection.any? { |item| item[:height] == 240 }
# => true
# Conclusion:
# True! There must be at least 1 result with a height of 240
# Returns early on TRUTHY; otherwise must check all items

collection.one? { |item| item[:demon] == :maybe }
# => true
# Conclusion:
# True! There must ONLY be a single result with a demon status of :maybe
# Returns early on FAILURE; otherwise must check all items

collection.none? { |item| item[:height] == 100 }
# => true
# Conclusion:
# True! There must not be a single height that is equal to 100
# Returns early on FAILURE; otherwise must check all items
```

## filter_map

[Documentation](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-filter_map)

**What is it**
Maps and returns truthy elements

**When to use it**
You need a subset of a collection without any nil items.

This is syntactical sugar for `collection.map(&:method).compact`

```ruby
collection = [
  { name: "Rumi", demon: :maybe, height: 62 },
  { name: "Mira", demon: :no, height: 70 },
  { name: "Zoey", demon: :no, height: 63 },
  { name: "Jinu", demon: :yes, height: 73 },
  { name: "Gwi-ma", demon: :yes, height: 240 },
  { name: "Mystery Saja", demon: :yes, height: 72 },
  { name: "Bobby", demon: :no, height: 65 },
]

collection.filter_map { |item| item if item[:demon] == :yes }
#=> [{name: "Jinu", demon: :yes, height: 73}, {name: "Gwi-ma", demon: :yes, height: 240}, {name: "Mystery Saja", demon: :yes, height: 72}]

# Example of just using .map
collection.map { |item| item if item[:demon] == :yes }
#=> [nil, nil, nil, {name: "Jinu", demon: :yes, height: 73}, {name: "Gwi-ma", demon: :yes, height: 240}, {name: "Mystery Saja", demon: :yes, height: 72}, nil]
```

## to_h

[Documentation](https://docs.ruby-lang.org/en/4.0/Enumerable.html#method-i-to_h)

**What is it**
Interprets the collection as a Hash

**When to use it**
You need to associate value pairs into key-value configuration

```ruby
collection = [
  ["Rumi", { demon: :maybe, height: 62 }],
  ["Mira", { demon: :no, height: 70 }],
  ["Zoey", { demon: :no, height: 63 }],
  ["Jinu", { demon: :yes, height: 73 }],
  ["Gwi-ma", { demon: :yes, height: 240 }],
  ["Mystery Saja", { demon: :yes, height: 72 }],
  ["Bobby", { demon: :no, height: 65 }],
]

collection.to_h
# =>
# {
#   "Rumi" => { demon: :maybe, height: 62 },
#   "Mira" => { demon: :no, height: 70 },
#   "Zoey" => { demon: :no, height: 63 },
#   "Jinu" => { demon: :yes, height: 73 },
#   "Gwi-ma" => { demon: :yes, height: 240 },
#   "Mystery Saja" => { demon: :yes, height: 72 },
#   "Bobby" => { demon: :no, height: 65 },
# }
```

_Find, zip, to_h, partitionnnn,_  
_Fit check for my Enumerable era_

![Celebration & Ramen](/img/2026/celebrate-ramen.png "Property of KPop Demon hunters")
