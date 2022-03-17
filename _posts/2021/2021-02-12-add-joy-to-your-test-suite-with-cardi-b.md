---
layout: post
title: Add joy to your test suite with Cardi B
date: 2021-02-12 10:58 -0500
categories:
- articles
tags:
- ruby
- rspec
- humor
---

We've been in the pandemic now for nearly a full year. That's enough time to make everyone feel a bit down and exhausted. Humor is a great way of coping with pandemic fatigue or you know just generally putting you in a good mood. What better way to encourage yourself than having Cardi B join you for testing your code.
<!--excerpt-->

This started as a joke in a Slack thread and quickly became a fun little hack I put together. Specifically what I wanted to happen was:

* Play a positive sound effect when the test suite passes
* Play a bummer sound effect when the test suite fails

![Cardi B eowwwww](/img/2021/eowwww.gif)

## Finding a sound effect
First off, you'll have to download some sound effects. You can find these online by searching YouTube or other soundboard sites and then downloading the files. Once found, save them somewhere you'll remember.

## Playing sound effects in Ruby

How do you play a sound effect in Ruby? Well, Ruby doesn't have any knowledge of audio but by using `system(...)` it can shell out to your operating system's primary audio player. For OSX this meant using `afplay [audio-file]`. There's a downside here though. Using `system` makes the ruby program wait until the system call is finished. So if you had say a 30 second sound effect, you'd have to wait the spec runtime plus 30 seconds before the script would finish running.

We can do better here. Much better.

Using `fork` we can tell Ruby to fork a process in the background to run our shell command. This combined with
`exec` allows us to run our sound effect in parallel of the script meaning that the sound effect will play as the script runs. No more waiting! This is what it looks like `fork { exec("...") }`

## Adding sound effects to your test suite

Now for the RSpec hack. Within your `RSpec.configure do |config|` block, we'll need to add an `after(:suite)` hook. Next we dig into the RSpec::Core::Reporter to count the number of failed examples. Finally we can put it all together in our RSpec configuration.

``` ruby
  config.after(:suite) do
    if config.reporter.failed_examples.size == 0
      fork { exec("afplay ~/Music/okurrrt.mp3") }
    else
      fork { exec("afplay ~/Music/eowwww.mp3") }
    end
  end
```

If you want to get even fancier you can add some variety and randomness to the sound effect played. We can add randomization by using Ruby's **array#sample**. For variety, I'll add clips of #SELFIE from
The Chainsmokers and DJ Khaled _another one_ to the list. Why not?

``` ruby
  config.after(:suite) do
    if config.reporter.failed_examples.size == 0
      success = [
        "~/Music/okurrrt.mp3",
        "~/Music/selfie.mp3",
        "~/Music/another_one.mp3"
      ]
      fork { exec("afplay #{success.sample}") }
    else
      fork { exec("afplay ~/Music/eowwww.mp3") }
    end
  end
```

![Cardi B okurrrt](/img/2021/okurrrt.gif)

With the above, I can _okurrrt_ and _#selfie_ my way to a green testing suite. Want to hear some more of Cardi B? [Check out this YouTube video.](https://youtu.be/YplKPH_qcRw?t=154){:target="_blank"}
