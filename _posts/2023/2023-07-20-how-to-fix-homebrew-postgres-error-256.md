---
layout: post
title: How to fix homebrew postgres error 256
categories:
- today-i-learned
tags:
  - mac
  - postgres
---

At home, I'm a Linux user but at work most companies use Mac for engineers. It's been
a couple years since I've used Mac so I'm learning my way back around it along with
Homebrew. One issue I ran into the other day was postgresql service not starting
sucessfully. No amount of `brew services restart postgresql` would fix it. Turns
out to be a pretty simple fix.

<!--excerpt-->

I first noticed the issue when trying to run a Ruby on Rails server locally. The response
from running `rails s` looked something like:

``` bash
psql: could not connect to server: No such file or directory
    Is the server running locally and accepting
    connections on Unix domain socket "/tmp/.s.PGSQL.5432"?
```

Alright, so this is saying the server isn't running or listening to requests. Checking
`brew services list` to see the output rendered:

![Brew services list output](/img/2023/brew-service-list.png)

Next was looking up what a 256 error which led me to an excellent [StackOverflow post](https://stackoverflow.com/questions/39710384/cannot-connect-to-postgres-server-running-through-brew-services).
Following the post suggested three step process of: stopping the service, removing the pid file, and restarting the service.

![Brew services stop output](/img/2023/brew-services-stop.png)

The pid file can be found in one of two locations:

* `/usr/local/var/postgres/postmaster.pid`
* `/opt/homebrew/var/postgresql/postmaster.pid` (Apple M1)

I found mine within the `/opt/` directory. Last was just restarting the service.

![Brew services start output](/img/2023/brew-services-start.png)

After this running `rails s` worked as expected.

Thanks for reading it was a quick one.
