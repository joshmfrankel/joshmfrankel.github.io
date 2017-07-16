---
layout: post
title: Using Ruby's Array#zip method to insert values between every index
categories: Today I learned
---

arr = ["1040933", "1083538"]
arr.zip([DateTime.now] * arr.size)
#=> [["1040933", Thu, 13 Jul 2017 13:01:51 -0400], ["1083538", Thu, 13 Jul 2017 13:01:51 -0400]]
Hash[*arr.zip([DateTime.now] * arr.size).flatten]
#=> {"1040933"=>Thu, 13 Jul 2017 11:53:51 -0400, "1083538"=>Thu, 13 Jul 2017 11:53:51 -0400}
